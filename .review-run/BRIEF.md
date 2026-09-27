REVIEW BRIEF

WHY THIS REVIEW EXISTS
The same OpenCode model that wrote these fixes also wrote a set of Ranger fixes. They compiled and passed the tests, but 6 of the 10 were broken in practice. A passing tools/verify_all.py says nothing about whether a fix actually works. Do a line-by-line CORRECTNESS review of every fix commit. For each hunk, read the diff, open the code around it, and trace the real execution path: how it is registered, when the hook fires, which objects exist and are non-null at that moment, and what data it reads.

KNOWN FAILURE PATTERNS: check every fix for each of these
- GetTarget() called in Load()/Validate() or in a check or hook where it is always null. Use GetUnitOwner(), GetCaster(), or eventInfo.GetActor()/GetActionTarget(), whichever fits.
- The damage school read from the effect's MiscValue instead of the spell's SchoolMask (GetSpellInfo()->GetSchoolMask() or damageInfo->GetSchoolMask()).
- Only rank 1 of a spell chain handled, e.g. a hard-coded first-rank id. Use sSpellMgr->GetFirstSpellInChain()/IsRankOf(), or handle every rank id, and confirm every rank is actually bound (a spell_script_names row per rank or the "-id" ranked form, or a C++ registration that covers all ranks).
- Only self-cast rival auras removed (a caster-GUID filter) when auras from other casters must go too, or the reverse.
- A level or scaling hook that runs before the owner or master is set (for pets and summons, InitStats/OnCreate runs before SetOwnerGUID/SetCreatorGUID), or that reads the level from the wrong unit.
- Also check for: a missing or mismatched registration (the script name in C++ against the SQL row, or the AddSC loader call); wrong proc flags, phase or hit masks; an effect index mismatch; a percent treated as a flat value or the reverse; spell or creature ids that don't exist in the DBC or DB; SQL without a DELETE before each INSERT, or in the wrong db folder; tests or scenarios that assert what the new code does rather than what the report expects.

LIVE-BEHAVIOUR PARITY
Check each fix against how live CoA (Ascension) behaved.
- Primary source: the Ascension DB at https://ascension-db.ascension-archive.workers.dev/. Its data is JSON at https://hertigservices.github.io/ascension-data/. Start at manifest.json to find the spell files, then query by spell id.
- Secondary source: the Discord archive at https://ascension-archive.vercel.app/.
Cite the links. If the sandbox blocks them, say so in REVIEW.md and fall back to the DBC and tooltip data in the repo.

REPO RULES: read AGENTS.md and .agents/docs/code-review.md first
- CoA C++ goes only in src/server/coa/, with NO comments.
- SQL goes only in NEW files under data/sql/updates/pending_db_*/ (the right db folder), with a DELETE before every INSERT. Never edit a SQL file that exists on main.
- Commit style: `fix(CoA/<Class>): <what now works>` with `Fixes #N` / `Refs #N` in the body. Match the reference form already used on the branch.
- Every bug fix is a NEW commit. Never amend, rebase, squash, reset or force-push. Never touch main. Don't open PRs, don't comment on or close issues, and don't touch upstream jealous-sound/azerothcore-wotlk-coa in any way.
- Make the smallest correct fix. Don't refactor or grow the scope; put any extra findings in REVIEW.md as DEFERRED.
- Verification: fetch main first (`git fetch origin main:main` if it isn't there). On every branch you change, run `python3 -B tools/verify_all.py --stages source --base main`, then `python3 apps/codestyle/codestyle-cpp.py --files <changed .cpp/.h>` and `python3 apps/codestyle/codestyle-sql.py --files <new .sql>`. You don't need to build the full server, because John's PC will build and test afterwards. Don't spend time trying to compile.
- Keep costs down: use targeted rg and ranged reads. src/server/coa/AscensionCompat.cpp is enormous, so never read it end to end. Use at most 1-2 sub-agents.

REVIEW.md
For each issue give: the number and title; a verdict of CORRECT, FIXED-IN-REVIEW (with the commit), STILL BROKEN, or DEFERRED; the reason, with file:line, data evidence or a link; and what to check in-game (GM commands such as .learn/.cast/.aura, expected numbers, combat-log lines). Finish with the list of review commits and the verification results: the exact VERIFY ALL line and the linter output.
