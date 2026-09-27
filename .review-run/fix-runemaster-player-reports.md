# fix(CoA/Runemaster): Palm Sigil gate, Warpdagger action bar and Eternal Magic

## Summary

Three separate Runemaster defects, one commit each:

1. **Palm Sigils are greyed out in Runeshroud.** The server grants the
   `CasterAuraSpell` marker the sigils are gated on, but the marker is a passive
   aura, and the core never sends a passive aura to the client - so the client
   never learns the player has it and refuses the spell whatever the player does.
   The marker is made client-visible.
2. **Warpdagger copies itself onto the action bar on every use.** The travel code
   unlearned the return spell and learned it again on every cycle, and the CoA
   client auto-places a newly learned spell. The unlearn is removed.
3. **Eternal Magic does not refresh three Runeblade charges.** The tooltip clause
   was dead; the code restored one charge unconditionally.

## Root cause and fix, per issue

**#4760, #5175, #5410, #5414 - Palm Sigils unusable in Runeshroud.** Every Palm
Sigil carries `CasterAuraSpell = 808089`, the dummy aura named "Runeshroud or
Waveforged", which `SyncRuneshroudOrWaveforged`
(`src/server/coa/AscensionRunemasterTalents.cpp:48-57`) grants while Runeshroud
500288, the Waveforged window 500469 or the Runic Breakout window 520767 is on the
player. The Ascension DB shows 808089 was added to the client's `Spell.dbc` on
2026-08-13 **with the passive flag set**
(<https://ascension-db.ascension-archive.workers.dev/#search?kind=spell>,
record 808089), and

```cpp
bool Aura::CanBeSentToClient() const
{ return !IsPassive() || GetSpellInfo()->HasAreaAuraEffect() || HasEffectType(SPELL_AURA_ABILITY_IGNORE_AURASTATE); }
```

(`src/server/game/Spells/Auras/SpellAuras.cpp:1096-1099`) drops a passive aura, so
the client never receives 808089. That is why the tooltip keeps reading
"Requires Runeshroud or Waveforged" and the buttons stay greyed: the four
reporters describe a _client_ refusal, not a server one. The server half is
already correct and already tested - `runemaster-palm-sigil-runeshroud.json` and
`runemaster-waveforged-palm-sigil-gate.json` prove the marker syncs with Runeshroud
in both directions and that the sigil is refused with
`SPELL_FAILED_CASTER_AURASTATE` (22) without it.

`ExposeRuneshroudOrWaveforgedMarker` in `AscensionRunemasterTalents.cpp` clears
`SPELL_ATTR0_PASSIVE` on that one record, and logs it instead of changing it when
its first effect is no longer the expected dummy.

#5414 was filed against `4c3a06dd0399`, this branch's base, so it was live at the
time of writing; #5410 against `47dd22ffe6d0` (one day earlier). All four
reports are the same defect. Note that only Arcane 805380 was ever covered by a
scenario, and the two sigils named in the reports - Fire 534803 and Earth 805382

- were not, which is why the existing regression did not catch this.

**#4138, #4820 - Warpdagger copies itself onto the action bar.** A comment on
#4138 attributes this to `Player::SetTemporarySpellReplacement` sending
`SMSG_SUPERCEDED_SPELL` twice per travel
(`src/server/game/Entities/Player/Player.cpp:13837-13841`) and the client
auto-placing the new spell. That part is right, but the comment's premise - that
`learnSpell(SPELL_WARP)` is "guarded, only when the player does not already know
it" - is not: `ClearTravel` unlearned the return spell immediately before
`StartTravel` checked, so the guard was defeated on every cycle after the first.
`ClearTravel` no longer removes the return spell. It cannot: `SetTemporarySpellReplacement`
requires `HasActiveSpell` on both halves of the pair
(`Player.cpp:13830-13832`), and `spell_ascension_runemaster_return::CheckReturn`
already refuses the cast with `SPELL_FAILED_CANT_DO_THAT_RIGHT_NOW` once the
marker is gone, so a permanently known Warp is inert rather than broken. This is a
`Refs`, not a `Fixes`: the two SUPERCEDED packets per travel remain, so the
client may still place one copy per use. Working around those means the 38-file
`SetTemporarySpellReplacement` blast radius the comment already weighed.

**#5254, #3651 - Eternal Magic does not refresh three Runeblade charges.** 806698
reads "Reduces the cooldown of Primordial Blast by -4 sec and it now refreshes 3
charges of Runeblade." The -4 sec half is an `ADD_FLAT_MODIFIER -4000` spell mod
and already works. The charges were dead: `runemaster_secondary_casts::OnSpellCast`
restored exactly one Runeblade charge after any Primordial Blast or Smolder
(`src/server/coa/AscensionRunemasterSecondary.cpp:145-149`) whatever the talent
said. It now restores three while the player holds Eternal Magic and one
otherwise. #3651 is the automated audit report for the same spell, so it is
closed here.

**#4588 - Frost Glyph not refreshing. Not a bug as designed (no change).** The
Glyph passives are an exclusive rotation: the ADB tooltip for Arcane Glyph 805727
is "Using Elemental Burst or Primordial Blast **while Flame Glyph is active** now
grants you Arcane Glyph", and `GenerateGlyph`
(`src/server/coa/AscensionRunemasterGlyphs.cpp:78-90`) casts exactly one glyph per
use for that reason. The reporter's own expectation - "either both refresh or not"

- is a design choice, and refreshing the lower glyph would break the rotation.
  Logged for a maintainer decision rather than changed.

**#4490, #5269, #4421 - free level-10 Glyphic Ruin. Working as designed (no
change).** 801179 is a class-32 node in `SpellbookTreeSpellData.h:3919`, so the
client's talent tree grants its first rank, and #4436's merged fix requires a
specialization tree's free (AECost 0, TECost 0) identity nodes to be granted
automatically - `knight-spec-tree-free-nodes.json` is that regression. #4421
("I can't choose the talent", level 11) is the same behaviour seen from the other
side: the node is already granted. The reporters believe the spec talent should
arrive at 11; the client's own tree data says otherwise. One check the reviewer
can do with a client DBC: `CharacterAdvancement.dbc`, the row for 801179, dwords
14 and 15 (AECost, TECost) - both should be 0.

**#4979, #4188 - Homebound Runestone 560316 and Scrying Orb 804674 create no
object. Deferred (no change).** Both are a single `SPELL_EFFECT_SUMMON` (MiscValue
1 and 5 respectively, per the ADB) whose summon template has no data. The ADB's
NPC list contains exactly one plausible candidate, creature **50038 "Scrying
Orb"**, and no "Homebound Runestone" creature at all, and neither id appears in
`SpellbookTreeSpellData.h` or `AscensionSpellProgressionData.h` beyond a level
grant (`src/server/coa/AscensionCustomClassData.h:845,853`). Wiring the effect
needs `SummonProperties.dbc` (the group and template behind MiscValue 5), which is
client data this machine does not have, and the fix would be a new
`creature_template` row plus whatever script the orb needs for "control directly
and see through". Not guessable from the tooltip.

## Tests to run on the PC

```sh
python -B tools/verify_all.py --stages build,gameplay --scenario runemaster-palm-sigil-fire-earth-runeshroud
python -B tools/verify_all.py --stages build,gameplay --scenario runemaster-palm-sigil-runeshroud
python -B tools/verify_all.py --stages build,gameplay --scenario runemaster-waveforged-palm-sigil-gate
python -B tools/verify_all.py --stages build,gameplay --scenario runemaster-tattoo-rank-supersede
python -B tools/verify_all.py --base origin/main
python apps/codestyle/codestyle-cpp.py --files src/server/coa/AscensionRunemasterTalents.cpp src/server/coa/AscensionRunemasterSecondary.cpp src/server/coa/AscensionRunemasterTravel.cpp
```

The new `runemaster-palm-sigil-fire-earth-runeshroud` is **server-side coverage
only**: it asserts that Fire and Earth are refused with reason 22 without
Runeshroud and complete with it, which passes both before and after this change,
because the server half was already correct. It closes a real coverage hole - the
two sigils the reports name were never exercised - but it is **not** a
fail-before/pass-after test and does not prove the fix. The fix is only observable
in a client.

In game, for each of the three fixes:

- _Palm Sigils._ Level 24 Runemaster, `.learn 534803`, `.learn 805382`, `.aura
500288 1` (or cast Runeshroud). Pass: the sigil buttons are **no longer greyed
  out**, the tooltip no longer says "Requires Runeshroud or Waveforged", and
  `.cast 534803` applies the aura. Then `.aura 500288 0` and confirm the buttons
  grey out again. If they are still grey with the aura up, the client DBC's
  passive bit for 808089 is not what the ADB shows - stop and report it.
- _Warpdagger._ Level 10 Runemaster, `.learn 500287`, cast Warpdagger, cast Warp to
  return, repeat three times. Pass: the action bar gains at most one new Warp
  button (the first learn) and not one per use. Count the `SMSG_LEARNED_SPELL`
  packets in the log for 500587: exactly 1 across the whole sequence, where it
  was 3 before. The `spellbook_learned_alerts` metric
  (`src/server/coa/CoAGameplayTest.cpp:1686-1691`) counts them if a scenario is
  written for it.
- _Eternal Magic._ Learn 707141 (Runeblade, 3 native charges from
  `SpellCharges.dbc`) and 806698. Spend two charges, then `.cast 800732`
  (Primordial Blast). Pass: `spell_charges` for 707141 returns to 3 with Eternal
  Magic and to 2 without (`.aura` the spell or read the charge counter in the
  spellbook). `SPELL_EFFECT_ADD_FLAT_MODIFIER -4000` on the talent is the
  cooldown half, which needs no check.

## Risks

- **Making 808089 non-passive is a behaviour change beyond the client view.** A
  passive aura cannot be cancelled by the player; a non-passive one can. A
  player who right-click-cancels the marker while in Runeshroud would grey the
  sigils out again until Runeshroud is reapplied, because
  `SyncRuneshroudOrWaveforged` only runs on aura apply/remove. Nothing on the
  server reads 808089 other than `HasAura` in that same function, and the aura is
  a `SPELL_AURA_DUMMY` with no mechanical effect, so the server-side blast radius
  is nil; the cancellation window is the one thing to watch. If it matters, set
  `AURA_INTERRUPT_FLAG_ON_CANCEL` on the record or re-sync on a timer.
- **The Palm Sigil fix is unverified against a client.** Everything above is
  derived from the ADB record and `Aura::CanBeSentToClient`. The startup log
  (`Skipped unexpected Runeshroud or Waveforged marker record`) is the guard; if
  the record's first effect is not a dummy, the marker was already client-visible
  and the grey-out has another cause.
- Warpdagger: the return spell stays in the spellbook after the first travel. It
  fails to cast when no marker exists, so it is inert, but it is visible.
- Eternal Magic: three charges is the tooltip's number and Runeblade's maximum
  (`MaxCharges` 3 from `SpellCharges.dbc`, which `ApplyClientSpellCharges`
  already reads), so `RestoreSpellCharge(rank, 3)` is a top-up to full, not an
  over-charge.

## Not addressed

- #4138 / #4820's residual `SMSG_SUPERCEDED_SPELL` auto-place (38 files).
- #4588 Frost Glyph refresh - design question.
- #4490, #5269, #4421 Glyphic Ruin - working as designed.
- #4979, #4188 Homebound Runestone and Scrying Orb - need `SummonProperties.dbc`.
- The 29 confirmed dead clauses from the 145-report automated audit
  (`~/src/groups-run/runemaster-audit.md`). They are one class-wide unit of work,
  not part of this PR.

Fixes #4760
Fixes #5175
Fixes #5410
Fixes #5414
Fixes #5254
Fixes #3651
Refs #4138
Refs #4820
