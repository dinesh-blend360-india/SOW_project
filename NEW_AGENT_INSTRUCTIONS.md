# Instructions for the next agent working on this project

Read this file first, in full, before writing any code. It tells you what to read, how to work,
what quality bar to meet, which documents to keep updated, how to test, and what to write into
`agent2.txt` after every session so a human can understand what you did just by reading commits.

This project already has a full handoff system. This file does not replace it — it tells you how
to use it and sets the standard of care expected on top of it.

---

## 1. Before you write any code

Read these, in this exact order. Do not skip any of them, even if you think you already know the
project.

1. `agent-handoff/README.md` — the index for everything else.
2. `agent-handoff/PROJECT_OVERVIEW.md`, `REPOSITORY_MAP.md`, `SYSTEM_ARCHITECTURE.md`,
   `INTEGRATION_CONTRACTS.md`.
3. `agent-handoff/CURRENT_PROJECT_STATE.md` — what is actually done vs. in progress, right now.
   Treat this as a snapshot, not a guarantee; verify against real code before relying on it.
4. `agent-handoff/TASK_BOARD.md` — what's queued, blocked, or in progress.
5. `agent-handoff/KNOWN_ISSUES_AND_BLOCKERS.md` — don't rediscover a documented problem.
6. `agent-handoff/ARCHITECTURAL_DECISIONS.md` — don't silently contradict a recorded decision.
7. `agent-handoff/DEVELOPMENT_WORKFLOW.md` — the process rules (committing, migrations, dependency
   additions, reviewing your own diff).
8. If the task touches `sow_module/` in either app repo, or `sow_docs/`:
   `blend_ode_be/sow_module/CLAUDE.md`. It has its own stricter rules on top of everything else,
   and they win if anything conflicts.
9. `sow_docs/tracking/prompt_log.md` (newest entries at the bottom) and
   `sow_docs/build_plan/SOW_EDITOR_BUILD_PROMPTS_v1.md` — find the next unfinished checkpoint (CP-xx)
   before starting new work. Work the checkpoints in order unless told otherwise.
10. `agent2.txt` at the project root (see section 5) — the last few sessions' plain-English summary.
    Read this to understand what the previous agent actually did and what it flagged as unresolved.

Then inspect the real application code in `blend_ode/` and `blend_ode_be/` directly. Documentation
is a snapshot from when it was last written, not a live feed. If a doc disagrees with the code,
the code is right — fix the doc, or if you can't fix it in the same session, write down the drift.

Do not start coding until you can say, in your own words: which checkpoint you are doing, what it
requires, what files it touches, and what "done" looks like for it.

---

## 2. How to develop — the quality bar

This is not optional polish. Work below this bar gets reverted or redone.

- **Never guess a fact you can check.** Read the actual source file, run the actual command, or
  ask the user. Do not assume a version, a config value, a test result, or an API shape — verify it.
- **Never claim something works without having run it.** "Should work" is not a result. Run the
  typecheck, the test suite, and the build; paste the real output (pass/fail counts), not a summary
  you inferred.
- **Three repositories, never mixed.** Root (docs only), `blend_ode` (frontend), `blend_ode_be`
  (backend + Node collab service) are separate Git repos. Run `git status` in each one you touch,
  separately, before committing. Never commit frontend files from the backend's working tree or
  vice versa, and never commit either app repo's files into the root repo.
- **Follow the build plan in order.** Read the checkpoint's own text in
  `SOW_EDITOR_BUILD_PROMPTS_v1.md` before implementing it — goals, tasks, constraints, acceptance
  checks are all specified per checkpoint. Don't jump ahead or skip acceptance checks.
- **Respect existing decisions.** Don't silently drift from an `ARCHITECTURAL_DECISIONS.md` entry.
  If you must change direction, write a new entry that explicitly supersedes the old one and say why.
- **Don't touch the existing CKEditor or native-editor code** except at the swap points the build
  plan names. Never delete it before the checkpoint that explicitly says to.
- **Shared code has one source of truth.** Edit `blend_ode/src/sow_module/shared/` only, then run
  `bash blend_ode_be/sow_module/sync_shared.sh` to vendor it into the backend. Never hand-edit the
  vendored copy.
- **No secrets.** Never commit a real `.env` file, a real key, or a real token. Only `.env.example`
  files with placeholder values. Never apply a Snowflake migration without explicit user approval —
  there is one shared database.
- **Small, logical commits**, one per repo, each explaining *why* the change exists, not just what
  changed (the diff already shows what). Prefer several small commits over one large one.
- **Never push without explicit confirmation from the user**, even if you were fine to commit
  locally on your own judgment. Before pushing, tell the user exactly which repos, which branches,
  and which commits — and wait for a yes.
- **Never force-push, hard-reset, or rewrite history** without explicit authorization.
- **If a fact is missing or ambiguous, stop and ask.** Do not fill the gap with a guess and move on.
- **Read your own diff before committing.** Look specifically for: secrets, leftover debug
  `print`/`console.log` statements, and any file you didn't mean to touch (temporary demo files,
  scratch scripts, unrelated edits already sitting in the working tree from someone else).

---

## 3. The actual development loop, per checkpoint or task

1. Read the checkpoint's text in the build plan, plus anything it says to read first.
2. Inspect the real code the checkpoint touches — don't trust a doc's description of it.
3. Implement the change.
4. Write or update tests for the change. Prefer testing against real behavior (a real editor
   instance, a real HTTP client fake, a real document shape) over a mock that could drift from
   reality — several real bugs in this project were only caught by this (see `prompt_log.md`,
   e.g. the Yjs shape that the validator originally walked wrong).
5. Run the typecheck, the relevant test suite(s), and the build for every repo you touched — not
   just the one you think the change lives in. A backend contract change can silently break a
   frontend caller with no backend test failure at all.
6. Fix anything you broke. Do not edit a test just to make it pass — if a test is wrong, say so
   explicitly and fix the test for the right reason, not to clear a red mark.
7. Update the documentation this session's change makes stale (see section 4) — in the same
   session, not "later."
8. Commit, with a message that explains why. Do not push yet.
9. Append the session's summary to `agent2.txt` (see section 5).
10. Only after all of the above: tell the user what you'd push and where, and wait for confirmation.

---

## 4. Documentation and tracker updates — required every session that changes anything

Do these in the same session as the change, not retroactively:

- **`sow_docs/tracking/prompt_log.md`** — append one entry per checkpoint, in the exact format
  given in `blend_ode_be/sow_module/CLAUDE.md` (`## <CP-ID>: <title> (<date>)`, Summary,
  Decisions/assumptions, Registry of new/changed files/functions/endpoints, Open questions,
  Handoff to next prompt). Append only — never overwrite a previous entry.
- **`agent-handoff/SESSION_LOG.md`** — one entry per meaningful session, newest on top, using
  `agent-handoff/templates/SESSION_ENTRY_TEMPLATE.md`. Include real commit hashes and real test
  results, repo by repo.
- **`agent-handoff/CHANGELOG.md`** — append what was built/fixed, grouped by date and repo, with
  commit hashes.
- **`agent-handoff/TASK_BOARD.md`** — move a task's status as it actually changes (`NOT_STARTED` →
  `IN_PROGRESS` → `BLOCKED`/`COMPLETED`), and add anything newly discovered.
- **`agent-handoff/KNOWN_ISSUES_AND_BLOCKERS.md`** — add anything you find that isn't fixed in the
  same session; mark anything you did fix as resolved.
- **`agent-handoff/CURRENT_PROJECT_STATE.md`** — update the status snapshot if a checkpoint range
  completed, or anything you were told is "done" is no longer accurate.
- **`agent-handoff/ARCHITECTURAL_DECISIONS.md`** — append (never edit in place) a new entry if you
  made a decision for this project that doesn't already have one.
- **`sow_docs/tracking/core_api_freeze.md`** — if your change touches a route, WebSocket message,
  component prop, or close code listed there, add a dated "Change log" line describing exactly
  what changed and whether it's breaking.
- **Fix any other doc your change makes inaccurate** (`SYSTEM_ARCHITECTURE.md`,
  `INTEGRATION_CONTRACTS.md`, `REPOSITORY_MAP.md`, `ENVIRONMENT_AND_SECRETS.md`,
  `LOCAL_DEVELOPMENT_SETUP.md`, `TESTING_AND_VALIDATION.md`) in the same session.

---

## 5. Testing — what to actually run, and what to report

Run these for whatever you touched (and the ones that cross a boundary, even if you think you
didn't touch the other side):

```bash
# Frontend (blend_ode)
npx tsc --noEmit
CI=true npm test -- --watchAll=false --testPathPattern=sow_module
npm run build

# Backend (blend_ode_be)
python -m pytest tests -q

# Node collab service (blend_ode_be/sow_module/collab)
npx tsc --noEmit
npx eslint src
npx vitest run

# Combined check across both repos, for anything under sow_module/
bash blend_ode_be/sow_module/run_checks.sh
```

Rules:

- **Paste the real summary line** ("175 passed", "9 failed, 556 passed"), never a paraphrase.
- **Know the pre-existing failure baseline before you start** (see
  `agent-handoff/KNOWN_ISSUES_AND_BLOCKERS.md` #9 and `phaseD_report.md`), so you can tell "still 9
  failing, unrelated" apart from "I introduced a new failure."
- If a suite fails and you believe the failure is pre-existing, prove it: check it against the
  documented baseline, or re-run the same suite on the commit before your change.
- **List every test file you added or changed**, and what each one actually asserts, in your
  session summary — not just a pass count.
- Don't claim a UI change works from reading the code alone. Where you can, drive it in a real
  browser (or headless Chrome) and say what you actually saw, including anything you couldn't
  verify this way.

---

## 6. `agent2.txt` — write this after every session, without exception

Create (or append to) a single plain-text file named `agent2.txt` at the project root
(`C:\workspace\SOW_project\agent2.txt`). This is **not** a replacement for the handoff docs in
section 4 — it's a short, human-readable summary so the user can open it after your commits and
understand what happened in under a minute, without reading any diff.

Append a new block each session — never delete or rewrite a previous block. Use this exact shape:

```
===========================================================
SESSION: <YYYY-MM-DD HH:MM> — <one-line description of what you worked on>
===========================================================

WHAT I DID
- <plain-English bullet, no jargon, one per meaningful change>
- <...>

CHECKPOINTS / TASKS
- <CP-xx or task name>: <done / partly done / blocked> — <one line why>

TESTS I RAN AND THE REAL RESULT
- <command> -> <real pass/fail line>
- <command> -> <real pass/fail line>
(list every suite you ran, not just the ones that passed)

FILES CHANGED (by repo)
- blend_ode: <short list or "none">
- blend_ode_be: <short list or "none">
- root: <short list or "none">

DOCS / TRACKERS UPDATED
- <file>: <what you changed in it>
(list every tracker file from section 4 that you touched this session)

COMMITS (not pushed unless stated)
- <repo>: <short hash> — <one-line message>

PUSH STATUS
- <"not pushed, awaiting confirmation" / "pushed to origin/<branch> after user confirmed">

STILL BROKEN / NOT DONE / NEEDS A DECISION
- <anything left unresolved, in plain English, so the next person doesn't have to rediscover it>

WHAT TO DO NEXT
- <the next concrete step>
```

Keep the language plain — write it so a non-technical person skimming the commit history could
still follow what happened. Save the technical detail for `prompt_log.md` and `SESSION_LOG.md`;
`agent2.txt` is the short version that sits on top of those.

---

## 7. Before you ever push

1. Run every test suite relevant to what changed, for every repo touched, and have the real
   results in hand.
2. Confirm `agent2.txt` is updated for this session.
3. Confirm every tracker in section 4 is updated for this session.
4. Tell the user, explicitly: which repos, which branches, which commits (hashes + one-line
   messages), and a one-line summary of what's in them. Do not push until they say yes.
5. Never include `.env` files, real secrets, or temporary/demo-only files in what you push.
