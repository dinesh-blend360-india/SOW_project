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

---

## 2026-11-11 — CP-40 completed

### blend_ode (frontend)
- `feat(sow_module)` awareness throttle (200ms trailing window), presence list read from Yjs
  awareness state, `PresenceStack` avatars in the page chrome; browser-level 5 MB chunked-sync test;
  `matchMedia` jsdom shim for antd's `Avatar.Group` (`dd18554`)

### blend_ode_be (backend, incl. the collab Node service)
- `test(sow_module)` two-real-client awareness relay test in the collab service (`b163eb4`)

CP-40 is now complete. Still open from CP-39: the two-user manual test (explicitly deferred by the
user this session, not forgotten) — see `TASK_BOARD.md` and `sow_docs/tracking/phaseD_report.md`.

---

## 2026-11-11 (later) — CP-41 completed

### blend_ode (frontend)
- `feat(sow_module)` reconnect with exponential backoff/full jitter and a fresh session per attempt
  (`SowCollabProvider`); `requestCollabSession` extracted for standalone reuse; "last saved" time in
  the status bar; test proving an offline edit resends exactly once (`fb1d77a`)

No backend/collab change this checkpoint. Next: CP-42 (Snowflake-unreachable rule, graceful
shutdown).

---

## 2026-11-11 (latest) — CP-42 completed, with an Opus correctness review

### blend_ode_be (backend, the collab Node service)
- `feat(sow_module)` the Snowflake-unreachable rule (`OutageGuard`: periodic lease renewal, a
  permanent read-only latch once a room's own lease validity lapses) and graceful shutdown
  (`CollabService.shutdown()`: drain, checkpoint, release leases, evict clients, bounded by a
  timeout with a hard-exit backstop). Hardened after a targeted Opus review found and this session
  fixed: an un-gated checkpoint path on a read-only room, several unhandled-rejection crash risks,
  a narrow edit/lease-leak window during shutdown, and a pre-existing (CP-24) room_full connection
  leak. Three deeper, pre-existing or Snowflake-query-level gaps need live-Snowflake verification
  and are documented instead of fixed blind — see `KNOWN_ISSUES_AND_BLOCKERS.md` #14-16.
  (`b76201a`)

This was the last of the three checkpoints explicitly scoped for this session's budget plan
(CP-40/41 on Sonnet, CP-42 Sonnet + a short Opus review). Next checkpoint and model: to be decided
with the user.

---

## 2026-11-11 (even latest) — CP-43 completed (Sonnet only, per user direction)

### blend_ode_be (backend, the collab Node service)
- `feat(sow_module)` `SnapshotScheduler` (snapshot after N durable updates or an idle period,
  whichever first) and `DataStore.pruneUpdates` (compaction with a configurable safety tail,
  implemented for both `InMemoryDataStore` and `SnowflakeDataStore`, plus a shared contract test).
  Found and fixed a real pre-existing gap: `SnapshotManager.takeSnapshot` (CP-23) was never actually
  called from production code before this checkpoint built the trigger for it. (`7beb610`)

The user set Sonnet 5.5 as the only model going forward (no Opus, for cost reasons) — this
checkpoint and its own review were done entirely on Sonnet.

---

## 2026-11-11 (newest) — CP-45 completed

### blend_ode_be (backend, the collab Node service)
- `feat(sow_module)` periodic per-connection access re-check, the `POST /internal/access-changed`
  hook (FastAPI's `notify_access_changed` already called it; nothing existed on the Node side to
  receive it), and a revocation-poll backstop. Drive-by fix: `CheckpointReason` was missing
  `'shutdown'`, which the wire contract already had — CP-42's shutdown checkpoint had been labeled
  `'explicit'` instead. (`98438f2`)

### blend_ode (frontend)
- `feat(sow_module)` `describeSessionCloseCode`: a clear message when a live session closes because
  access was removed or the SOW was finalized, instead of a bare "Offline" status. (`e9ff19b`)

---

## 2026-11-11 — CP-48 completed

### blend_ode (frontend)
- `test(sow_module)` CP-48: verified the intake pipeline/parity test were already built (CP-13/14/31/
  36); added the two real gaps — text-format fuzzing and the hostile corpus through `intake()`
  itself, not just `sanitizeHtml`. (`dcd69eb`)

## 2026-11-12 — Fixes from the CP-40 to CP-48 review

### blend_ode_be
- `fix(sow_module)` overlapping flushes no longer lose or duplicate updates (KNOWN_ISSUES #15) (`d0c45e5`)
- `feat(sow_module)` server enforces the 3-level list cap, CP-28 gap (`e2e4842`)
- `fix(sow_module)` merged cells export to Word and PDF, CP-30 gap (`1a588fb`)

### Root
- `docs` KNOWN_ISSUES #15 marked resolved (`953df76`)

## 2026-11-12 — CP-49: generated content insert (editor side)

### blend_ode
- `feat(sow_module)` insertContent and add-in insertion audited, idempotent, role-gated; shared SHA-256 and
  `CONTENT_INSERTED` message; add-ins routed through the live editor in Lexical mode

### blend_ode_be
- `feat(sow_module)` Owner records `content_inserted` audit rows (hash only, Option A); shared copy synced

## 2026-11-12 — CP-50: streaming preview and smart paste

### blend_ode
- `feat(sow_module)` streaming preview (local ghost, single commit) and Markdown smart paste with undo to plain text

## 2026-11-12 — CP-51: existing drafts open in the live editor

### blend_ode
- `feat(sow_module)` conversion report notice for SOW admins, Open Items navigation in the live editor, draft details (open items, add-ins, name) reported to the Owner

### blend_ode_be
- `feat(sow_module)` Owner: conversion report, no-op checkpoint check for unedited older drafts, DRAFT_META, fidelity batch script; backend checkpoint accepts the reported details

## 2026-11-12 — CP-52: regeneration, intake sync and reset to latest in a live room

### blend_ode
- `feat(sow_module)` reload-latest in the live editor, room-replaced notice, latest-draft wiring on the screen

### blend_ode_be
- `feat(sow_module)` Owner: save-then-replace (prepare/replace internal endpoints), RELOAD_LATEST, saved-content hash, stale rooms
- `feat(sow_module)` hooks on regenerate, intake sync, continue and accept-suggestion

## 2026-11-12 — CP-53 (Phase F close) and CP-54 (design gate, participants)

### blend_ode
- `feat(sow_module)` removal notices for LLM insert; legacy-open notice for all roles with admin details; fuzzing at scale

### blend_ode_be
- `feat(sow_module)` GET /sows/{sow_id}/collab/participants (+ tests)

## 2026-11-12 — CP-55/56: mentions and anchored comments; live session fix

### blend_ode
- `feat(sow_module)` comment threads in the live document: select, comment, hover to highlight, reply, delete, mentions
- `fix(sow_module)` the live editor requested its session with `sow:<id>:draft`, which the backend rejects; now `sow-<id>`

### blend_ode_be
- `feat(sow_module)` checkpoint stores live comment threads in the native columns and delivers mentions through the native code
- `feat(sow_module)` Owner: commenters may change only the comments map; threads are sent when they change

## 2026-11-12 — CP-57/58: comments panel, persistence, audit, redaction

### blend_ode
- `feat(sow_module)` comments panel: filters, resolve/reopen, edit and delete own comments, admin redaction, orphan badge, states

### blend_ode_be
- `test(sow_module)` audit event names and redacted-text storage tests for live comments
