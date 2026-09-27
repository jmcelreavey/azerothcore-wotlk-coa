# Review: OpenCode groups 3-6 fix branches

Scope: the four branches from `.review-run/HANDOFF.md`, each one based on `main` `4c3a06dd0`. I reviewed every fix
hunk line by line against its registration, the point where its hook fires, and the objects that exist at that
moment. I checked each hunk for the known failure patterns in `.review-run/BRIEF.md`.

**Review fixes: pushed.** Pushes to the fix branches were accepted, so every review commit is on its fix branch
itself and no patches were exported. None of OpenCode's commits was changed. My first review commit on
`fix/runemaster-player-reports` (`c900b32`) was re-authored and replaced with `4bd8bcf` via `--force-with-lease`; that
replaced only my own commit on top of `4917df4`.

**Live-parity sources: blocked.** The sandbox's egress proxy refuses a CONNECT (403) to
`ascension-db.ascension-archive.workers.dev`, `hertigservices.github.io` and `ascension-archive.vercel.app`, and
the `hertigservices/ascension-data` repository is outside this session's GitHub scope. Nothing below is
re-checked against the ADB. The ADB facts quoted in the handoff are carried forward as the author's claims, and my
own checks use repo data only: `AscensionSpellProgressionData.h`, `SpellbookTreeSpellData.h`,
`SpellbookNotifyData.h`, `item_template` and the source.

Verdict totals across the 15 issues (11 closed with `Fixes`, 4 partial with `Refs`): **8 CORRECT, 7
FIXED-IN-REVIEW, 0 STILL BROKEN**. #4011, #4875 and #4119 count as CORRECT because the partial fix is right; the
rest of #4875 and #4119 is deferred. The deferred items (the handoff's own and the new ones from this review) are
listed per branch.

---

## fix/barbarian-weapon-ranks (d53b64229)

### #5124 Brutal Swing higher ranks refuse one-handed weapons: **CORRECT**
### #5135 Decapitate higher ranks refuse one-handed weapons: **CORRECT**

- **Registration.** `ClearContradictingWeaponSlotRequirement` runs from `ApplyAscensionBarbarianSpellChanges`
  (`src/server/coa/AscensionBarbarian.cpp:68`). That function is called from `ApplyAscensionClassMechanics`
  (`AscensionClassMechanics.cpp:1062`), which the spell custom-attribute load reaches at
  `AscensionCompat.cpp:6079`. It therefore runs once for every `SpellInfo` at startup. Nothing on the target, the
  caster or a timing condition is involved.
- **Every rank is covered.** `AscensionSpellProgressionData.h:25-31` gives Brutal Swing as 500913 followed by
  500996-501002, and `:132-135` gives Decapitate as 804414 followed by 806905-806908. The ranges in the fix match
  `BARBARIAN_BRUTAL_SWING` and `BARBARIAN_DECAPITATE` in `AscensionClassMechanics12To17.cpp:55-66` exactly.
  806906, which has no mask, fails the `!EquippedItemInventoryTypeMask` guard and is left alone.
- **No later pass undoes it.** The only other writer of `EquippedItemInventoryTypeMask` is
  `RepairedRangedInventoryMask` (`AscensionClassMechanics.cpp:1160`). That function returns early when the mask is
  0 and also returns early for any melee subclass (`:961`).
- **The check path matches the claim.** `Item::IsFitToSpellRequirements` (`Item.cpp:912-921`) rejects a one-handed
  weapon when the mask is `1<<17`. For item 25 (Worn Shortsword, `InventoryType` 21, `WEAPONMAINHAND`), the
  scenario is valid: the main-hand clause at `:916` needs bit 21, and the mask does not have it.
- **Subclass constants.** `ITEM_SUBCLASS_WEAPON_SPEAR` is 17, per `ItemTemplate.h:361`.
- Minor: the commit message and scenario call item 25 a "dagger", but it is a one-handed sword. This is cosmetic
  and does not change the test.
- **Risk to confirm on the PC (not fixable server-side).** The client's own `Spell.dbc` carries the same mask,
  because that is where the ADB's "in Bag" rendering comes from. If the client pre-checks equipped-item
  requirements, the button can stay red and the error can appear with no server round trip. The server fix is
  still necessary, but it may not be sufficient on its own.
- **Risk to confirm on the PC (scenario).** I could not read Decapitate's `TargetAuraState` or its health gate. If
  it is an execute-style spell, casting it on a full-health target fails before `CheckItems` runs. The scenario's
  `cast_failure == 0` or `== 29` asserts would then fail for both ranks. That is a test fault, not a gameplay one.

**In game.** Level 26 Barbarian: `.additem 25`, equip it in the main hand, `.learn 500913`, `.learn 500996`,
`.learn 804414`, `.learn 806905`, then `.cast 500996` on a dummy. Expected: the button is not red, and the combat
log shows a Brutal Swing hit. Repeat with `.cast 806905`, lowering the target's health first if Decapitate has an
execute gate. Unequip and cast again: expect "Must have a ... equipped" (reason 29). The startup log should show
`Cleared the equipment slot requirement of Barbarian rank ...` 10 times (7 Brutal Swing + 3 Decapitate) and no
`Skipped unexpected Barbarian weapon requirement record` line.

**Deferred (from the handoff; I agree).** The same mask artefact exists on Templar Argent Blade 807263-807268 and
Blade Tempest 8054099, and on Reaper Desolate 500425-500428.

---

## fix/summon-kill-credit (34c06882f)

### #4011 Necromancer pets and loot flag: **CORRECT**
### #4875 Mob doesn't loot when killed by raised pets (Refs): **CORRECT as partial; remainder DEFERRED**
### #4119 Necromancer inconsistent XP attribution (Refs): **CORRECT as partial; remainder DEFERRED**

- **Hook and path.** `spell_ascension_necromancer_ability::Hit` → `Order(player, target, id)` for any
  `Command()` spell (family flag 16777216), 500991 or 805871 (`AscensionNecromancerAbilities.cpp:359-360`). The
  caster is checked to be the owning player (`:348`). The target falls back from the explicit target to the
  victim to the selection, and it is validated with `IsValidAttackTarget` before the new lines run
  (`AscensionNecromancerSummons.cpp:314-319`). No null object is reachable.
- **The mechanism really yields XP and loot.** `LowerPlayerDamageReq(health, true, level)`
  (`Creature.cpp:3951`) zeroes `_playerDamageReq` and sets `_damagedByPlayer`, so
  `IsDamageEnoughForLootingAndReward` (`:3946`) is true. The loot recipient is still set by the Ghoul's first hit:
  `Unit.cpp:1233-1234` → `SetLootRecipient(attacker)` → `GetCharmerOrOwnerPlayerOrPlayerItself`, and the owner GUID
  is set in `IsSummonedBy` (`AscensionNecromancerSummons.cpp:363`). `Unit::Kill` (`Unit.cpp:14914-14924`)
  therefore keeps the recipient and rewards the Necromancer.
- **No wider tagging.** An uncommanded minion kill still gives nothing, which is the rule the reporter and the
  maintainer agreed in #4011. The spells that `Responds()` maps to the Ghoul (504868 etc., `:142-147`) match the
  scenario.
- **Edge case, acceptable.** The tag is applied even if no minion is in range to respond. This is "a player used an
  ability on the mob", which the maintainer ruled should tag. `EnterEvadeMode` → `ResetPlayerDamageReq`
  (`Creature.cpp:3965`) clears it again after a leash.

**In game.** Level 20 Necromancer: `.learn 500971`, `.learn 504868`, raise a Ghoul, then `.cast 504868` on a hostile
mob and never attack it yourself. Expected: an XP gain line and a lootable corpse (sparkles). A same-level second
character sees no loot. Control: let a Ghoul kill a mob with no command. Expected: no XP and no loot.

**Deferred (from the handoff; I agree).** The "no player input" part of #4875 and #4119 needs `m_ControlledByPlayer`
on Necromancer minions, which gates 44 core call sites.

---

## fix/runemaster-player-reports (749b7cfcf, f36856f79, 60066584d, 4917df417 + review c900b32)

### #4760, #5175, #5410, #5414 Palm Sigil greyed out (Runeshroud or Waveforged marker 808089): **FIXED-IN-REVIEW (4bd8bcf, 9feb61d)**

- **The diagnosis holds.** `Aura::CanBeSentToClient` (`SpellAuras.cpp:1097-1099`) drops a passive aura unless it
  is an area aura or an `ABILITY_IGNORE_AURASTATE` aura. The marker is applied by `CastSpell(player, 808089, true)`
  in `SyncRuneshroudOrWaveforged` and `KeepRunicTempestMarker` (`AscensionRunemasterTalents.cpp:49-58, 88-93`), and
  those run from the aura apply and remove hooks of 500288, 500469 and the Runic Breakout window. The client greys
  a spell out by its `CasterAuraSpell`, so it needs to receive the aura.
- **Registration.** `ExposeRuneshroudOrWaveforgedMarker` is called from `ApplyAscensionRunemasterTalentContracts`
  (`AscensionCompat.cpp:6084`). The early returns in front of it (`:202-216`) are for other ids only.
- **Defect found in review.** The fix required `SpellFamilyName == 38` before it looked at the record at all. No
  repo data confirms the family of this marker, which was added to `Spell.dbc` on 2026-08-13. With any other
  family value, the passive flag stayed set and nothing was logged, so the whole fix would have been a silent
  no-op. That is the same "passes source checks, does nothing in game" shape this review is looking for.
  **4bd8bcf** keys the fix on the spell id alone. The existing dummy-effect check still logs an unexpected record.
- **Side effect of making the aura non-passive.** A non-passive aura is saved to `character_aura` on logout, and
  nothing re-evaluated it at login, so a saved marker could outlive the Runeshroud or window it stands for.
  **9feb61d** adds a `PLAYERHOOK_ON_LOGIN` script that runs `SyncRuneshroudOrWaveforged` and
  `KeepRunicTempestMarker`. It is registered in `AddSC_AscensionRunemasterTalents`, so no CMake change is needed.
- **Harness break introduced by `749b7cf`.** `apps/coa-tests/runemaster_passives` compiles this file against a stub
  fixture that has no `LOG_ERROR`, `IsAura`, `SpellInfo::Attributes`, `SPELL_ATTR0_PASSIVE` or `SPELL_AURA_DUMMY`.
  9feb61d adds those stubs, plus `PlayerScript`. That harness **already fails to compile on `main`** because its
  fixture has drifted about 20 symbols behind the source (`int32`, `ObjectAccessor`, `SpellEffectInfo`,
  `Aura::SetMaxDuration` and others). I compiled it with g++ on `main` and on the branch: after 9feb61d, the branch
  produces exactly `main`'s error set and adds nothing. Repairing the fixture is a separate `main` task.
- The new scenario `runemaster-palm-sigil-fire-earth-runeshroud` is coverage only. The harness casts on the
  server, so it cannot see the client-side grey-out. The handoff says as much.

**In game.** Level 24 Runemaster: `.learn 534803`, `.learn 805382`, then `.aura 500288`. Expected: the Palm Sigil
buttons light up, the tooltip loses the red "Requires Runeshroud or Waveforged" line, and `.cast 534803` applies.
`.unaura 500288`: the buttons grey out again. Relog with and without Runeshroud up, and the state must match. The
startup log must not contain `Skipped unexpected Runeshroud or Waveforged marker record`.

### #4138, #4820 Warpdagger action-bar copies (Refs): **CORRECT as partial**

- On main, `removeSpell(ReturnSpell)` erased the temporary Warp 500587 (`Player.cpp:3646-3650`, `NEW/TEMPORARY` →
  `erase`). `StartTravel`'s `GetSpellMap().find(child) == end()` guard (`AscensionRunemasterTravel.cpp:133-135`)
  then re-learned it on every travel, and each `_addSpell` sent a fresh learned packet. With the unlearn removed,
  Warp is learned once per session and never saved (`learnSpell(child, true)` is temporary).
- It stays inert. `spell_ascension_runemaster_return::CheckReturn` refuses the cast unless `CanReturn`, which
  needs a live marker and `HasActiveSpell(Warpdagger)`.
- **Not stated in the commit.** `ReturnSpell()` also maps Echo Rune 500270 → Echo Return 500272, so Echo Return
  now stays known as well. The same `CheckReturn` guard covers it, except for its own triggered expiry cast, which
  needs the marker path anyway. This is harmless, but the in-game check should include Echo Rune.
- The two `SMSG_SUPERCEDED_SPELL` packets per travel remain, as the commit says.

**In game.** Level 10 Runemaster: `.learn 500287`, then three out-and-back Warpdagger cycles. Expected: at most one
new Warp button, and one `SMSG_LEARNED_SPELL` for 500587 per session. Casting Warp with no dagger out fails with
"Can't do that right now". Repeat with Echo Rune 500270.

### #5254, #3651 Eternal Magic 806698 restores 3 Runeblade charges: **FIXED-IN-REVIEW (696ed17)**

- `Player::RestoreSpellCharge(spellId, count)` exists (`Player.h:1861`) and clamps to `MaxCharges`
  (`Player.cpp:17239-17249`). `KnownRank` resolves the Runeblade rank the player knows, so every rank is handled.
- `HasAura(806698, own GUID)` is correct for a self-cast passive talent.
- The added restore stays inside the existing `HasAura(SPELL_RIFTBLADE 92154)` branch. The tooltip's "now
  refreshes 3" reads as upgrading Riftblade's existing 1-charge refresh, so that matches.
- **Defect found in review.** The tooltip names only Primordial Blast, but `4917df4` also gave 3 charges after
  Smolder 801087. Smolder is its own trainer rank chain (`AscensionSpellProgressionData.h:2664-2669`,
  502623-502628), not an override of Primordial Blast. **696ed17** restricts the 3 charges to Primordial Blast.
- **Harness break introduced by `4917df4`.** The `runemaster_secondary` fixture stubbed
  `RestoreSpellCharge(uint32)` with one argument, so the new two-argument call does not compile. That harness uses
  MSVC, so it would fail on your PC. 696ed17 updates the stub and adds asserts: 1 charge from Smolder, 1 from
  Primordial Blast without the talent, and 3 from Primordial Blast with it. This harness also already fails to
  compile on `main` (`GetAuraEffect` and `EFFECT_1` are missing from its fixture). I stubbed those two in a scratch
  runner only, and there the harness **passes with 696ed17 and fails the Smolder assert on 4917df4**.
- Nit, not changed: `SPELL_ETERNAL_MAGIC_CHARGES = 3` sits in the spell-id enum.

**In game.** Riftblade Runemaster with 707141 and 806698: spend two Runeblade charges, then `.cast 800732`. Expected:
3/3 charges. Without 806698: 2/3. Spend two charges, then cast Smolder: 2/3 with or without the talent.

### Handoff verdicts spot-checked (G5a)

- #4588 (exclusive glyph rotation), #4490 / #5269 / #4421 (auto-granted tree node per #4436), and #4979 / #4188
  (need `SummonProperties.dbc`): the reasoning is consistent with the code. I have no evidence against them.

### Runemaster audit spot-check (G5b; not redone in full)

- I checked all 75 "not obtainable" ids against `SpellbookTreeSpellData.h` and `AscensionSpellProgressionData.h`.
  Only #3370 (804269) appears, and it is a class-27 node named "Deprecated", which `learnSpell` refuses. The
  verdict holds, but its evidence line ("no node") is inaccurate.
- I spot-checked the group-A and group-B gaps (520138, 520237, 800758, 803013, 705563): none has a script, a SQL
  row or a scenario. Those verdicts hold.
- **Doubtful "false positive" verdicts (DEFERRED, reclassify as possible dead clauses):**
  - #2731 Primordial Power (705556, 705557) carries `Apply Aura: Dummy 4`.
  - #2738 Focusing Crystals (705564, 707879) carries `Dummy: Unknown 5`.
  - #2133 Glyphic Infusion (800733) is three bare `Trigger Spell` effects.

  None of those ids appears anywhere in `src/`, `modules/`, the pending SQL or the scenarios, and a Dummy aura does
  nothing without a script. Each needs its tooltip checked: if the dummy is only a tooltip value holder, the
  verdict stands; otherwise the clause is dead.

---

## fix/mob-scaling (8904fcbe5)

### #4296 Mobs scaling off high-level bots: **CORRECT** (realm-wide path only)

- `DesiredLevel` (`AscensionCompat.cpp:6188-6228`) now makes two passes, non-bots first and bots second, and
  breaks after the first pass that found anyone. In "nearest" mode the nearest non-bot decides the level. In
  "highest" mode (`MaxLift = 0`) the highest non-bot decides, and bots count only when no player is in sight.
  `meilleure` is reset for each pass.
- `ScaleForEngager` (`:6257-6262`) returns before it writes the pending engager level or calls `SelectLevel()` for
  a bot, so a bot pull cannot reach `DesiredLevel`'s pending-engager override at `:6219-6225`.
- `WorldSession::IsBot()` exists (`WorldSession.h:1259`), and the file already uses it at `:663, :3274, :3441`. Null
  sessions are guarded.
- **Scope caveat.** `CanScaleCreature` returns false when `CreatureScalingOwnedPerViewer` is set (`:6111`). The
  shipped `destiny_weaver.conf.dist` enables the per-viewer path, whose `ViewFor`
  (`destiny_weaver_scaling.cpp:203-240`) scales only from the viewer's own level, so a bot cannot move it. On a
  default realm, this fix therefore changes nothing, and #4296 reproduces only with `DestinyWeaver.Enable=0` or
  `DestinyWeaver.LevelScaling=0`. The handoff's line that #4296 is "possible under the default configuration"
  contradicts its own design note. Treat the fix as correct for the fallback path.

**In game.** Set `DestinyWeaver.Enable=0`, `CoA.LevelScaling=1` and `CoA.LevelScalingMaxLift=5`. A level 2 character
in Sunstrider Isle, with a level 22 playerbot standing closer to a Mana Wyrm: the Wyrm's nameplate level stays in
the level-2 band and does not jump to about 19. Remove the player, and the bot alone scales the Wyrm as before.

**Documentation fixed in review (df5a7da).** `docs/coa/level-scaling.md` and `level-scaling-vs-main.md` said
`CoA.LevelScalingMaxLift` defaults to 0 and that the nearest character always decides the level. Both the dist file
and the `GetOption` fallback are 5, and at 0 the highest character in sight decides instead. The docs now say so,
and they record the new bot rule.

**Deferred (from the handoff; I agree).** #1471, #4382 and #211 are per-viewer design work. #5134's
full-health/out-of-combat re-evaluation rule is still unfixed.

---

## Review commits

| Branch | Commit | Delivery |
| --- | --- | --- |
| `fix/runemaster-player-reports` | `4bd8bcf` fix(CoA/Runemaster): expose the Runeshroud or Waveforged marker whatever its spell family | pushed |
| `fix/runemaster-player-reports` | `696ed17` fix(CoA/Runemaster): Eternal Magic's three charges come from Primordial Blast only | pushed |
| `fix/runemaster-player-reports` | `9feb61d` fix(CoA/Runemaster): re-sync the Runeshroud or Waveforged marker on login | pushed (head) |
| `fix/mob-scaling` | `df5a7da` docs(CoA/Scaling): document the shipped level-lift cap and the bot rule | pushed (fast-forward) |

No review commits on `fix/barbarian-weapon-ranks` or `fix/summon-kill-credit`: nothing on either branch can be
fixed from source here. The remaining items there are in-game confirmations, or the deferred scope above.

## Not fixed, and why

- **Barbarian: whether the client refuses the cast itself.** This depends on the client's own `Spell.dbc` and
  cannot be fixed on the server. Confirm it in game.
- **Barbarian: a possible execute gate on Decapitate in the scenario.** This needs the DBC's `TargetAuraState`,
  which is not in the repo. If the scenario fails with a reason other than 0 or 29, lower the target's health
  before the Decapitate casts.
- **Barbarian: Templar/Reaper copies of the slot-mask artefact.** These need the client DBC to confirm the ids;
  the handoff's `8054099` is not a valid 6-digit CoA id.
- **Summons: #4875 and #4119, kills with no player input.** The maintainer ruled this is not wanted, and it would
  need `m_ControlledByPlayer`.
- **Audit: doubtful verdicts #2731, #2738, #2133.** These need the tooltips, and the ADB is blocked here.
- **Stale harness fixtures on `main`** (`runemaster_passives`, `runemaster_secondary`). These predate these branches.

## Verification

- `fix/runemaster-player-reports` at 9feb61d: `python3 -B tools/verify_all.py --stages source --base main` gave
  `source PASSED 38.1 s 10 passed, 0 failed` and
  `VERIFY ALL: PASSED .cache/verify-all/20260927-192525/report.json`.
- `fix/mob-scaling` at df5a7da: the same command gave
  `VERIFY ALL: PASSED .cache/verify-all/20260927-192520/report.json`.
- Both results are for the source stage only. Build, unit and gameplay were SKIPPED.
  `--stages harness --harness runemaster_passives` is UNAVAILABLE here
  (`requires workspace tools missing ... Test-LocalLoginCollections.py`), so the two Runemaster harnesses were run
  by hand with g++, as described above.
- `python3 apps/codestyle/codestyle-cpp.py --files src/server/coa/AscensionRunemasterTalents.cpp
  src/server/coa/AscensionRunemasterSecondary.cpp`: "Everything looks good". On `AscensionRunemasterTravel.cpp`, the
  earlier run also passed.
- `codestyle-sql.py`: not run, because no SQL was added on any branch.
- Nothing was compiled or run in game. Every behaviour claim above comes from source and repo-data analysis, and
  the in-game checks listed are for John's PC. Timings, regressions and flaky checks: unknown, because nothing was
  measured.
