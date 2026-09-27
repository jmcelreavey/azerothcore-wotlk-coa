# fix(CoA/Necromancer): a commanded minion kill hands the mob to the caster

## Summary

A Necromancer who points their Ghouls at a mob gets no experience and no loot
for it, even though they just used an ability on that mob. The mob is tagged
(the loot recipient is set) and then the tag is thrown away at the moment of
death, because the core only records a creature as "damaged by a player" for a
player, a unit a player moves, a unit a player charms, or a _player-controlled_
unit owned by a player. A Necromancer summon is a `Guardian` created by
`Map::SummonCreature`, not a minion added through `Unit::SetMinion`, so it is
never player-controlled and no clause applies. `Order()` now records the caster's
engagement on the target before it sends the order.

## Root cause and fix, per issue

**#4011 - Necromancer pets and loot flag.** The reporter narrowed this one
himself in a comment: the problem is not a summon that runs off and attacks a mob
on its own, it is that **Command: Undead does not flag the mob it targets**. The
maintainer agreed in the same thread ("If the player uses any ability including
commands then it should tag"). Core `d212cbdb0bb8` (2026-09-17), one day before
#4221.

Mechanically: `Unit::DealDamage` (`src/server/game/Entities/Unit/Unit.cpp:1236-1257`)
enters the tagging block for any attacker that is null, controlled-by-player or
created-by-player - a Necromancer summon is the last of those - and then computes

```cpp
bool damagedByPlayer = unDamage && attacker && (attacker->IsPlayer() || attacker->m_movedByPlayer
    || attacker->GetCharmerGUID().IsPlayer()
    || (attacker->IsControlledByPlayer() && attacker->GetOwnerGUID().IsPlayer()));
```

`TempSummon::InitStats` sets `m_ControlledByPlayer` only for a trigger with spells
(`src/server/game/Entities/Creature/TemporarySummon.cpp:217-223`) and
`Unit::SetMinion` (which does set it, plus `UNIT_FLAG_PLAYER_CONTROLLED`) is
reached only through `AddMinion`, not through `Map::SummonCreature`
(`src/server/game/Entities/Object/Object.cpp:2277-2303`). The Necromancer's
`IsSummonedBy` sets owner and creator GUIDs but never the flag
(`src/server/coa/AscensionNecromancerSummons.cpp:363-364`), unlike the Tinker's
(`AscensionTinkerSummons.cpp:255`) and the Xoroth's
(`AscensionXorothSummons.cpp:109`), which set it by hand for the same reason. So
`damagedByPlayer` is false, `Creature::IsDamageEnoughForLootingAndReward()`
(`src/server/game/Entities/Creature/Creature.cpp:3946-3949`) is false, and
`Unit::Kill` clears the loot recipient (`Unit.cpp:14914-14917`): no loot, no
experience, no kill credit.

Fix: `Order()` (`AscensionNecromancerSummons.cpp:312`) calls
`Creature::LowerPlayerDamageReq(creature->GetHealth(), true, player->GetLevel())`
on the target when the creature is not already rewardable. That is exactly what
using an ability on a mob means, and it is the narrowest statement of the rule
the reporter and the maintainer agreed on.

**#4875 - mob doesn't loot when killed by raised pets. NOT fixed (Refs).** The
reporter's own repro is a pet that kills a mob with no player input, which is the
half of #4011 the maintainer explicitly ruled _not_ a bug ("My previous comment
was regarding pets that have attacked mobs with no player input which should not
tag mobs and is working as intended"). Making that case work needs the minion to
satisfy the `IsControlledByPlayer()` clause, and `m_ControlledByPlayer` gates 44
call sites in `src/server/game` - spell scaling (`Spell.cpp:3504`), AoE targeting
(`SpellInfo.cpp:1983,2009,3093`), movement, threat and victim selection. It could
not be evaluated or tested here. `Order()` does cover the reporter's stated
workflow (round up a pull, use a command, let the Ghouls work), but not the
literal "kill mobs with your pet but don't touch it". Left for a follow-up that
can measure the blast radius.

**#4119 - Necromancer inconsistent XP attribution. NOT fixed (Refs).** Two parts.
The report's own scenario is Undead Assault with no player input, which is the
same not-a-bug case; a comment on #4119 says so ("This is to prevent AFK farming
with a pet"). Run 3's theory that the summon despawns before its damage lands was
checked and does not hold: `AscensionNecromancerSummons.cpp:637` despawns after
100 ms, which is there to let the explosion resolve. The remaining part - XP for
a minion-only kill - is the same `m_ControlledByPlayer` question as #4875.

## Tests to run on the PC

```sh
python -B tools/verify_all.py --stages build,gameplay --scenario necromancer-commanded-minion-kill-xp-loot
python -B tools/verify_all.py --stages build,gameplay --scenario necromancer-1456-command-undead-four-ghouls
python -B tools/verify_all.py --stages build,gameplay --scenario knight-of-xoroth-greater-imp-solo-kill-xp
python -B tools/verify_all.py --base origin/main
python apps/codestyle/codestyle-cpp.py --files src/server/coa/AscensionNecromancerSummons.cpp
```

`necromancer-commanded-minion-kill-xp-loot` raises a Ghoul, casts Command: Undead
504868 on a Kobold, and requires that the caster's experience rises and that
`loot_count >= 1` after `loot_creature`, while `melee_attack_count` stays 0 so
only the Ghoul dealt damage. **It fails on `main`** (no experience, no loot) and
passes with this change, so it is the fail-before/pass-after check. The two
existing scenarios are regression guards: the four-Ghouls one proves the order
still reaches every Ghoul, and the Xoroth one proves the #4221 pet path is
untouched.

In game: level 20 Necromancer in Mulgore, `.learn 500971`, `.learn 504868`,
`.cast 500971` (raise the Ghoul), `.cast 504868` on a nearby hostile mob, and
let the Ghoul finish it. Pass: the combat log shows a kill credit and an
experience gain for the Necromancer, the corpse is lootable by that character,
and another character at the same level cannot loot it. Then repeat without the
command: the minion-only kill should still give nothing, which is the documented
behaviour and the regression guard for over-tagging.

## Risks

- This makes a command a tag. A player can tag a mob from 25 yd with a command
  and let a minion kill it, which is the Ascension behaviour the reporter
  describes and the maintainer asked for, but it is a small loosening of
  "someone must hurt it first".
- `LowerPlayerDamageReq(creature->GetHealth(), ...)` zeroes the player damage
  requirement outright. The requirement is `GetHealth() / 2` at spawn, so
  passing the current health always satisfies it; a mob already damaged more than
  half by somebody else is still tagged by the command, which is intended.
- Creatures that already have `CREATURE_FLAG_EXTRA_NO_PLAYER_DAMAGE_REQ` are
  skipped, so nothing changes for them.
- No gameplay scenario covers a minion-only kill with no player input, so the
  "still not tagged" half of the contract is only checked by the in-game repro.

## Not addressed

- #4875 and #4119's "no player input" half: needs `m_ControlledByPlayer` on
  Necromancer minions, deferred as described above.
- Corpse Explosion (`src/server/coa/AscensionNecromancerAbilities.cpp:18-31`)
  calls `corpse->RemoveCorpse()` without checking whether the corpse still has
  loot, which is a plausible source of the "experience but no loot" wording in
  #4875. Not changed: the report does not identify Corpse Explosion, and skipping
  unlooted corpses changes which corpses the ability consumes.

Refs #4875
Refs #4119
Fixes #4011
