# Working in this workspace

This root folder holds three independent Git repositories: this one (docs only), `blend_ode/`
(frontend), and `blend_ode_be/` (backend). They are never merged, never submoduled — always work in
the correct repository's own working tree, and check `git status` in each one separately before
assuming anything about uncommitted state.

## Before doing anything

1. Read `agent-handoff/README.md`.
2. Read every file it points you to before beginning implementation — at minimum
   `PROJECT_OVERVIEW.md`, `REPOSITORY_MAP.md`, `SYSTEM_ARCHITECTURE.md`, `INTEGRATION_CONTRACTS.md`,
   `CURRENT_PROJECT_STATE.md`, `TASK_BOARD.md`, `KNOWN_ISSUES_AND_BLOCKERS.md`, and
   `ARCHITECTURAL_DECISIONS.md`.
3. Inspect the actual application repositories (`blend_ode/`, `blend_ode_be/`) directly — the
   documentation describes verified facts as of when it was written, not a live feed. If source
   code disagrees with a doc, trust the code, and fix the doc (or note the drift if you can't).
4. If the work touches the SOW editor module specifically (`sow_module/` in either app repo, or
   `sow_docs/`), also read `blend_ode_be/sow_module/CLAUDE.md` — it has its own, stricter standing
   rules layered on top of everything here.

## While working

- Maintain the handoff documentation as you go — don't defer it to "later." See
  `agent-handoff/DEVELOPMENT_WORKFLOW.md` for exactly when to update what.
- Record every meaningful session in `agent-handoff/SESSION_LOG.md`.
- Update `agent-handoff/TASK_BOARD.md` and `agent-handoff/CHANGELOG.md` as status actually changes,
  not retroactively at the end.
- Preserve existing technical decisions (`agent-handoff/ARCHITECTURAL_DECISIONS.md`) unless you have
  a documented reason to change one — write a new entry that supersedes the old one, don't silently
  drift from it.
- Work inside the correct repository. Never commit `blend_ode` changes from `blend_ode_be`'s
  working tree or vice versa, and never commit either app repo's files into this root repo (both
  are git-ignored here on purpose).
- Verify changes through the actual tests/builds for whatever you touched
  (`agent-handoff/TESTING_AND_VALIDATION.md`) before calling anything done. Never state something is
  complete or tests passed without having actually run them.

## Full detail

Everything else — process rules, architecture, environment setup, known issues — lives in
`agent-handoff/`. This file stays short on purpose; it is the entry point, not the documentation
itself.
