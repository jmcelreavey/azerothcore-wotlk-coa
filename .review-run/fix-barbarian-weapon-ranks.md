# fix(CoA/Barbarian): accept one-handed weapons on Brutal Swing and Decapitate higher ranks

## Summary

Brutal Swing rank 2 and Decapitate rank 2 both stop working the moment a
Barbarian buys the next rank: the client reports "Must have a melee weapon
equipped" for a character who was using the same spell, with the same weapons,
one rank earlier. The cause is in the client spell data, not in the player's
gear. Both chains' higher ranks carry an `EquippedItemInventoryTypeMask` that
admits only a two-handed weapon, while the same record's
`EquippedItemSubClassMask` still lists one-handed axes, maces, swords, fists,
daggers and spears. A dual-wielding Barbarian is rejected by the higher rank and
accepted by rank 1.

`ApplyAscensionBarbarianSpellChanges` now clears the slot mask for those ranks
when the record's own subclass list spans both one- and two-handed weapons, and
logs the record instead of changing it when the expected shape is missing.

## Root cause and fix, per issue

**#5124 - Brutal Swing 500996 (ranks 2-8).** `EquippedItemInventoryTypeMask` is
`1 << INVTYPE_2HWEAPON` (131072) on 500996-501002 and 0 on rank 1 (500913).
`Item::IsFitToSpellRequirements` (`src/server/game/Entities/Item/Item.cpp:912-921`)
rejects a one-handed weapon against that mask, and
`Player::HasItemFitToSpellRequirements`
(`src/server/game/Entities/Player/Player.cpp:13124-13158`) therefore finds no
equipped weapon that fits, so `Spell::CheckItems`
(`src/server/game/Spells/Spell.cpp:7421-7422`) returns
`SPELL_FAILED_EQUIPPED_ITEM_CLASS` (29). The Ascension DB renders the extra
requirement on exactly those ranks and not on rank 1
(<https://ascension-db.ascension-archive.workers.dev/#search?kind=spell>,
records 500913 / 500996 / 501002).

**#5135 - Decapitate 806905.** Same mask on 806905, 806907 and 806908. Rank 3
(806906) does not have it, so the ability works at level 34 and fails at levels
26, 44 and 52 - a per-rank design change would not skip a middle rank, which is
what proves this is stray data. The reporter dual-wields two one-handed maces
("Venerable Mass of McGowan"), which rank 1 accepted.

The neighbouring `RepairedRangedInventoryMask`
(`src/server/coa/AscensionClassMechanics.cpp:958-972`) cannot help: it returns
the mask unchanged for any subclass mask that contains a melee subclass
(line 961), so the melee chains were never repaired.

**Fix.** `ClearContradictingWeaponSlotRequirement` in
`src/server/coa/AscensionBarbarian.cpp`, called from
`ApplyAscensionBarbarianSpellChanges`, which the existing
`GLOBALHOOK_ON_LOAD_SPELL_CUSTOM_ATTR` hook already reaches
(`src/server/coa/AscensionCompat.cpp:6079`). For a spell in 500996-501002 or
806905-806908 whose item class is a weapon, whose subclass mask contains at least
one one-handed and at least one two-handed subclass, and whose slot mask is
non-zero, the slot mask becomes 0. A record outside that shape logs
`Skipped unexpected Barbarian weapon requirement record {}` so a different client
DBC is visible at startup. The equipped-weapon requirement itself is untouched.

**Sweep.** All 26 Barbarian rank chains in `AscensionProgression::Ranks` were
compared. The only other inventory-slot mismatch is Throw Weapon 806876-806881,
whose subclass is `THROWN` (16) and is already repaired by
`RepairedRangedInventoryMask`, which adds `1 << INVTYPE_THROWN` for that
subclass. Decapitate 806904 and 807116 carry the same artefact but appear in
neither `AscensionProgression::Ranks` nor `SpellbookTreeSpellData.h`, so they are
not obtainable and are left alone.

## Tests to run on the PC

```sh
python -B tools/verify_all.py --stages build,gameplay --scenario barbarian-weapon-requirement-ranks
python -B tools/verify_all.py --base origin/main
python apps/codestyle/codestyle-cpp.py --files src/server/coa/AscensionBarbarian.cpp
```

`barbarian-weapon-requirement-ranks` casts rank 1 and rank 2 of both chains with
a one-handed dagger equipped, requires `cast_failure == 0` and
`spell_cast_count == 1` for each, and keeps a second, unarmed Barbarian as the
negative control that must still get reason 29. **It passes on `main` for rank 1
and fails for rank 2**, so it is the fail-before/pass-after check.

In game: level 26 Barbarian, `.additem 25` (Worn Dagger) in the main hand,
`.learn 500913`, `.learn 500996`, `.learn 804414`, `.learn 806905`, then
`.cast 500996` on a target dummy. Pass: the cast completes and the target loses
health. Before the fix the same command returns "Must have a melee weapon
equipped". Repeat for `.cast 806905`. Then `.aura` nothing / unequip the dagger
and confirm the spell is refused again, so the equipped-weapon requirement
survives.

Startup log: no `Skipped unexpected Barbarian weapon requirement record` line. If
one appears, the client DBC does not have the shape this commit expects and the
fix has not applied - stop and report it.

## Risks

- The fix depends on the client DBC having the shape run 3 read. The guard turns
  a shape mismatch into a startup error rather than a silent no-op.
- Clearing the mask makes ranks 2+ accept exactly the weapons rank 1 accepts.
  If Ascension ever intended a 2H-only rank, the ADB would have to say so in the
  tooltip; it does not ("Requires: Axe or Two-Handed Axe or Mace or ... or
  Dagger" on both).
- Not observable in the gameplay harness: the harness casts server-side, so it
  cannot show a client-side greyed-out button. This is a server-side cast failure
  (`SPELL_FAILED_EQUIPPED_ITEM_CLASS`), which the harness does observe.
- The same data artefact exists in three other classes (Templar Argent Blade
  807263-807268 and Blade Tempest 8054099, Reaper Desolate 500425-500428). Left
  out of this branch on purpose; the class-scoped change is the reason.

## Not addressed

- Decapitate 806904 / 807116: same artefact, not obtainable.
- Templar and Reaper copies of the same defect, for a follow-up.
- The reported `a72bd1c171ed` core revision in #5124 and #5135 is not in this
  clone's history, so neither report could be dated against the commits that
  touched this data.

Fixes #5124
Fixes #5135
