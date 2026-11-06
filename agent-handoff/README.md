# Agent Handoff System

This is the starting point for any AI agent (or person) continuing development on this project.
It exists so that development context, decisions, and status survive the end of any one session —
you should be able to clone this root repository, place the two application repositories inside it,
open an agent, and continue work with a real understanding of the project instead of re-deriving
everything from scratch.

## What this is

A structured set of Markdown files, each covering one concern, split into two kinds:

- **Stable reference** — describes things that don't change every session: the project's purpose,
  the repository layout, the architecture, the integration contracts, how to set up a local
  environment, and the accumulated decision log.
- **Living documents** — updated every meaningful session: the current state, the task board, known
  issues, the changelog, and the session log.

Mixing these into one file (or worse, one giant README) is exactly what this system avoids —
stable docs would get stale clutter from constant status updates, and living docs would get buried
in architecture explanation that doesn't need to be read every time.

## Repositories involved

Three independent Git repositories, never merged, never submoduled:

| Repo | Role |
|---|---|
| This one (root) | Documentation and the agent-handoff system only. No application code. |
| `blend_ode` | Frontend (React, Create React App). |
| `blend_ode_be` | Backend (FastAPI, Snowflake). |

See `REPOSITORY_MAP.md` for what's inside each, and `LOCAL_DEVELOPMENT_SETUP.md` for exactly how to
clone and place all three.

## Reading order before starting any development work

1. **This file.**
2. `PROJECT_OVERVIEW.md` — what the project is, who it's for, what's in scope.
3. `REPOSITORY_MAP.md` — where things live.
4. `SYSTEM_ARCHITECTURE.md` and `INTEGRATION_CONTRACTS.md` — how the pieces actually talk to each
   other, verified from source.
5. `CURRENT_PROJECT_STATE.md` — what's actually done vs. in progress vs. not started, right now.
6. `TASK_BOARD.md` — what's queued, and what you might pick up.
7. `KNOWN_ISSUES_AND_BLOCKERS.md` — don't rediscover a documented problem.
8. `ARCHITECTURAL_DECISIONS.md` — don't silently contradict a decision someone already made and
   recorded a reason for.

Before writing any code, also read `DEVELOPMENT_WORKFLOW.md` for the actual process rules, and
`LOCAL_DEVELOPMENT_SETUP.md` + `ENVIRONMENT_AND_SECRETS.md` to get a working environment.

**Then inspect the real application repositories.** This documentation describes verified facts as
of when it was written, not a live feed — code changes faster than docs get updated. If something
here disagrees with what you read in `blend_ode`/`blend_ode_be`, trust the code, and fix the
documentation (or note the drift in `SESSION_LOG.md` if you can't fix it in the same session).

## Full file inventory

| File | Purpose |
|---|---|
| `README.md` | This file |
| `PROJECT_OVERVIEW.md` | Objective, problem, solution, scope, phase |
| `REPOSITORY_MAP.md` | What's in each repo, where to implement what |
| `SYSTEM_ARCHITECTURE.md` | Components, request lifecycle, data flow, diagrams |
| `INTEGRATION_CONTRACTS.md` | Every API contract, auth flow, shared data model |
| `LOCAL_DEVELOPMENT_SETUP.md` | Clone, install, configure, run, verify |
| `ENVIRONMENT_AND_SECRETS.md` | Every env var, what it's for, and a known secret-exposure warning |
| `DEVELOPMENT_WORKFLOW.md` | Process rules for planning, implementing, reviewing, committing |
| `TESTING_AND_VALIDATION.md` | How to run every test suite, and the last real results |
| `CURRENT_PROJECT_STATE.md` | What's done, in progress, not started — kept current |
| `TASK_BOARD.md` | Status-tracked task list |
| `KNOWN_ISSUES_AND_BLOCKERS.md` | Bugs, gaps, limitations, with reproduction/impact |
| `ARCHITECTURAL_DECISIONS.md` | The project-wide decision log (indexes the SOW module's own) |
| `CHANGELOG.md` | Human-readable history by date, with commit hashes |
| `SESSION_LOG.md` | Chronological record of every development session |
| `templates/` | Copy-paste templates for session/task/ADR entries |

## How to update project memory

- **Every meaningful session** (anything that reads, writes, or decides something another session
  needs to know) gets a `SESSION_LOG.md` entry, using `templates/SESSION_ENTRY_TEMPLATE.md`. A pure
  Q&A session with no file changes doesn't need one.
- **Every completed or status-changed task** gets reflected in `TASK_BOARD.md`.
- **Every shipped change** gets a line in `CHANGELOG.md`, grouped by date.
- **Every significant technical decision** — something another session shouldn't casually reverse
  without knowing why it was made — gets an `ARCHITECTURAL_DECISIONS.md` entry via
  `templates/ADR_TEMPLATE.md`. Don't silently change an Accepted decision; supersede it explicitly.
- **Every newly-found bug, gap, or limitation** goes in `KNOWN_ISSUES_AND_BLOCKERS.md` as you find
  it, not after the fact.
- If something you read here turns out to be wrong or stale, fix it in the same session if you can.

## How to continue work from another agent's session

1. Read `SESSION_LOG.md`'s most recent entry (or entries, if several happened since you last
   looked) — it has starting state, what changed, what's unresolved, and recommended next actions.
2. Cross-check against `TASK_BOARD.md` for anything marked `IN_PROGRESS` or `BLOCKED`.
3. Run `git status` in **each** of the three repositories separately before assuming a clean
   starting point — uncommitted work from a prior session may still be sitting in the working tree.
4. Don't assume a previous session's claimed test results still hold — re-run them
   (`TESTING_AND_VALIDATION.md` has the commands) before relying on a "passed" status that might now
   be stale.

## A note on verification

Everything in this system distinguishes, as explicitly as possible:
- **Verified** — read directly from source, a command's actual output, or `git log`/`git status`.
- **Reasonable inference** — stated as such, with the reasoning given.
- **Unknown** — marked "UNKNOWN — needs verification" rather than guessed at.

If you add to this system, keep that discipline. A wrong but confident-sounding doc is worse than
an honest gap.
