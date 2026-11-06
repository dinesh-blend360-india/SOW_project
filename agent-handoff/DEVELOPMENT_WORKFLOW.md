# Development Workflow

This is process, not architecture — read `SYSTEM_ARCHITECTURE.md`/`REPOSITORY_MAP.md` for what the
system looks like, and `README.md` in this folder for how the handoff system itself is organized.

## Before touching an existing feature

1. Find the owning repo and area via `REPOSITORY_MAP.md`.
2. Read the actual source — the file(s) implementing the feature, not just this documentation.
   Documentation describes intent at a point in time; code is current truth. If they disagree,
   trust the code and flag the doc as stale (fix it, or note the drift in `SESSION_LOG.md`).
3. Check `KNOWN_ISSUES_AND_BLOCKERS.md` and `TASK_BOARD.md` for anything already known about that
   area — don't rediscover a documented limitation from scratch.
4. If the feature is part of the SOW editor module (`sow_module/` in either repo), read
   `blend_ode_be/sow_module/CLAUDE.md` first — it has its own, stricter standing rules that apply
   on top of everything here.

## Planning an implementation

1. Identify every repository the change touches (root docs, `blend_ode`, `blend_ode_be`, or more
   than one). A change that spans frontend and backend needs both sides planned together, even if
   implemented and committed separately.
2. Check `ARCHITECTURAL_DECISIONS.md` for any decision the plan would need to go against. Preserve
   existing decisions unless you have a documented reason to change them (write a new ADR that
   supersedes the old one — don't silently drift from it).
3. For anything non-trivial, add or update a task in `TASK_BOARD.md` before starting, so the next
   session (or person) can see work in progress rather than finding an unexplained half-done
   change.
4. Where practical, run the relevant tests/build **before** making changes, so you have a real
   baseline to compare against afterward (see `TESTING_AND_VALIDATION.md` — this project has 9
   known pre-existing backend test failures; know that number before you start, so you don't
   mistake "still 9" for "I introduced one" or vice versa).

## Maintaining compatibility

- Don't change an existing endpoint's request/response shape without checking every caller
  (`INTEGRATION_CONTRACTS.md` lists the endpoints; grep both repos for the exact path).
- Don't rename or remove a Python function/class that another module imports directly — several
  cross-module calls in this codebase are direct function reuse, not interfaces (e.g. the SOW
  module's internal endpoints call `assert_deal_access`/`save_sow_draft`/`get_role_and_features`
  directly; breaking those signatures breaks the SOW module too).
- The SOW module's own constraint applies project-wide in spirit even outside that module: prefer
  reusing an existing function over copying its logic.

## Introducing a new dependency

- Check it isn't already available transitively before adding it directly (this bit the SOW
  module once — `generic-pool` was available transitively via `snowflake-sdk` but had to be added
  explicitly once imported directly).
- Pin a specific version, not a floating range, for anything security-sensitive (the project's
  existing `snowflake-sdk` pin to `^3.4.1` was specifically chosen to fix a disclosed advisory —
  follow that precedent for other packages with known CVEs).
- Record the addition and why in `CHANGELOG.md`.

## Database migrations

- **Never apply a migration to Snowflake without explicit approval from whoever owns the
  database.** This is a hard rule, not a style preference — there is one shared Snowflake instance
  across the whole project.
- Write the migration file, write a guard test if the area warrants one (see the SOW module's
  `test_sow_module_migration_011.py` for the pattern: parse the SQL text itself and assert it only
  touches tables it's supposed to, never an existing application table), and present both for
  approval before running anything live.
- Number migrations sequentially within whichever migrations folder the change belongs to
  (`migrations/` is the active one; see `KNOWN_ISSUES_AND_BLOCKERS.md` for the unresolved question
  about `db/migrations/`).

## Reviewing changes

- Re-run the test suite(s) for every repo you touched, not just the one you think the change lives
  in (a backend contract change can silently break a frontend caller with no backend test failure
  at all).
- For the SOW module specifically, `bash blend_ode_be/sow_module/run_checks.sh` runs everything
  relevant across both repos in one command.
- Read your own diff once before committing, specifically looking for: secrets, debug `print`/
  `console.log` left in, and any file you didn't mean to touch.

## Maintaining documentation

- If a change makes something in `SYSTEM_ARCHITECTURE.md`, `INTEGRATION_CONTRACTS.md`,
  `REPOSITORY_MAP.md`, or `ENVIRONMENT_AND_SECRETS.md` inaccurate, update that file in the same
  session — don't leave it for "later." Later sessions have no way to know it's stale.
- Append to `CHANGELOG.md` and `SESSION_LOG.md` every session that changes anything (see their own
  files for exactly what to record, and `templates/` for the entry format).
- Update `TASK_BOARD.md` (move a task's status, or add a newly-discovered one) and
  `KNOWN_ISSUES_AND_BLOCKERS.md` (add anything newly found) as you go, not retroactively.
- `ARCHITECTURAL_DECISIONS.md` is append-and-supersede, never silently edited in place for a
  decision already marked Accepted.

## Committing changes

- Each of the three repositories (root, `blend_ode`, `blend_ode_be`) is committed to **separately**
  — never commit frontend changes from inside the backend repo's working tree or vice versa. Run
  `git status` in each repo you touched before committing anything, so nothing unexpected gets
  swept in.
- Prefer small, logical commits over one large one — easier to review, easier to revert.
- Write commit messages that explain *why*, not just *what* — the diff already shows what changed.
- Record the repository, branch, resulting commit hash, and a one-line summary in `SESSION_LOG.md`.
- Do not push without explicit human authorization, even if committing was fine to do
  autonomously. Report push status accurately — "committed, not pushed" is a normal, expected state
  to report, not something to apologize for or rush to fix unasked.

## Handing over unfinished work

- Commit what's safely committable, even if incomplete — don't leave only uncommitted working-tree
  changes as the sole record of partial progress.
- Write a `SESSION_LOG.md` entry regardless of whether the session finished its objective. Include
  unresolved issues and recommended next actions explicitly — a cold reader (another session, or a
  person) should be able to pick up from that entry alone without re-deriving what you were doing.
- If you leave a task in `TASK_BOARD.md` as `IN_PROGRESS` or `BLOCKED`, say specifically what's
  blocking it or what the next concrete step is — not just that it's unfinished.
- Never silently abandon a decision you were part-way through reversing — if you started changing
  an Accepted ADR's approach but didn't finish, say so explicitly rather than leaving the code in a
  state that matches neither the old nor the new decision.
