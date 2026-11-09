# Task Board

Statuses: `NOT_STARTED`, `IN_PROGRESS`, `BLOCKED`, `IN_REVIEW`, `COMPLETED`. Only tasks grounded in
the module's own build plan, a documented open question, or a verified gap found during this
inventory are listed — nothing here is invented scope. Use `templates/TASK_ENTRY_TEMPLATE.md` for
new entries.

---

## IN_PROGRESS

### SOW-CP21: Node Express skeleton, ticket verification, CI-style checks
- **Priority**: High
- **Affected repository**: blend_ode_be (`sow_module/collab/`)
- **Dependencies**: CP-18/19/20 (complete)
- **Relevant files**: `blend_ode_be/sow_module/collab/src/`, `sow_docs/build_plan/SOW_EDITOR_BUILD_PROMPTS_v1.md` (CP-21 section)
- **Description**: Build the actual Express + `ws` server that hosts live WebSocket connections,
  verifies the tickets FastAPI's session endpoint mints, and calls the internal endpoints built in
  CP-20. Per the module's own build plan — read that prompt in full before starting.
- **Acceptance criteria**: per the build plan's own CP-21 acceptance checks (not restated here to
  avoid drift between two copies — read the source).
- **Notes**: Next checkpoint in the module's sequential build plan; nothing else should jump ahead
  of it in that module.

---

## NOT_STARTED

### SOW-CP25: Two-browser live-editing proof
- **Priority**: High (first real end-to-end UI milestone for the module)
- **Affected repository**: both
- **Dependencies**: CP-21 through CP-24 (not started)
- **Relevant files**: `sow_docs/build_plan/SOW_EDITOR_BUILD_PROMPTS_v1.md`
- **Description**: The module's build plan's own first real UI milestone — two browser sessions
  editing the same SOW live. Requires the browser-side Lexical editor and WebSocket client to exist
  first.
- **Acceptance criteria**: per the build plan.

### SOW-ROLE: Resolve decisions.md Q2 (who is sow_admin / finalize-capable)
- **Priority**: Medium
- **Affected repository**: blend_ode_be
- **Dependencies**: none technical — needs a product decision from the project owner
- **Relevant files**: `blend_ode_be/src/sow_module/collab_session.py` (`resolve_role`),
  `sow_docs/tracking/decisions.md` (Q2)
- **Description**: The code currently maps `FE_ADMIN` -> `sow_admin`, everyone else who passes
  `assert_deal_access` -> `editor`. This was built as an explicit, documented stand-in, not a
  resolution of the open question. Get a real answer, then update `resolve_role` and its tests.
- **Acceptance criteria**: `decisions.md` Q2 has a recorded user answer; `resolve_role` matches it;
  `tests/test_sow_module_collab_session.py` and `test_sow_module_collab_internal.py` updated to
  match the new rule if it differs from today's.

### SOW-FENCE: Independent fencing-token re-verification at checkpoint time
- **Priority**: Low (documented, deliberate gap, not urgent while the feature isn't live)
- **Affected repository**: blend_ode_be
- **Dependencies**: a second, narrowly-scoped Snowflake connection into `BLEND_INTERNAL_IT_APPS.NEXDRAFT` from FastAPI
- **Relevant files**: `src/sow_module/collab_internal.py` (`checkpoint`)
- **Description**: See `KNOWN_ISSUES_AND_BLOCKERS.md` #7 for the full context.
- **Acceptance criteria**: `checkpoint` rejects a request whose `fencingToken` doesn't match the
  live `SOW_COLLAB_LEASE` row for that deal, with a test proving it.

### INFRA-SECRET-ROTATE: Rotate and scrub secrets committed in `blend_ode`
- **Priority**: High (security)
- **Affected repository**: blend_ode
- **Dependencies**: coordination with whoever owns each credential (DB, Agents API key, CKEditor
  license, Azure AD app registration)
- **Relevant files**: `.env.development`, `.env.production`, `src/config/MsalConfig.ts`
- **Description**: See `KNOWN_ISSUES_AND_BLOCKERS.md` #1. Rotate every exposed credential; decide
  whether history-scrubbing is required by policy; create a real `.env.example` afterward
  (see INFRA-ENV-EXAMPLE below).
- **Acceptance criteria**: all four secret categories rotated; `.env.development`/`.env.production`
  no longer contain values that work against production systems; a decision recorded (in
  `ARCHITECTURAL_DECISIONS.md` or a new note) on whether history was scrubbed and why/why not.

### INFRA-ENV-EXAMPLE: Create `.env.example` for `blend_ode`
- **Priority**: Medium
- **Affected repository**: blend_ode
- **Dependencies**: INFRA-SECRET-ROTATE (do this after rotation, not before, to avoid creating an
  example from already-compromised values)
- **Relevant files**: new file `blend_ode/.env.example`
- **Description**: See `KNOWN_ISSUES_AND_BLOCKERS.md` #3. `ENVIRONMENT_AND_SECRETS.md` in this
  handoff system already lists every variable name and purpose as source material.
- **Acceptance criteria**: `.env.example` exists with every variable from
  `ENVIRONMENT_AND_SECRETS.md`'s frontend table, placeholder values only.

### INFRA-CI: Add an automated test-running CI pipeline to both repos
- **Priority**: Medium
- **Affected repository**: both
- **Dependencies**: deciding how to handle the 9 pre-existing backend test failures (xfail them or
  fix them first, so the pipeline isn't permanently red)
- **Relevant files**: new `.github/workflows/*.yml` in each repo
- **Description**: See `KNOWN_ISSUES_AND_BLOCKERS.md` #2.
- **Acceptance criteria**: a PR-triggered workflow in each repo runs the commands in
  `TESTING_AND_VALIDATION.md` and fails the PR on an unexpected test failure.

### INFRA-MIGRATIONS-CLARIFY: Resolve the `migrations/` vs `db/migrations/` question
- **Priority**: Low
- **Affected repository**: blend_ode_be
- **Dependencies**: an answer from the project owner
- **Relevant files**: `migrations/`, `db/migrations/`
- **Description**: See `KNOWN_ISSUES_AND_BLOCKERS.md` #5.
- **Acceptance criteria**: this handoff system's docs updated once the answer is known; if
  `db/migrations/` is confirmed dead, it's either removed or clearly marked archival.

### INFRA-DEPS-CLARIFY: Resolve `Pipfile` vs `requirements.txt` as the backend's authoritative dependency source
- **Priority**: Low
- **Affected repository**: blend_ode_be
- **Dependencies**: none
- **Relevant files**: `Pipfile`, `requirements.txt`, `README.md`
- **Description**: See `KNOWN_ISSUES_AND_BLOCKERS.md` #6.
- **Acceptance criteria**: one is picked as authoritative; the other is either regenerated from it
  or removed; README updated to match.

---

## BLOCKED

*(nothing currently blocked as of this writing — SOW-CP25 above is listed NOT_STARTED rather than
BLOCKED because its dependency, CP-21, is itself in progress, not stalled)*

---

## IN_REVIEW

*(none)*

---

## COMPLETED

### SOW-CP00-CP20: SOW collaboration module build-out through CP-20
- **Priority**: —
- **Affected repository**: both
- **Description**: See `CHANGELOG.md` for the itemized history and `CURRENT_PROJECT_STATE.md` for
  what's actually built. All 20 checkpoints' acceptance checks passed at the time each was
  completed; CP-16/17's Snowflake migrations are written and tested but **not yet applied** — that
  application step is a separate, explicitly-gated action, not part of "complete."

### DOCS-HANDOFF: Create this agent-handoff documentation system
- **Priority**: —
- **Affected repository**: root
- **Description**: This task. See `SESSION_LOG.md` for the session that created it.

## Update 2026-11-09
- **SOW-CP21 to SOW-CP25: COMPLETED** (see CHANGELOG).
- **NEW — DEMO-ACCESS:** get the demo user `FE_SOW_LEXICAL_EDITOR` (or `FE_ADMIN`) through the admin screens, or use a deal the user already owns. Status: BLOCKED on an admin.
- **NEW — BACKEND-500-SYNC:** `/pricing_tool/hubspot_deals_sync_info` returns 500. Needs the backend traceback before a fix. Status: BLOCKED on logs.
- **DONE — PRICING-START-DATE-CRASH:** fixed in 8853099.
- **Next:** SOW-CP26 (toolbar).
