# G6 design - mob level scaling seen through the wrong player

Scope: #1471, #4296, #4382, #1444, #5134, #211.
Branch: `fix/mob-scaling` (one commit, bot handling only - see §1).
Nothing here was compiled, run or reproduced: this machine has no build
toolchain, database, server or client. The only verification run was
`python3 -B tools/verify_all.py --stages source --base main` (PASSED) plus
`codestyle-cpp.py --files src/server/coa/AscensionCompat.cpp`.

## 0. The two paths, and which one a report can even be about

CoA has **two** mutually exclusive level-scaling implementations. Which one runs
is decided by one atomic flag.

|                | realm-wide path                                                                                                                                                                    | per-viewer path                                                                                                                              |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| code           | `src/server/coa/AscensionCompat.cpp:6109-6290` (`CanScaleCreature`, `AscensionCompatLevelScalingScript::DesiredLevel`, `AscensionCompatLevelScalingEngageScript::ScaleForEngager`) | `modules/mod-destiny-weaver/src/destiny_weaver_scaling.cpp` (`ViewFor` :203, `OnPatchValuesUpdate` :611, per-attacker damage hooks :658-717) |
| ownership      | mutates the **shared** `Creature`                                                                                                                                                  | rewrites level/health/power **in the outgoing packet per recipient**                                                                         |
| switch         | runs when `LocalLevelScaling::CreatureScalingOwnedPerViewer` is false (`CanScaleCreature`, `AscensionCompat.cpp:6111`)                                                             | sets that flag in `ApplyTuning()` (`destiny_weaver_scaling.cpp:882`) when `DestinyWeaver.Enable && DestinyWeaver.LevelScaling`               |
| shipped config | `CoA.LevelScaling = 1`, `CoA.LevelScalingMaxLift = 5` (`src/server/coa/conf/coa.conf.dist:81,140`)                                                                                 | `DestinyWeaver.Enable = 1`, `DestinyWeaver.LevelScaling = 1` (`modules/mod-destiny-weaver/conf/destiny_weaver.conf.dist:24,34`)              |

**With the shipped `.dist` values the per-viewer path wins and the realm-wide
path is inert.** Every symptom below that says "a mob has one level for
everybody", "a mob's health and damage change when a high-level player engages
it" or "mobs scale off the highest player nearby" can therefore only happen on a
realm that has turned `DestinyWeaver.Enable` or `DestinyWeaver.LevelScaling` off
(or does not load the module). That is a real configuration - the fork ships the
fallback on purpose - but it changes what each report means, and the reporter
cannot see which mode their realm runs.

A documentation defect makes this worse: `docs/coa/level-scaling.md:33-34` says
`CreatureMaxLift` "is **0 (no ceiling) by default here**", and
`docs/coa/level-scaling-vs-main.md` repeats it, while `coa.conf.dist:140` ships
`5`. The `.dist` is the effective value; the doc is wrong. See §4.

## 1. #4296 - mobs scaling off high-level bots (FIXED on this branch)

**Report.** Level 2 in Sunstrider Isle; mobs were the character's level until a
level 22 bot entered the area, at which point they jumped to about 19 and stayed
there until the bot left. Core `7b66e4380261` (2026-09-18).

**Root cause (realm-wide path).** `DesiredLevel`
(`src/server/coa/AscensionCompat.cpp:6188-6220`) walked `map->GetPlayers()` with
filters for alive / not a GM / same phase / within `GetSightRange()` / valid
attack target - and **no bot filter**. With `useNearestPlayer` (which is
`CreatureMaxLift != 0`, i.e. true at the shipped value of 5) the _nearest_
qualifying session wins, so a bot that happened to be closer than the player
took the creature's level. `ScaleForEngager` (:6247) had the same gap: a bot's
first hit or pull wrote the pending engager level, which `DesiredLevel` then
preferred over everyone in sight.

**Fix.** Prefer the nearest non-bot session; fall back to the nearest bot only
when no player is in range, so a bot-only area behaves exactly as before. A
bot's pull no longer writes the pending engager level. GM exclusion is unchanged.

**Why this is the right rule.** `docs/coa/level-scaling.md` states the intent
outright: "The **nearest** character decides the level either way, never the
highest level in sight: taking the maximum hands one player's level to
everybody." The rule is written about _characters_; a bot is not one, and the
`CoA.LevelScalingMaxLift` comment in the `.dist` names bots as the case the cap
exists for ("the low level bots fighting them are killed by creatures that were
never theirs"). A hard `IsBot()` exclusion elsewhere in the same file confirms
the convention (`AscensionCompat.cpp:663,3274,3441`,
`AscensionHighRisk.cpp:335,528`).

**Risk.** Low, and bounded. A realm that _wants_ bots to drive scaling gets the
old behaviour only where no player is in range. The per-viewer path is not
touched. No gameplay scenario uses `bot: true` except two that are not scaling
scenarios, so `destiny-weaver-scaling`, `level-scaling-damage-engagement`,
`skinning-dungeon-scaled-view` and `skinning-open-world-level-scaling` are
unaffected either way (they use real sessions).

**Test plan (PC).**

1. `CoA.LevelScaling=1`, `CoA.LevelScalingMaxLift=5`,
   `DestinyWeaver.Enable=0` (force the realm-wide path).
2. Level 2 character in Sunstrider Isle. `.gps on`, then log in a level 22 bot
   inside 20 yd of a Mana Wyrm.
3. Pass: the Wyrm's nameplate level does not rise; with the fix reverted it
   rises to about 19 and drops back when the bot leaves.
4. Then move the character out of the bot's area and `.gps off` the bot: with no
   player in range the mob may fall back to the bot's level - that is the
   documented fallback, not a regression.
5. Re-run `python -B tools/verify_all.py --stages build,gameplay --scenario
level-scaling-damage-engagement` and `--scenario destiny-weaver-scaling`
   with `DestinyWeaver.Enable=1` to confirm the per-viewer path is unaffected.
6. `python -B tools/verify_all.py --base origin/main` before the PR.

## 2. #1471 - one scaled level for the whole party (NEEDS THE PER-VIEWER REWORK)

**Report.** "Scaling always shows the highest lvl player's scaled mob lvl to
others in the party, making them unable to do anything but watch & miss
everything." Level 51, map 1, core `be725043c415` (2026-09-16). The maintainer
already confirmed the mechanism in a comment and pointed at this issue as the
tracking ticket: "Confirmed: a mob takes one scaled level for the whole group,
from the highest-level player in range. Per-player levels like on Ascension need
a larger rework, kept open for it."

**Why the per-viewer path is the fix and not this branch.** The realm-wide path
cannot express "a different level per viewer": it calls `creature->SelectLevel()`
and rewrites `UNIT_MOD_ARMOR` on the shared object (`AscensionCompat.cpp:6165,
6180, 6247`), and `Unit::Kill` / threat / combat scaling then all read that one
number. The per-viewer path already exists and already does the right thing -
`ViewFor` computes a `CreatureView` per (creature, viewer) and
`OnPatchValuesUpdate` writes level/health/power into the outgoing
`SMSG_UPDATE_OBJECT` for each recipient, leaving the server-side object at its
authored values. `LocalLevelScaling.h:169-184` documents exactly this division
of labour, including `ScaleDungeonCreatureLevelForViewer` (:191) which also
brings a creature _down_ inside a dungeon, which is what #1444 asks for.

**So the recommendation is configuration, not code**: the reports are real on a
realm running the realm-wide path and should not occur on the shipped default.
Two things are needed:

1. **Make the mode visible at runtime.** Nothing logs which path owns scaling. A
   single `LOG_INFO("coa.scaling", "level scaling owner: {}", ...)` in
   `ApplyTuning()` and in `OnBeforeConfigLoad`, naming per-viewer vs realm-wide,
   would have answered four of these six reports on its own.
2. **Reconcile the docs with the `.dist`** (§4) so a realm owner reading the
   documentation picks the right setting.

**Not attempted here.** Turning the per-viewer path on for realms that chose the
other one is a deployment decision, and rewriting the realm-wide path to be
per-viewer is exactly the larger rework #1471 already tracks.

## 3. The remaining four

### #4382 - a mob really gains levels, it is not only the nameplate

**Report.** A level 60 fights Stitches; he scales to 57 "as expected". A level 26
character then sees him still at 57, with the level 60 health pool and crushing
blows; he resets when combat ends. Core `b3717c137710` (2026-09-19).

**Which path.** Realm-wide. `ScaleForEngager` (:6247) calls `creature->SelectLevel()`
for the engager, which changes `GetCreatureBaseStats(level, unit_class)` and
therefore max health, damage and armour for **every** viewer until
`OnAllCreatureUpdate` puts it back after combat. The report is precise: the
displayed level, the health pool and the damage all move together, which is
exactly what a shared-object mutation does and exactly what the per-viewer path
avoids.

**Also worth noting (a real, separate defect on this path).**
`OnAllCreatureUpdate` (:6142) only re-evaluates when
`!creature->IsInCombat() && creature->IsAlive() && creature->GetHealth() == creature->GetMaxHealth()`,
and only every 1000 ms. So once a creature has taken a single stray hit it keeps
the level it was scaled to for the rest of the fight, and it cannot be re-scaled
by anyone else until it returns to full health out of combat. That is the
mechanism behind #5134 as well, and it is independent of the per-viewer rework.

**Recommendation.** Per-viewer scaling (or the module default) for the display and
stats; separately, decide whether the "never re-scale a creature that is not at
full health" rule is intended. It is currently unexplained in
`docs/coa/level-scaling.md`.

### #1444 - Ragefire Chasm mobs do not scale (NOT REPRODUCIBLE)

**Report.** Two players, level 22 and 15, mobs "don't scale". Core
`2abd8e291ade` (2026-09-16). The maintainer's comment: "Can't reproduce on
current code: Ragefire Chasm mobs scale up (level 14 to 37 next to a level 40
player). They take one level for everyone, from the highest player in range;
per-player levels are tracked in #1471." Classified **already working on main /
the remaining part is #1471**. Note the reporter's expectation ("mobs should
scale to our different levels") is the per-viewer feature, not a bug.

### #5134 - some enemies do not scale up (PARTLY #4296, partly the full-health rule)

**Report.** Level 26 in Wailing Caverns "with playerbots"; some enemies stay
around 19. Core `a72bd1c171ed` - **that revision is not in this clone's
history**, so it could not be dated. Two candidate causes, both real:

1. A player bot nearer than the character took the level. Fixed on this branch
   (§1) - and the reporter explicitly says "with playerbots".
2. The creature had taken damage, or was in combat, so `OnAllCreatureUpdate`
   never re-evaluated it (§3, #4382). Not fixed here.

Classification: **partly fixed**, with the remainder being the full-health rule
above. A scenario that pins cause 2 would need a mob damaged below full health
out of combat next to a higher-level player; none of the four existing scaling
scenarios covers that.

### #211 - Shadowfang Keep drops item level 15 loot (DESIGN ONLY)

**Report.** Level 60 in Shadowfang Keep; mobs scale to 57 but drop ilvl 15 loot.
No core revision given.

**Root cause.** Loot is filled from the loot template in `Unit::FillLoot`, which
uses `GetCreatureTemplate()` - the authored template - and never the scaled
level. `docs/coa/level-scaling.md:203-204` says this is deliberate: "Not
scaled, deliberately, and matching the reference implementation: **resistances**
(template-based and level-independent there too) and **loot**, which is one
corpse shared by everyone who tagged it."

**Why it is still not automatic.** The per-viewer path does not scale loot
either, and the shared-corpse argument is real: one corpse, many looters, and a
loot table frozen at the moment of death. Scaling loot per looter would need
`OnAfterLootTemplateProcess` (`src/server/game/Loot/LootMgr.cpp:563`) - the hook
exists - plus a decision about which viewer's level applies (the looter, the
tagger, the corpse's killer?) and about grouping. That is a design decision with
balance and exploit implications, not a bug fix.

**Recommendation.** If item level should follow the killer's band, the smallest
correct shape is: in `OnAfterLootTemplateProcess`, when
`LocalLevelScaling::CreatureViewMaxHealthResolver` (or a new equivalent) has a
level for the loot owner, rebase each item's `ItemLevel` from the template level
to the viewer's scaled level using the same delta the per-viewer path uses for
health. Do it only when the creature is dead, only for the loot owner, and only
for items that were not already scaled by other means (heirlooms, custom drops).
Until that decision is made, **#211 is a design issue, not a defect.**

## 4. Documentation defects found on the way (safe to fix separately)

- `docs/coa/level-scaling.md:33-34` says `CreatureMaxLift` is "0 (no ceiling) by
  default here"; `coa.conf.dist:140` ships `5`.
  `docs/coa/level-scaling-vs-main.md` repeats the claim. The `.dist` wins; the
  docs should say so and explain that `0` also switches `DesiredLevel` from
  "nearest" to "highest in sight", which is a second, undocumented meaning of the
  same knob.
- `docs/coa/level-scaling.md:203-204` documents that loot is deliberately not
  scaled but does not link it to #211, so a player filing "loot is not scaled"
  is answering a question the docs already close.
- Nothing documents that a bot is now a second-class reference player for
  scaling, or that no log line says which path owns scaling.

## 5. Test plan summary

| Change                                    | Needs                                                   | Command on the PC                                                                                                                    | Pass                                                                                                       |
| ----------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| `fix/mob-scaling` bot preference          | worldserver (realm-wide path: `DestinyWeaver.Enable=0`) | `python -B tools/verify_all.py --stages build,gameplay --scenario level-scaling-damage-engagement --scenario destiny-weaver-scaling` | all four existing scaling scenarios pass; in-game, a nearby bot does not lift a mob off a low-level player |
| per-viewer ownership log line (not done)  | worldserver                                             | -                                                                                                                                    | startup log names the owner                                                                                |
| docs vs `.dist` reconciliation (not done) | docs only                                               | `--stages source`                                                                                                                    | no behaviour change                                                                                        |
| #211 loot scaling (design only)           | worldserver + a decision                                | none until designed                                                                                                                  | -                                                                                                          |
