# fix(CoA/Scaling): a nearby bot no longer takes a creature's level from a player

## Summary

On a realm running the realm-wide level-scaling path, a mob's level is decided
by the _nearest_ character in its sight range. Bots were in that set, so a level
22 bot walking into Sunstrider Isle lifted the Mana Wyrms a level 2 character was
fighting up to its own band, and the level came back only when the bot left.

Scaling now prefers the nearest session that is not a bot, and falls back to the
nearest bot only when no player is in range, so a bot-only area behaves exactly as
it does today. A bot's pull no longer sets a creature's level either.

## Root cause and fix

**#4296 - Mobs scaling off high-level bots.** Level 2 in Sunstrider Isle, mobs
were the character's level until a level 22 bot entered the area, at which point
they jumped to about 19. Core `7b66e4380261` (2026-09-18).

`DesiredLevel` in `src/server/coa/AscensionCompat.cpp:6188` walked
`map->GetPlayers()` filtering for alive, not a GM, same phase, within
`creature->GetSightRange()`, and a valid attack target - and nothing else. With
`useNearestPlayer` (which is `CreatureMaxLift != 0`, true at the shipped
`CoA.LevelScalingMaxLift = 5`) the closest qualifying session won, so proximity,
not ownership, decided. `ScaleForEngager` (:6247) had the same gap: a bot's first
hit or pull wrote the pending engager level, which `DesiredLevel` then preferred
over everyone in sight.

The fix runs the existing scan twice, first for non-bot sessions and then for
bot sessions, breaking as soon as one pass found a candidate, and returns early
from `ScaleForEngager` for a bot-owned engager. `desired` is seeded with the
creature's authored level and the "highest in sight" mode only ever raises it, so
running the bot pass second cannot lower a level a player already set.

**Why bots are a second-class reference here.** `docs/coa/level-scaling.md` states
the rule as being about characters: "The **nearest** character decides the level
either way, never the highest level in sight: taking the maximum hands one
player's level to everybody." The `CoA.LevelScalingMaxLift` comment in
`src/server/coa/conf/coa.conf.dist:123-137` names bots as the case the cap exists
for ("the low level bots fighting them are killed by creatures that were never
theirs"), and `IsBot()` is already the exclusion convention in the same file for
other class features (`AscensionCompat.cpp:663,3274,3441`,
`AscensionHighRisk.cpp:335,528`). GM exclusion is unchanged.

**Scope note, and the reason four of the six G6 issues are not in this PR.** CoA
has two level-scaling implementations and only one runs at a time:
`LocalLevelScaling::CreatureScalingOwnedPerViewer` makes `CanScaleCreature`
(`AscensionCompat.cpp:6111`) return false, and `mod-destiny-weaver` sets it when
`DestinyWeaver.Enable && DestinyWeaver.LevelScaling`
(`modules/mod-destiny-weaver/src/destiny_weaver_scaling.cpp:882`). The shipped
`.dist` values are `1` for both, so **with the default configuration the per-viewer
path owns scaling and this commit's code path is inert.** Every "one level for the
whole group" symptom can therefore only occur on a realm that turned the module
off, and the reporter cannot see which mode their realm runs. #1471, #4382 and
#5134 are that problem and need the per-viewer rework #1471 already tracks; #211
(loot item level) is a design decision. All of it is written up in
`~/src/groups-run/mob-scaling-design.md`, which should travel with this PR.

## Tests to run on the PC

```sh
python -B tools/verify_all.py --stages build,gameplay --scenario level-scaling-damage-engagement
python -B tools/verify_all.py --stages build,gameplay --scenario destiny-weaver-scaling
python -B tools/verify_all.py --stages build,gameplay --scenario skinning-dungeon-scaled-view
python -B tools/verify_all.py --stages build,gameplay --scenario skinning-open-world-level-scaling
python -B tools/verify_all.py --base origin/main
python apps/codestyle/codestyle-cpp.py --files src/server/coa/AscensionCompat.cpp
```

All four existing scaling scenarios use real sessions (none sets `"bot": true`;
only two scenarios in the whole catalogue do, and neither is a scaling scenario),
so they must be unaffected in both module modes - that is the regression guard.
`level-scaling-damage-engagement` is a known baseline failure, so compare against
its recorded result rather than expecting a clean pass.

In game (the fail-before/pass-after check, which no scenario covers):

1. Set `DestinyWeaver.Enable = 0` so the realm-wide path owns scaling; keep
   `CoA.LevelScaling = 1` and `CoA.LevelScalingMaxLift = 5`.
2. Make a level 2 character in Sunstrider Isle. Note a Mana Wyrm's nameplate
   level.
3. Log in a level 22 player bot (or use a second account with a bot session)
   within 20 yd of the Wyrm.
4. Pass: the Wyrm's nameplate level does not rise. With the change reverted it
   rises to roughly 19 and drops back when the bot logs out - which is the report.
5. Negative control: with the level 2 character still in the area and the bot
   further away than the character, the level follows the character. Then move
   the character out of sight and confirm the mob may follow the bot - that is
   the documented fallback, not a regression.
6. A realm with no bots must behave identically before and after; the four
   scenarios above are that check.

## Risks

- **A bot-only area still scales, to the bot.** Deliberate: a realm that farms
  with bots keeps the behaviour it has. The alternative (never scale for a bot)
  would silently drop scaling wherever no player is present.
- **A bot that pulls from outside the creature's sight range no longer sets the
  level.** It keeps its authored level instead. Bots normally fight what they can
  see, where the fallback pass still applies, but a scripted bot pulling through a
  pet from long range would see a difference.
- **The per-viewer path is untouched.** If a realm's reports are actually coming
  from the per-viewer path, this commit changes nothing for them and the real
  answer is in the design document.
- The two-pass loop evaluates `IsBot()` per player per creature per second, as
  before; the cost is one extra pass over `map->GetPlayers()` when no player is in
  range, which is the cheap case.
- No scenario covers a bot as the scaling reference, so the fail-before/pass-after
  evidence is the in-game repro above. Writing one needs a scenario with
  `"bot": true` players on both sides, which does not exist yet.

## Not addressed

- #1471, #4382: one level for the whole group because the realm-wide path mutates
  the shared creature. Needs the per-viewer rework #1471 tracks.
- #1444: could not be reproduced on current code (maintainer comment on the
  issue); its expectation is the per-viewer feature.
- #5134: partly this commit, partly the rule that a creature is only re-evaluated
  while it is out of combat **and** at full health
  (`AscensionCompat.cpp:6142-6146`), which is documented nowhere.
- #211: loot item level is not scaled on either path, by design
  (`docs/coa/level-scaling.md:203-204`).
- A log line naming which path owns scaling, and the docs/`.dist` contradiction
  over `CoA.LevelScalingMaxLift` (docs say the default is 0, the `.dist` ships 5).
  Both are in the design document; neither is a behaviour change.

Fixes #4296
