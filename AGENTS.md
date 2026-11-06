# Agent instructions (for coding agents other than Claude Code)

This root folder holds three independent Git repositories: this one (documentation only),
`blend_ode/` (frontend), and `blend_ode_be/` (backend). They are never merged, never submoduled —
always work in the correct repository's own working tree, and check `git status` in each one
separately before assuming anything about uncommitted state.

## Before doing anything

1. Read `agent-handoff/README.md` in full.
2. Read the files it points you to — at minimum `PROJECT_OVERVIEW.md`, `REPOSITORY_MAP.md`,
   `SYSTEM_ARCHITECTURE.md`, `INTEGRATION_CONTRACTS.md`, `CURRENT_PROJECT_STATE.md`,
   `TASK_BOARD.md`, `KNOWN_ISSUES_AND_BLOCKERS.md`, and `ARCHITECTURAL_DECISIONS.md` — before
   writing any code.
3. Inspect the actual application repositories directly. This documentation reflects what was true
   when it was last updated, not a live feed; if the code disagrees with a doc, the code is right —
   fix the doc to match, or record the discrepancy if you can't fix it immediately.
4. If your work touches the SOW editor module (`sow_module/` inside either app repo, or the
   `sow_docs/` folder at the root), also read `blend_ode_be/sow_module/CLAUDE.md` — it carries its
   own, stricter rules on top of everything here.

## Required practices throughout development

- **Maintain the handoff documentation.** Treat `agent-handoff/` as living project memory, not a
  one-time artifact. Update it as architecture, status, or decisions change — see
  `agent-handoff/DEVELOPMENT_WORKFLOW.md` for when to update which file.
- **Log every meaningful session** in `agent-handoff/SESSION_LOG.md`, using
  `agent-handoff/templates/SESSION_ENTRY_TEMPLATE.md`. Include what you actually did, actual test
  results (never claim a test passed without running it), commit hashes, and push status.
- **Keep `TASK_BOARD.md` and `CHANGELOG.md` current** as you pick up, progress, or finish work.
- **Respect `ARCHITECTURAL_DECISIONS.md`.** Don't silently contradict an Accepted decision — write a
  new entry that explicitly supersedes it if you need to change direction.
- **Work in the correct repository.** Never mix commits across the three repos. Check `git status`
  in each one you touch before committing, and never force-push, hard-reset, or rewrite history
  without explicit human authorization.
- **Never push without explicit authorization**, even if committing locally was fine to do on your
  own judgment.
- **Verify before claiming done.** Run the actual relevant tests/builds
  (`agent-handoff/TESTING_AND_VALIDATION.md`) and report real results, including failures — don't
  state something works because it plausibly should.

## Everything else

The full documentation set — architecture, integration contracts, environment variables, local
setup, known issues, and process detail — lives in `agent-handoff/`. Start at
`agent-handoff/README.md`; this file is only the entry point.
