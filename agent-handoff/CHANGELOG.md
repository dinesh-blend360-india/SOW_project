# Changelog

Human-readable history of completed development work, organized by date. Verified against actual
commit history in each repository (`git log`) — entries below are not reconstructed from memory.
This changelog starts from the first commit of the SOW editor module initiative (2026-10-30); any
earlier project history exists in each repo's own `git log` but is not re-transcribed here.

When adding an entry: use the real commit hash from `git log`, not a guess. Group by date, then by
repository.

---

## 2026-11-06

### blend_ode_be
- `feat(sow_module)`: FastAPI internal endpoints for the Node collab service — `resolve-access`,
  `document-state`, `checkpoint`, all service-key protected and hidden from the public OpenAPI
  schema; `notify_access_changed` wired into guest revocation. CP-20. (`408b0f8`)
- `feat(sow_module)`: FastAPI collab session endpoint (`POST /pricing_tool/sows/{sow_id}/collab/session`),
  30-second signed ticket, `FE_SOW_LEXICAL_EDITOR` feature flag, Python role table mirroring the
  frontend's. CP-19. (`dc0218b`)
- `feat(sow_module)`: Snowflake data layer for the Node collab service (`SnowflakeDataStore`),
  live-tested against real Snowflake; found and fixed a binary-bind bug (`TO_BINARY(?, 'HEX')`
  instead of a raw Buffer through a MERGE subquery). CP-18. (`427f479`)
- `docs(sow_module)`: finalized the standard-tables (not hybrid-tables) decision for
  `SOW_COLLAB_*`, and the grants migration file. CP-17. (`a1b55cb`)
- `docs(sow_module)`: wrote the `SOW_COLLAB_*` tables migration (not yet applied to Snowflake).
  CP-16. (`65fe343`)
- `docs(sow_module)`: SOW_COLLAB tables migration, written not applied. CP-16. (`65fe343`)
- `spike(sow_module)`: 100-client WebSocket broadcast scale spike and Phase B report. CP-15. (`fc2ba23`)
- `docs(sow_module)`: recorded a real, live Snowflake connection test and a grants gap found while
  testing it. (`0eb9300`)

## 2026-11-01 to 2026-11-06

### blend_ode_be
- `spike(sow_module)`: one shared HTML/LLM/paste/markdown intake pipeline. CP-14. (`514799d`)
- `spike(sow_module)`: backend `html_allowlist.py`, parity tests against the TypeScript allowlist,
  HTML fidelity GO/NO-GO decision. CP-13. (`9b6b586`)
- `spike(sow_module)`: vendored CP-12 HTML/Lexical round-trip fidelity files and fixtures. (`43ae685`)
- `spike(sow_module)`: crash-durability spike using real Yjs updates. CP-11. (`0e1d34b`)
- `spike(sow_module)`: lease and fencing-token spike script. CP-10. (`8ec6373`)
- `spike(sow_module)`: batched durable-log latency spike script. CP-09. (`34f2a19`)
- `spike(sow_module)`: Snowflake connection and capability spike scripts. CP-08. (`ec19556`)

### blend_ode
- `spike(sow_module)`: one intake pipeline for paste, LLM output, markdown. CP-14. (`8df94ff`)
- `spike(sow_module)`: `sanitizeHtml`, hostile-input fixtures, GO/NO-GO decision for HTML handling.
  CP-13. (`f0f09f4`)
- `spike(sow_module)`: HTML <-> Lexical round-trip fidelity using pinned Lexical 0.44.0. CP-12.
  (`a64ffb8`)

## 2026-10-30 — SOW editor module kickoff

### blend_ode_be
- `chore(sow_module)`: contracts and tooling scaffold for frontend, collab service and backend.
  (`03cb0cc`)
- `docs(sow_module)`: collaboration contracts, architectural decisions, and storage schema. CP-05.
  (`660220b`)
- `docs(sow_module)`: frontend and backend audits (what exists today, what must not break).
  (`aa1852b`)
- Initial scaffold: `sow_module/` (`CLAUDE.md`, `README.md`, prompt/test logs), `sow_module/collab/`
  (Node service tooling — tsconfig, Vitest, ESLint, `.env.example`, `DataStore` interface and
  in-memory store), `sync_shared.sh`/`shared_hash.py`/`run_checks.sh`, `src/sow_module/` package and
  `ENV.md`, smoke and drift tests. (`8fd2375`)

### blend_ode
- `chore(sow_module)`: role table, tests and a snapshot helper for the collab module. (`756c955`)
- `feat(sow_module)`: shared contracts for roles, protocol, errors, bridge types, and HTML
  allowlist rules. (`47cbd5d`)
- Initial scaffold: `src/sow_module/shared/` (placeholder entry point, source of truth for shared
  TypeScript), `src/sow_module/testing/` (shared-folder drift test and hash helper),
  `src/sow_module/tsconfig.shared.json` (strict, no-DOM typecheck config). (`9bb0829`)

## 2026-11-06 — Documentation reorganization

### SOW_project (root; not committed to any repo — tracked here only)
- Moved all SOW editor module tracking/reference docs (previously inside `blend_ode_be/sow_module/`
  and `blend_ode_be/src/sow_module/`, plus the standalone `documentation/` narrative folder and the
  build-plan/run-guide files) into a new root-level `sow_docs/` folder, organized by type
  (`tracking/`, `build_plan/`, `guides/`, `narrative/`, `reference/`). Left a `MOVED.md` pointer at
  each original location. Updated path references inside the build plan, run guide, testing guide,
  narrative docs, and `blend_ode_be/sow_module/CLAUDE.md` to point at the new location. This move is
  staged (not committed) in `blend_ode_be` as of this writing — see `SESSION_LOG.md` for the
  uncommitted-state details and the open question about whether `sow_docs/tracking/` should be
  re-committed inside `blend_ode_be` instead of living outside both repos.
- Created this `agent-handoff/` system at the project root.

---

*Earlier history*: both app repos have commits before 2026-10-30 (the pricing tool / SOW Studio
application itself, pre-existing before the live-collaboration editor initiative). See each repo's
own `git log` for that history; it is not transcribed here.

## 2026-11-06 to 2026-11-09 — SOW editor CP-21 to CP-25, fixes

### blend_ode_be
- `feat(sow_module)` CP-21 collab service shell, ticket verification, FastAPI client, boundary and secret checks (21c1f01)
- `feat(sow_module)` CP-22 rooms, lease manager, persist queue (5cb4da9)
- `feat(sow_module)` CP-23 snapshots, reopen, seeding (2228568)
- `feat(sow_module)` CP-24 WebSocket protocol, role gating, validation boundary (83ae4a2)
- `fix(sow_module)` collab service starts on Windows (166120c)

### blend_ode
- `feat(sow_module)` CP-25 LexicalDraftEditor shell with the same props as DraftEditor (df475bb)
- `fix(pricing)` pricing pages no longer crash on a missing project start date (8853099)
- `fix(sow_module)` named import for LexicalErrorBoundary (485d3cc)

### Root
- `docs(sow_module)` CP-25 record (407467b)

## 2026-11-09 to 2026-11-11 — SOW editor CP-26 to CP-40 (partial), fixes

### blend_ode (frontend)
- `feat(sow_module)` CP-26 toolbar: inline formats, headings, alignment, undo and redo (`cedf2a1`)
- `feat(sow_module)` CP-27 highlight, font size, font family; span styles kept on import (`c1513aa`)
- `feat(sow_module)` CP-28 three-level lists with depth cap (`70d5fdb`)
- `feat(sow_module)` CP-29/30 tables: insert, rows, columns, header, widths, merge, shading (`4db5c7a`)
- `feat(sow_module)` CP-31 to 33 paste intake, placeholders, find and replace, stable block IDs (`8d389ec`)
- `feat(sow_module)` CP-34 to 36 page chrome, action bar, editor API, add-in adapter (`8283773`)
- `feat(sow_module)` CP-38 flagged screen integration (`0b02b88`)
- `fix(sow_module)` print: toolbar hidden, `@page` A4 at top level (`5d0b024`, `c64f5e7`)
- `feat(sow_module)` editor shown as an A4 page with 20 mm margins (`7d8c2b8`)
- `feat(sow_module)` CP-40 (partial) shared chunk framing (`f9441ad`)
- `feat(sow_module)` CP-40 (partial) browser provider sends and receives chunks (`0a0d6d7`)
- Dependencies: none added in this period.

### blend_ode_be (backend, incl. the collab Node service)
- `feat(sow_module)` CP-37 Owner checkpoint to the draft system; fixed the CP-24 validation boundary for
  the real Lexical Yjs shape; added `@lexical/yjs`, `@lexical/rich-text` (0.44.0) and `happy-dom`
  (20.14.6) to the collab service (`62e248c`)
- `feat(sow_module)` CP-38 Owner refuses to overwrite changes made elsewhere (`8744189`)
- `feat(sow_module)` CP-40 (partial) Owner chunked sync; 100-connection room cap (`6e643e4`)
- `chore(sow_module)` shared re-sync (`847555f`; earlier `ad942b2`, `1b2d13d`, `e29ecf7`)
- `fix(sow_module)` table cell style groups in `html_allowlist.py` (`bce5042`)

### Root
- `docs(sow_module)` CP-26 to CP-39 logs; `core_api_freeze.md`; `phaseD_report.md` (`ed04323` to `f114978`)
- `chore` ignore the local to-do file (`6ea7448`)
