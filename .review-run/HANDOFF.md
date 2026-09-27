# Groups 3-6 (continuation of run 3): handoff

**Clone:** `~/src/azerothcore-wotlk-coa-groups` (base `main` = `4c3a06dd0`, push disabled, nothing pushed)
**Scope:** G3 Barbarian weapon ranks (#5124, #5135) · G4 summon kill credit (#4875, #4011, #4119) ·
G5 Runemaster sweep (14 player reports + 145 audit reports) · G6 per-viewer mob scaling (#1471, #4296, #4382, #1444, #5134, #211)
**Not in scope (done by run 3 on the PC):** G1 talent validation (`fix/talent-validation`), G2 buff exclusivity (`fix/buff-exclusivity`).

**Companion files in this folder:** `runemaster-audit.md` (the 145-report sweep),
`mob-scaling-design.md` (the G6 write-up), and one PR draft per branch
(`fix-barbarian-weapon-ranks.md`, `fix-summon-kill-credit.md`,
`fix-runemaster-player-reports.md`, `fix-mob-scaling.md`).

**What was NOT done here.** This machine has no build toolchain, database, worldserver or
game client. Nothing was compiled, no SQL was applied, no scenario was executed and no issue
was reproduced in game. The only verification run was
`python3 -B tools/verify_all.py --stages source --base main` on each branch (**PASSED** on all
four), `codestyle-cpp.py --files` on every changed `.cpp` (**PASSED**),
`run.py validate` on the three new scenario files (**valid**), and `git diff --check` (clean).
No new `.cpp` file was added, so **no CMake reconfigure is required**; if the PC adds one for
the deferred work, run `~/src/coa-dev/setup-dev.sh configure`.

---

## Result at a glance

### Verdict counts

| Group                         | Fixed  | Already worked / working as designed | Not obtainable or not a bug | Confirmed defect, deferred | Total issues                  |
| ----------------------------- | ------ | ------------------------------------ | --------------------------- | -------------------------- | ----------------------------- |
| G3 Barbarian weapon ranks     | 2      | 0                                    | 0                           | 0                          | 2                             |
| G4 summon kill credit         | 1      | 0                                    | 1                           | 2                          | 3 (+#4011 in the same thread) |
| G5a Runemaster player reports | 6      | 4                                    | 2                           | 2                          | 14                            |
| G5b Runemaster audit          | 1      | 39                                   | 75                          | 30                         | 145                           |
| G6 mob scaling                | 1      | 1                                    | 1                           | 3                          | 6                             |
| **Total**                     | **11** | **44**                               | **79**                      | **37**                     | **170**                       |

G4: #4011 is one issue reported against three sub-symptoms; the two "not a bug" halves are the
no-player-input case, which the maintainer ruled out in #4011's own thread.
G5a "deferred" is #4588 (a design question) and #4979 + #4188 (need `SummonProperties.dbc`);
#4588 is counted once, so G5a is 6 fixed + 4 as designed + 3 deferred + 1 = 14.

### Branches

| Branch                          | Commits | Files | Issues closed                                                |
| ------------------------------- | ------- | ----- | ------------------------------------------------------------ |
| `fix/barbarian-weapon-ranks`    | 1       | 2     | #5124, #5135                                                 |
| `fix/summon-kill-credit`        | 1       | 2     | #4011 (refs #4875, #4119)                                    |
| `fix/runemaster-player-reports` | 4       | 4     | #4760, #5175, #5410, #5414, #5254, #3651 (refs #4138, #4820) |
| `fix/mob-scaling`               | 1       | 1     | #4296                                                        |
| **Total**                       | **7**   | **9** | **11 `Fixes`, 4 `Refs`**                                     |

All four branches are independent, each created from `main` at `4c3a06dd0`. No branch was pushed.
There is **no `fix/runemaster-audit` branch** - the audit's only real bug (#3651) is fixed on
`fix/runemaster-player-reports`, and its 30 confirmed gaps are one class-wide unit of work that
cannot be landed unverified. The reasoning is in `runemaster-audit.md`.

---

## 1. Summary table

### G3 - Barbarian weapon ranks (`fix/barbarian-weapon-ranks`)

| #    | Spell                      | Verdict   | Evidence (one line)                                                                                                |
| ---- | -------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------ |
| 5124 | Brutal Swing 500996 r2-r8  | **fixed** | `EquippedItemInventoryTypeMask` = 1<<17 on r2+ and 0 on r1 (500913); ADB renders the extra requirement on r2+ only |
| 5135 | Decapitate 806905 r2/r4/r5 | **fixed** | Same mask; rank 3 (806906) has none, so it is sporadic data, not a per-rank change                                 |

### G4 - Pet/summon kill credit (`fix/summon-kill-credit`)

| #    | Title                                       | Verdict             | Evidence                                                                                                                                                              |
| ---- | ------------------------------------------- | ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 4875 | Mob doesn't loot when killed by raised pets | **deferred (Refs)** | Repro is a minion-only kill with no player input, which the maintainer ruled out in #4011's thread; fixing it needs `m_ControlledByPlayer`, which gates 44 call sites |
| 4011 | Necromancer pets and loot flag              | **fixed**           | `Order()` never told the core the caster engaged the mob, so `_damagedByPlayer` stayed false and `Unit::Kill` cleared the loot recipient                              |
| 4119 | Necromancer inconsistent XP attribution     | **deferred (Refs)** | Same root cause as #4875; the "despawn before damage lands" theory from run 3 was checked and does not hold                                                           |

### G5a - Runemaster player reports (`fix/runemaster-player-reports`)

| #    | Spell                                                       | Verdict                             | Evidence                                                                                                                                                                        |
| ---- | ----------------------------------------------------------- | ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 4760 | Palm Sigils unusable (Runeshroud 500288)                    | **fixed**                           | Marker 808089 is a passive aura; `Aura::CanBeSentToClient` drops passive auras, so the client never learns the player has it                                                    |
| 5175 | Palm Sigil: Fire 534803 greyed out                          | **fixed**                           | Same; also reports the Warpdagger action-bar copies (see #4138)                                                                                                                 |
| 5410 | Palm Sigil bug                                              | **fixed**                           | Same                                                                                                                                                                            |
| 5414 | Riftblade - Palm Sigil: Fire                                | **fixed**                           | Same; filed against `4c3a06dd0399`, this branch's base                                                                                                                          |
| 4138 | Warpdagger action-bar copies                                | **partly fixed (Refs)**             | `ClearTravel` unlearned Warp 500587 so `StartTravel`'s "only learn it if unknown" guard never held; the two `SMSG_SUPERCEDED_SPELL` packets per travel remain                   |
| 4820 | Warpdagger creating copies                                  | **partly fixed (Refs)**             | Same                                                                                                                                                                            |
| 5254 | Eternal Magic 806698                                        | **fixed**                           | Tooltip says 3 Runeblade charges; the code restored exactly 1 unconditionally                                                                                                   |
| 3651 | Eternal Magic (audit report)                                | **fixed**                           | Same spell as #5254, closed with it                                                                                                                                             |
| 4588 | Frost Glyph not refreshing                                  | **not a bug (working as designed)** | The Glyph passives are an exclusive rotation (ADB tooltip for 805727: "... while Flame Glyph is active now grants you Arcane Glyph"); refreshing the lower glyph would break it |
| 4490 | Level 10 Glyphic starts with Glyphic Ruin                   | **not a bug (working as designed)** | 801179 is a class-32 tree node (`SpellbookTreeSpellData.h:3919`) and #4436 requires a spec tree's free identity nodes to be auto-granted                                        |
| 5269 | Knight of Xoroth and Glyphic get their 1st spec talent free | **not a bug (working as designed)** | Same, cross-class                                                                                                                                                               |
| 4421 | Can't choose Glyphic Ruin (level 11)                        | **not a bug (working as designed)** | Same behaviour seen from the other side: the node is already granted                                                                                                            |
| 4979 | Homebound Runestone 560316 creates no object                | **deferred**                        | One `SPELL_EFFECT_SUMMON` (MiscValue 1); no "Homebound Runestone" creature exists in the ADB, and wiring it needs `SummonProperties.dbc`                                        |
| 4188 | Scrying Orb 804674 does not work                            | **deferred**                        | One `SPELL_EFFECT_SUMMON` (MiscValue 5); the ADB's only candidate is creature 50038 "Scrying Orb", which has no `creature_template` row and no script                           |

### G5b - Runemaster audit (145 reports)

Full table with per-issue evidence: **`runemaster-audit.md`**.

| Verdict                                                           | Count   |
| ----------------------------------------------------------------- | ------- |
| fixed (on `fix/runemaster-player-reports`)                        | 1       |
| confirmed gap, deferred                                           | 29      |
| data/tooltip mismatch, deferred                                   | 1       |
| not obtainable / dead data                                        | 75      |
| false positive - works natively from the DBC                      | 33      |
| already worked - the fork's SQL, source or a scenario services it | 6       |
| **Total**                                                         | **145** |

### G6 - Mob scaling (`fix/mob-scaling` + `mob-scaling-design.md`)

| #    | Title                                            | Verdict               | Evidence                                                                                                                                         |
| ---- | ------------------------------------------------ | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| 4296 | Mobs scaling off high-level bots                 | **fixed**             | `DesiredLevel` had no bot filter, and the nearest-qualifying-session rule let a nearby bot take the level                                        |
| 1471 | One scaled level for the whole party             | **deferred (design)** | The realm-wide path mutates the shared creature; the per-viewer rework #1471 already tracks is the fix. Confirmed by the maintainer in the issue |
| 4382 | Mobs actually gaining levels (health and damage) | **deferred (design)** | `ScaleForEngager` calls `SelectLevel()` on the shared object, so health and damage move for everyone                                             |
| 1444 | Ragefire Chasm mobs do not scale                 | **already working**   | Maintainer could not reproduce on current code; the report's expectation is the per-viewer feature                                               |
| 5134 | Some enemies not scaling up to me                | **partly fixed**      | The bot part is this commit; the rest is the "only re-evaluate out of combat at full health" rule (`AscensionCompat.cpp:6142-6146`)              |
| 211  | Shadowfang Keep drops ilvl 15 loot               | **deferred (design)** | Loot is filled from the template on both paths and `docs/coa/level-scaling.md:203-204` says that is deliberate; one corpse, many looters         |

---

## 2. Per-issue details

### G3 #5124 and #5135 - fixed

**Expected behaviour.** Rank 1 and rank 2 of one chain are the same ability. The Ascension DB
tooltip for 500913 and 804414 lists the weapon requirement as "Axe or Two-Handed Axe or Mace or
Two-Handed Mace or Polearm or Sword or Two-Handed Sword or Staff or Fist Weapon or Dagger" with
no slot restriction; 500996, 501002 and 806905 list the same weapon types **plus "in Bag"**,
which is the ADB rendering of a non-zero `EquippedItemInventoryTypeMask`
(<https://ascension-db.ascension-archive.workers.dev/#search?kind=spell>).

**Root cause.** The same records keep the one-handed subclasses in
`EquippedItemSubClassMask` (173555 = 0x2A5F3) while
`EquippedItemInventoryTypeMask` = `1 << INVTYPE_2HWEAPON` (17) admits only a two-handed
weapon. `Item::IsFitToSpellRequirements`
(`src/server/game/Entities/Item/Item.cpp:912-921`) rejects a one-handed weapon at line 919-920,
`Player::HasItemFitToSpellRequirements` (`Player.cpp:13124-13158`) then finds no equipped
weapon that fits, and `Spell::CheckItems` (`Spell.cpp:7421-7422`) returns
`SPELL_FAILED_EQUIPPED_ITEM_CLASS` (29, `SharedDefines.h:1053`).

**Proven not to be a design choice, three ways.**

1. Decapitate rank 3 (806906) has the full one-handed + two-handed subclass list and **no** slot
   mask, sitting between rank 2 (masked) and rank 4 (masked).
2. The mask contradicts the subclass mask on the same row: six of the ten listed weapon types
   could never satisfy the requirement.
3. Two independent reports name the exact regression point (rank 2 / level 26) and the same
   weapon (one-handed maces), and rank 1 worked for the same character.

**Corrections to run 3's findings.** Run 3's DBC table is confirmed (ranks 2+ mask 131072,
rank 1 mask 0; SubClassMask 173555; Attributes[0] 0x50014; Attributes[6] 0x2000000 on ranks 2+,
which the ADB history shows changed on 2026-07-13/08-13 together with Attributes[1] and has no
bearing on the weapon check). Run 3 suggested putting the fix in `AscensionClassMechanics.cpp`
next to `RepairedRangedInventoryMask`; it lives in `AscensionBarbarian.cpp` instead, because that
is the existing Barbarian spell-info repair hook (`ApplyAscensionBarbarianSpellChanges`, already
shape-guarded for `BODY_BUILDER_SIZE`) and it keeps the branch Barbarian-only.
`RepairedRangedInventoryMask` cannot help: it returns the mask unchanged for any subclass mask
containing a melee subclass (`AscensionClassMechanics.cpp:961`).

**Sweep result.** All 26 Barbarian rank chains in `AscensionProgression::Ranks` were compared
against the ADB. The only other inventory-slot mismatch is Throw Weapon 806876-806881, whose
subclass is `THROWN` (16) and is already repaired by `RepairedRangedInventoryMask`, which adds
`1 << INVTYPE_THROWN` for that subclass. Crush 500915 and Berserker Axe 804138's rank-1 records
are proc/driver spells in the ADB, not mismatches. Decapitate 806904 and 807116 carry the same
artefact but are in neither the progression table nor `SpellbookTreeSpellData.h`.

### G4 #4011 - fixed; #4875 and #4119 - deferred

**Expected behaviour.** In #4011 the reporter narrowed his own report: summons running off and
attacking a mob on their own should not tag it, but "Command: Undead should flag a mob for
exp/loot and it's not". The maintainer agreed in the same thread: "If the player uses any
ability including commands then it should tag."

**Root cause (all three).** `Unit::DealDamage` (`Unit.cpp:1236-1257`) enters the tagging block
for any attacker that is null, controlled-by-player or created-by-player, then computes
`damagedByPlayer` from four clauses, the last being
`(attacker->IsControlledByPlayer() && attacker->GetOwnerGUID().IsPlayer())`.
`TempSummon::InitStats` sets `m_ControlledByPlayer` only for a trigger with spells
(`TemporarySummon.cpp:217-223`), and `Unit::SetMinion` - which does set it plus
`UNIT_FLAG_PLAYER_CONTROLLED` (`Unit.cpp:8360-8364`) - is reached only through `AddMinion`, not
through `Map::SummonCreature` (`Object.cpp:2277-2303`). The Necromancer's `IsSummonedBy` sets
owner and creator GUIDs but never the flag (`AscensionNecromancerSummons.cpp:363-364`), unlike
the Tinker's (`AscensionTinkerSummons.cpp:255`) and the Xoroth's
(`AscensionXorothSummons.cpp:109`). So `damagedByPlayer` is false,
`Creature::IsDamageEnoughForLootingAndReward()` (`Creature.cpp:3946-3949`) is false, and
`Unit::Kill` clears the loot recipient (`Unit.cpp:14914-14917`).

**Fix.** `Order()` calls `LowerPlayerDamageReq(creature->GetHealth(), true, player->GetLevel())`
on the target when it is not already rewardable. #4875's reporter's workflow (round up a pull,
command, let the Ghouls work) is covered; the literal "kill mobs with your pet but don't touch
it" is not, and that half needs `m_ControlledByPlayer`.

**Corrections to run 3's findings.**

- Run 3 said `Map::SummonCreature` sets `m_ControlledByPlayer` "via `Unit::SetMinion`". It does
  not; that is a different code path. The real reason no Necromancer summon qualifies is that
  the core only sets the flag for a trigger summon, and the Necromancer's own AI never sets it.
- Run 3 listed "damage from an already-despawned attacker arrives with a null attacker"
  (Undead Assault despawning ~100 ms after exploding) as a gap. Checked: the despawn is
  `DespawnOrUnsummon(100ms)` at `AscensionNecromancerSummons.cpp:637`, immediately after the
  explosion cast, precisely so the damage resolves first. Not a defect.
- Run 3's other two suggestions (property-64 plain TempSummons; Corpse Explosion deleting
  lootable corpses) were not pursued: the first is a consequence of the same
  `m_ControlledByPlayer` question, and the second is a plausible but unproven source of #4875's
  "experience but no loot" wording (Corpse Explosion calls `RemoveCorpse()` without checking for
  unlooted loot, `AscensionNecromancerAbilities.cpp:27`). Neither is identified by the report.

**Dating.** #4875's core `734dbab082b9` is 2026-09-22, i.e. **after** #4221 (2026-09-19), so
"already fixed by #4221" does not apply to it. #4011 (`d212cbdb0bb8`, 2026-09-17) and #4119
(`7b66e4380261`, 2026-09-18) predate it, but #4221 only credits a _player-controlled_ minion, so
it did not cover the Necromancer case in any of the three.

### G5a - see the summary table; the full argument per issue is in the PR draft

Two verdicts rest on ADB records worth quoting:

- **808089** "Runeshroud or Waveforged" is an `Apply Aura: Dummy` whose Flags table row says
  _Passive_, and the ADB history shows it was **added to the client's `Spell.dbc` on 2026-08-13**.
  `Aura::CanBeSentToClient` (`SpellAuras.cpp:1096-1099`) returns false for a passive aura, so the
  server held the marker and the client never got it. This is the whole of #4760/#5175/#5410/#5414.
- **Glyphic Ruin 801179**: the ADB gives it `Class: Runemaster` with no tree row or column and no
  tree reference, while a real node such as Magic Feeder 705546 has
  `Class: Runemaster - Runemaster (row 7, col 7)` and `tree: runemaster`. So 801179 is a node the
  tree grants rather than a passive with free costs, and #4436's merged behaviour applies.

### G5b - 145 automated reports

Method and full table in `runemaster-audit.md`. The 30 deferred items are grouped there into:
18 `PROC_TRIGGER_SPELL`/`PERIODIC_TRIGGER_SPELL` passives with no `spell_proc` row and no script;
2 `OVERRIDE_CLASS_SCRIPTS` values the core never reads (only 7801 and 7282 are hardcoded);
1 mixed record; 8 effects the ADB cannot name, which need the client DBC to classify at all; and
1 tooltip/DBC disagreement (#2737 Shroudwalker: the tooltip says Warpdagger incurs no cooldown in
Runeshroud, the DBC carries an `ADD_FLAT_MODIFIER -25000`).

**Corrections to run 3's expectations.** Run 3 expected the Cultist ratio (6 real bugs in 137);
this class has 29 genuine dead clauses among 70 obtainable reports. The difference is that the
Cultist tree was largely serviced and the Runemaster tree is not: only 6 of the 70 obtainable
reports are already covered by SQL, source or a scenario, and none of the 18 proc-shaped
talents has a `spell_proc` row. The audit also found a systematic problem worth a
class-wide issue: passives whose DBC effect is a proc aura with `TriggerSpell = 0` produce
nothing, and no script or row services them.

### G6 - see `mob-scaling-design.md` for the per-issue analysis

The load-bearing finding for the whole group: with the shipped `.dist` values
(`DestinyWeaver.Enable = 1`, `DestinyWeaver.LevelScaling = 1`) the **per-viewer** path owns
scaling and the realm-wide path this run could touch is inert
(`CanScaleCreature` returns false when `CreatureScalingOwnedPerViewer` is set,
`AscensionCompat.cpp:6111`). #4296 is the one G6 report that is possible under the default
configuration, and it is fixed. A second finding: `docs/coa/level-scaling.md:33-34` says
`CoA.LevelScalingMaxLift` defaults to 0 while `coa.conf.dist:140` ships 5, and the same knob
silently switches `DesiredLevel` between "nearest" and "highest in sight".

---

## 3. Branches, commits and PR drafts

PR drafts are separate files: `fix-barbarian-weapon-ranks.md`, `fix-summon-kill-credit.md`,
`fix-runemaster-player-reports.md`, `fix-mob-scaling.md`. Each contains the title, summary,
per-issue root cause and fix, the test commands, the in-game repro with a pass condition, risks,
and the "not addressed" list, in the style of run 3's reports.

### `fix/barbarian-weapon-ranks` (1 commit)

```
d53b64229 fix(CoA/Barbarian): accept one-handed weapons on Brutal Swing and Decapitate higher ranks
 src/server/coa/AscensionBarbarian.cpp                                |  39 ++++
 apps/coa-gameplay-test/scenarios/barbarian-weapon-requirement-ranks.json | 246 +++++
```

### `fix/summon-kill-credit` (1 commit)

```
34c06882f fix(CoA/Necromancer): a commanded minion kill hands the mob to the caster
 src/server/coa/AscensionNecromancerSummons.cpp                     |   3 +
 apps/coa-gameplay-test/scenarios/necromancer-commanded-minion-kill-xp-loot.json | 134 +++++
```

### `fix/runemaster-player-reports` (4 commits)

```
4917df417 fix(CoA/Runemaster): Eternal Magic restores three Runeblade charges
60066584d fix(CoA/Runemaster): stop re-teaching Warp on every Warpdagger travel
f36856f79 test(CoA/Runemaster): cover Palm Sigil: Fire and Earth under the Runeshroud gate
749b7cfcf fix(CoA/Runemaster): send the Runeshroud or Waveforged marker to the client
 src/server/coa/AscensionRunemasterTalents.cpp                       |  15 ++
 src/server/coa/AscensionRunemasterTravel.cpp                        |   1 -
 src/server/coa/AscensionRunemasterSecondary.cpp                     |   3 +-
 apps/coa-gameplay-test/scenarios/runemaster-palm-sigil-fire-earth-runeshroud.json | 152 ++++++
```

### `fix/mob-scaling` (1 commit)

```
8904fcbe5 fix(CoA/Scaling): a nearby bot no longer takes a creature's level from a player
 src/server/coa/AscensionCompat.cpp | 37 ++++++++++++++++++++++---------------
```

---

## 4. Test plans (per change)

Full detail is in the PR drafts; the short form:

| Change                 | Compile     | SQL  | Scenarios on the PC                                                                                                                                                                                | In-game repro                                                                                                                                                                                                                                                  |
| ---------------------- | ----------- | ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Barbarian weapon ranks | worldserver | none | `--stages build,gameplay --scenario barbarian-weapon-requirement-ranks` (fails on main for rank 2, passes after)                                                                                   | level 26 Barb, `.additem 25` + equip slot 15, `.learn 500996`, `.cast 500996` on a dummy: cast completes. Repeat for 806905. Unequip and confirm refusal returns. Check the startup log for `Skipped unexpected Barbarian weapon requirement record`           |
| Commanded minion kill  | worldserver | none | `--scenario necromancer-commanded-minion-kill-xp-loot` (fails on main, passes after), plus `necromancer-1456-command-undead-four-ghouls` and `knight-of-xoroth-greater-imp-solo-kill-xp` as guards | level 20 Necro in Mulgore, `.learn 500971`, `.learn 504868`, raise, `.cast 504868` on a hostile mob, let the Ghoul kill it: kill credit + XP for the Necro, corpse lootable by them and not by a same-level second character. Without the command, still no XP |
| Palm Sigil marker      | worldserver | none | `--scenario runemaster-palm-sigil-fire-earth-runeshroud` (coverage, not fail-before), plus the two existing gate scenarios                                                                         | level 24 Runemaster, `.learn 534803`, `.learn 805382`, `.aura 500288 1`: buttons no longer greyed, tooltip no longer says "Requires Runeshroud or Waveforged", `.cast 534803` applies. `.aura 500288 0` and they grey again                                    |
| Warpdagger             | worldserver | none | none written; the metric exists (`spellbook_learned_alerts`)                                                                                                                                       | level 10 Runemaster, `.learn 500287`, three out-and-back cycles: at most one new Warp button, and exactly 1 `SMSG_LEARNED_SPELL` for 500587 in the log (3 before)                                                                                              |
| Eternal Magic          | worldserver | none | none written; `spell_charges` is the metric                                                                                                                                                        | learn 707141 and 806698, spend two Runeblade charges, `.cast 800732`: charges return to 3. Without 806698, to 2                                                                                                                                                |
| Bot vs player scaling  | worldserver | none | the four existing scaling scenarios in both module modes; `level-scaling-damage-engagement` is a known baseline failure                                                                            | `DestinyWeaver.Enable=0`; level 2 in Sunstrider Isle, log in a level 22 bot within 20 yd: the Mana Wyrm's nameplate level must not rise (it did before, and fell back when the bot left)                                                                       |
| All branches           |             |      | `python -B tools/verify_all.py --base origin/main`                                                                                                                                                 |                                                                                                                                                                                                                                                                |

---

## 5. Verification run here

| Branch                          | `verify_all.py --stages source --base main` | `codestyle-cpp.py --files` | `run.py validate` | `git diff --check` |
| ------------------------------- | ------------------------------------------- | -------------------------- | ----------------- | ------------------ |
| `fix/barbarian-weapon-ranks`    | **PASSED** - 10 passed, 0 failed, 37.6 s    | PASSED                     | valid (32 steps)  | clean              |
| `fix/summon-kill-credit`        | **PASSED** - 10 passed, 0 failed, 37.3 s    | PASSED                     | valid (16 steps)  | clean              |
| `fix/runemaster-player-reports` | **PASSED** - 10 passed, 0 failed, 37.8 s    | PASSED (3 files)           | valid (20 steps)  | clean              |
| `fix/mob-scaling`               | **PASSED** - 5 passed, 0 failed, 1.2 s      | PASSED                     | n/a               | clean              |

No SQL files were added, so `codestyle-sql.py` had nothing to check. No new source file was
added, so **the PC does not need to re-run CMake configure** for any of these branches.

`VERIFY ALL: PASSED` is only a statement about the `source` stage. The `build`, `unit`,
`harness` and `gameplay` stages were all `SKIPPED`: this machine cannot run them. Every claim
about behaviour in this document and in the PR drafts is source- and data-level analysis.

---

## 6. G6 design summary

Full document: **`mob-scaling-design.md`**. In short:

- **Two implementations, one active at a time.** The realm-wide path
  (`AscensionCompat.cpp:6109-6290`) mutates the shared `Creature` with `SelectLevel()` and an
  armour restat, so one level is broadcast to everybody. The per-viewer path
  (`modules/mod-destiny-weaver/src/destiny_weaver_scaling.cpp`, `ViewFor` :203 and
  `OnPatchValuesUpdate` :611) rewrites level, health and power into the outgoing packet per
  recipient and leaves the server object alone. `LocalLevelScaling::CreatureScalingOwnedPerViewer`
  is the single switch, set by `ApplyTuning()` (:882) and honoured by `CanScaleCreature`.
- **The shipped configuration selects the per-viewer path**, so #1471, #4382 and #5134's
  shared-level symptoms should not occur on a default realm, and the reporter cannot see which
  mode they run. The recommended cheap addition is one `LOG_INFO` naming the owner.
- **#1471 and #4382** need the per-viewer rework the maintainer already tracks; the realm-wide
  path cannot express a per-viewer level at all.
- **#5134's second cause** is a real, undocumented rule: `OnAllCreatureUpdate` only re-evaluates
  a creature while it is out of combat **and** at full health, once per second. A creature that
  took one stray hit keeps the level it was scaled to for the rest of the fight.
- **#211** is a design decision. Loot is filled from the template on both paths and
  `docs/coa/level-scaling.md:203-204` says that is deliberate ("one corpse shared by everyone who
  tagged it"). The hook that would be needed, `OnAfterLootTemplateProcess`
  (`LootMgr.cpp:563`), exists; the unanswered questions are whose level applies and how grouping
  works.
- **Documentation defects found:** `docs/coa/level-scaling.md:33-34` and
  `level-scaling-vs-main.md` claim `CoA.LevelScalingMaxLift` defaults to 0 while
  `coa.conf.dist:140` ships 5, and neither documents that 0 also switches `DesiredLevel` from
  "nearest" to "highest in sight".

---

## 7. Open questions and risks for the reviewer

1. **The Palm Sigil fix is the one change that cannot be verified here.** It rests on the ADB
   saying 808089 is passive and on `Aura::CanBeSentToClient` dropping passive auras. If the
   client DBC differs, the startup log says so and the grey-out has another cause. A
   non-passive aura can also be cancelled by the player, which a passive one cannot - if that
   matters, add `AURA_INTERRUPT_FLAG_ON_CANCEL` to 808089 or re-sync it on a timer.
2. **Run 3's core revisions for the G3 reports (`a72bd1c171ed`) and G5a #4138's sibling
   (`a72bd1c171ed` in #4588) are not in this clone's history**, so #5124, #5135, #4588 and #5134
   could not be dated against the commits that touched this data. The G4 and G6 revisions all
   resolved.
3. **G4 is deliberately partial.** The "no player input" half of #4875 and #4119 needs
   `m_ControlledByPlayer` on Necromancer minions. That flag gates 44 call sites in
   `src/server/game`, including spell scaling and AoE targeting, and I could not compile or
   measure any of it. It is the obvious next G4 unit of work.
4. **G5b's 29 gaps are unverified in both directions.** Each needs either a `spell_proc` row
   whose ProcFlags/SchoolMask cannot be checked without the client DBC, or a new damage-path
   hook. The 18 proc-shaped ones share one mechanism and are the recommended next unit; the
   Cultist audit (PR #4717) is the model for how to land them with per-clause scenarios.
5. **G5b obtainability has one caveat.** Talent rank chains live in the client's
   `CharacterAdvancement.dbc`, read at runtime by `AscensionCoATalentData.cpp`, which this
   machine cannot read. 13 of the 145 reports name a higher rank, and a higher rank whose rank-1
   sibling is a tree node would look like dead data. All 13 were checked against
   `SpellbookTreeSpellData.h` for their own id first, and none of the 75 "not obtainable" rows
   carries an ADB tree reference or a row/column, which is the strongest signal available.
6. **`AscensionStockCoefficientData.h` is a grep trap.** It lists ~2700 Runemaster spells, so it
   appears in nearly every search and says nothing about whether a mechanic works. Two of my own
   first-pass classifications had to be corrected because of it.
7. **No overlap with run 3's group 1 or 2.** No spell_group id was allocated (G2 owns 1039-1044
   and 1140) and `SetTalentRank` in `AscensionCompat.cpp` was not touched - the G6 change is in
   the level-scaling block at :6188, 700 lines away. The three files G6 changes are all in
   `src/server/coa/`, and no new file was added.
8. **Same data artefact, other classes.** The G3 bug (a two-handed-only slot mask on a record
   whose subclass list includes one-handed types) also exists on Templar Argent Blade
   807263-807268 and Blade Tempest 8054099, and Reaper Desolate 500425-500428. Left out of a
   Barbarian branch on purpose; the same one-line invariant would close them.

---

## 8. Bundle for the PC (next step, after this run finishes)

Nothing has been pushed. To move the branches to the Windows PC for build and test, bundle them on the Mac (only the
commits on top of main 4c3a06dd0, which the PC already has):

```bash
cd ~/src/azerothcore-wotlk-coa-groups
git bundle create ~/src/groups-run/groups-branches.bundle \
  $(git for-each-ref --format='%(refname:short)' 'refs/heads/fix/*') ^main
git bundle verify ~/src/groups-run/groups-branches.bundle
git bundle list-heads ~/src/groups-run/groups-branches.bundle
```

Branches expected (final): `fix/barbarian-weapon-ranks`, `fix/summon-kill-credit`,
`fix/runemaster-player-reports`, `fix/mob-scaling`. There is deliberately **no**
`fix/runemaster-audit` (see the audit document) and no combined `fix/runemaster-sweep`; the Runemaster
work is one branch, four commits, and the G5b sweep produced a document rather than a branch.

On the PC (once the other job has released the checkout), fetch into the main checkout without touching other branches:
`git fetch <path-to>/groups-branches.bundle 'refs/heads/fix/*:refs/heads/fix/*'`, then build and verify each branch
serially with `tools/verify_all.py` as described in §4. No CMake reconfigure is needed: no new source
file was added by any branch. Two documents should travel with the branches for the reviewer:
`runemaster-audit.md` and `mob-scaling-design.md`.
