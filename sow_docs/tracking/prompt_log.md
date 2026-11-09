# Prompt log

## CP-00: Scaffold the module, tracking files and rules (2026-10-30)
Summary: Created the module folders and tracking files in both repos and recorded repository facts (see `../CLAUDE.md`).
Decisions/assumptions:
- Placement default assumed: tracking, docs and the Node collab service in `blend_ode_be/sow_module/`; component and shared TS in `blend_ode/src/sow_module/`. Not yet confirmed by the user.
- Both repos separate; `sow_project/` is not a repo.
- Node 22.23.4, npm 10.9.9 (installed 2026-10-30); whether MSAL is a dependency (`MsalConfig.ts` exists, package not in `package.json`).
Registry (new or changed):
- file: sow_module/CLAUDE.md, README.md, _tracking/*: rules, layout, logs
- file: src/sow_module/__init__.py (backend): package placeholder
- file: blend_ode/src/sow_module/shared/.gitkeep: placeholder for shared TS
Open questions for the user: Node version; confirm placement; MSAL package.
Handoff to next prompt: CP-01 audits CKEditor.

## CP-06 / CP-07 (tooling part only) (2026-10-30)
Summary: Added tooling without the contract-dependent parts. Role table (`roleToCapabilities`) and its tests are NOT done (need CP-05 contracts and role-mapping decision). Nothing here has been run for JS/TS: Node is not installed on this machine.
Decisions/assumptions:
- Shared hash algorithm lives in `sow_module/shared_hash.py` and is mirrored in `blend_ode/src/sow_module/testing/sharedHash.ts`.
- `SHARED.sha256` is written by `sync_shared.sh`; drift tests skip until it exists.
Registry (new or changed):
- file: sow_module/collab/{package.json,tsconfig.json,vitest.config.ts,eslint.config.js,.env.example,.gitignore}: Node service tooling
- interface: DataStore in sow_module/collab/src/data/store.ts: persistence boundary
- class: InMemoryDataStore in sow_module/collab/src/data/memory-store.ts (+ test)
- file: sow_module/sync_shared.sh, shared_hash.py, run_checks.sh: vendoring and combined checks
- file: tests/test_sow_module_smoke.py, tests/test_sow_module_shared_drift.py (backend)
- file: src/sow_module/ENV.md (backend): variable names
- file: blend_ode/src/sow_module/tsconfig.shared.json, testing/sharedHash.ts, testing/sharedDrift.test.ts
Open questions for the user: install Node 20+ so the JS checks can run.
Handoff to next prompt: CP-01.

## Setup verification (2026-10-30)
Summary: Node 22.23.4 installed; `npm install` run in both places; `run_checks.sh` passes 6/6. Added `shared/index.ts` placeholder so the shared typecheck has an input; ran `sync_shared.sh` (first SHARED.sha256). collab `engines` set to >=22. `npm install` in blend_ode touched `package-lock.json`; reverted with `git checkout`.
Open questions for the user: none new.
Handoff to next prompt: CP-01.

## CP-01: Audit DraftEditor — props, plugins, parity table (2026-10-30)
Summary: Read-only audit of `blend_ode/src/pricing-tool/components/DraftEditor.tsx` (754 lines) and its test file (425 lines, 19 active tests). Documented the full props contract (19 props), the 18-plugin CKEditor configuration, all 7 highlight options, 7 font sizes, 9 font families, table features, and every SOW-specific behavior (placeholder auto-unwrap, watermark, logo, provisional banner, read-only mode, save/download/copy-link, dirty indicator, add-in navigation). Created the parity table (41 rows) with Lexical plan and decision column.
Decisions/assumptions:
- No application files changed (read-only audit).
- `SerializedDraftComments` is `Record<string, unknown>` — an opaque blob, not a structured type.
- Two comment-edit tests are commented out; the comments-plugin integration is disabled.
- CKEditor highlight CSS variables come from `ckeditor5/ckeditor5.css`; exact RGB values need extraction for Lexical.
Registry (new or changed):
- file: sow_module/_tracking/audit_frontend.md: props table, parity table, test summary, TBD list
Open questions for the user:
- TBD 1: Table properties dialog — build custom, accept loss, or simpler context menu?
- TBD 2: Table cell properties dialog — same question.
- TBD 3: Table column resize — is `@lexical/table` experimental resize stable enough?
- TBD 4: Arbitrary pasted fonts — preserve all or normalize to the 9 allowed families?
Handoff to next prompt: CP-02 audits DraftEditorScreen (save, autosave, revisions, swap point).

## CP-02: Audit DraftEditorScreen — save flow, swap point, services (2026-10-30)
Summary: Read-only audit of `DraftEditorScreen.tsx` (1284 lines) and its integration test. Documented the complete save and autosave flow (15s interval, optimistic revision lock via `expected_revision`), historical-version read-only logic, download/copy-link implementation, and the exact swap point (lines 1097–1159). Audited 6 service files that read or write draft HTML. Identified 8 risks for the Lexical transition.
Decisions/assumptions:
- No application files changed (read-only audit).
- The swap point is confirmed: `DraftEditorScreen.tsx:1097` renders `<DraftEditor>` — this is the single replacement point.
- `SowStudioPage.tsx:2688` renders `DraftEditorScreen` unconditionally (no feature flag).
- `NativeDraftEditorScreen` is exported but not used in any routed page.
- Feature flags come from `localStorage.getItem("user_details").features` (array of strings).
- `FE_SOW_NATIVE_EDITOR` appears only in a test file, not application code.
Registry (new or changed):
- file: sow_module/_tracking/audit_frontend.md (part 2 appended): save/autosave flow, swap point, services table, risks list
Open questions for the user:
- Risk 2: Should we introduce a shared CSS class (`sow-draft-editable`) for both editors, or branch on editor type in the navigation helpers?
- Risk 3: Does Lexical's HTML serializer need to preserve `data-add-in-id`? Or is the text-matching fallback in `sowStudioAddInPresence.ts` sufficient?
Handoff to next prompt: CP-03 audits the backend (access, drafts, finalize, native editor).

## CP-03: Audit backend — access, drafts, finalize, native editor, migrations (2026-10-30)
Summary: Read-only audit of the backend. Documented the full `save_sow_draft` flow (revision-based optimistic locking, in-place update for normal saves, new-version insert for `create_version=True`), draft retrieval, generation, continue-draft, suggested versions, finalize pipeline (`publish_final_sow` → `assert_sow_finalizable` → S3/HubSpot), native editor evaluation (`SOW_NATIVE_*` tables, workspace save vs checkpoint, guest auth), access control chain, database config (`insecure_mode: True`), middleware, feature flags, and migrations 001–010. Created reuse table (17 entries) and do-not-break list (8 items).
Decisions/assumptions:
- No application files changed (read-only audit).
- Next migration number is **011** — migrations 011–013 were removed from the repo per `README_SOW_STUDIO_STAGING.md`.
- The `SOW_NATIVE_*` DDL exists only in the live database; migration 013 was removed.
- CORS config was not found in `common/middlewares.py`; needs user clarification.
Registry (new or changed):
- file: sow_module/_tracking/audit_backend.md: access control, save_sow_draft flow, draft retrieval, finalize, native editor, migrations, reuse table, do-not-break list
Open questions for the user:
- Q1: Migration numbering — does an applied-but-removed 011 exist in the live DB, or is 011 safe as the next number?
- Q2: `sanitize_rich_text` strips all class/style and has no `<mark>` — placeholders will not appear in the approval preview. Is this acceptable?
- Q3: Where is CORS configured? Not found in `common/middlewares.py`.
Handoff to next prompt: CP-04 audits everything that reads draft HTML and the LLM paths.

## CP-04: Audit HTML dependencies and LLM paths (2026-10-30)
Summary: Read-only audit of every piece of code that depends on draft HTML and every place generated text enters the draft. Documented 19 backend HTML consumers (from opaque pass-throughs like `_draft_revision` to structure-sensitive parsers like `_signatory_fields_are_complete`), 6 frontend HTML consumers (3 with CKEditor-specific selectors), 5 LLM paths (only `suggested_fix_agent` writes HTML back into the draft), and 6 backend test files with their DB mocking patterns. Identified CKEditor-specific markup in 2 test files (`<figure class="table">`). Compiled combined open questions list (9 items across CP-03 and CP-04).
Decisions/assumptions:
- No application files changed (read-only audit).
- `billing_schedule.py:_Tables` is a separate HTMLParser (not `_HtmlToBlocksParser`) — documented independently.
- `quality_guard.py:capture_quality_guard` does NOT directly parse draft HTML (hashes intake/pricing/sources inputs only).
- `sow_narrative_agent` output is JSON, never HTML — the frontend template renderer is the insertion point.
- `suggested_fix_agent` is the only LLM path that writes HTML into `SOW_DRAFT_VERSIONS`.
Registry (new or changed):
- file: sow_module/_tracking/audit_backend.md (part 2 appended): sections 14–17 (HTML consumers, LLM paths, test inventory, combined open questions)
Open questions for the user:
- Q4: Should the Lexical HTML contract avoid `<figure>` wrappers? (recommended yes)
- Q5: When to update `suggested_fix_agent` system prompt for Lexical HTML — CP-05 or later?
- Q6–Q8: Billing schedule headers, signatory regex, and literal brackets — notes for CP-05 contract.
- Q9: Test fixtures with CKEditor markup — add parallel Lexical fixtures or update existing?
Handoff to next prompt: CP-05 writes the contracts, roles and decisions.

## CP-05: Contracts, roles and decisions (2026-10-30)
Summary: Wrote the shared TypeScript contracts (roles, protocol, errors, bridge, html-contract JSON plus typed loader), `contracts.md` (7 endpoints with callers and gates, ticket, WebSocket lifecycle, role mapping, concurrency with the CKEditor save path) and `decisions.md` (11 ADRs, all Proposed). Types and documents only; `roleToCapabilities` is a fail-closed stub for CP-06. `tsc --noEmit` strict passes on `shared/`; `sync_shared.sh` re-vendored it.
Decisions/assumptions:
- Ticket travels in the first WebSocket message (`HELLO`), not the URL, so it stays out of logs. Binary frames = Yjs (byte 0 FrameKind), text frames = JSON control messages.
- Canonical HTML output uses `strong, i, u, s`; `b, em, strike, del` are import aliases; `figure, div, section, font, a` are unwrapped on import; `img` is dropped.
- `html-contract.json` is derived from audits and tests; table inline styles, `colgroup` and cell `text-align` are marked unverified (no saved real draft was available).
- Finalize is gated today exactly like editing (+ status Approved), so `sow_admin` cannot be derived from code; left as a question.
- D-B1 (reuse native guest/mention/audit tables) conflicts with CLAUDE.md rule and do-not-break item 7, and the `SOW_NATIVE_*` DDL is not in the repo; recorded in ADR-004.
- `html-contract.ts` casts the JSON import (string literals widen); a runtime schema test is deferred to CP-09.
Registry (new or changed):
- file: blend_ode/src/sow_module/shared/{roles,errors,protocol,bridge,html-contract}.ts, html-contract.json, index.ts: contracts
- function: roleToCapabilities in shared/roles.ts: stub (CP-06)
- file: sow_module/_tracking/contracts.md, decisions.md: contracts and ADRs
- file: sow_module/collab/src/shared/* (vendored) and SHARED.sha256 refreshed by sync_shared.sh
- endpoint: POST /sows/{sow_id}/collab/session; POST /sow-native/guest/{sow_id}/collab/session; WS /ws/sow/{sow_id}; POST /internal/sow-collab/{resolve-access,document-state,checkpoint}; POST /internal/access-changed (all proposed, none implemented)
Open questions for the user: Q1 role mapping, Q2 who may finalize, Q3 D-B1, Q4 D-B3, Q5 D-B4; also G1, Q-C1, Q-C2, Q-N1, TBD-1..4, samples of real saved drafts.
Handoff to next prompt: CP-06 sets up tooling and implements roleToCapabilities once Q1/Q2 are answered.

## CP-06: Frontend module tooling and role table (2026-10-30)
Summary: Confirmed `shared/` type-checks under strict mode with `src/sow_module/tsconfig.shared.json` (extends the app tsconfig; `lib: esnext`, `types: []`, so DOM and Node globals fail the check) and imports only relative files and one JSON file (no React, antd, Node or DOM). Confirmed CRA Jest discovers tests under `src/sow_module/`. Implemented `roleToCapabilities` for real, exported the table to `shared/role-table.json` from a test, and added `can()`.
Decisions/assumptions:
- Added a `rename` capability (the prompt requires editor to be unable to rename); only `sow_admin` has it. This is stricter than today's code, where any editor can pass `sow_name` to `save_sow_draft`. Contracts updated.
- Features are `Partial<Record<Capability, boolean>>`: only `false` removes a capability; `true` or absent changes nothing. Unknown role returns no capabilities.
- The role table JSON lives in `shared/` so `sync_shared.sh` vendors it to the backend; the test compares against it and rewrites only with `UPDATE_SOW_MODULE_SNAPSHOTS=1`.
- Q2 (who counts as sow_admin) is still unanswered; the table does not depend on it.
- No dependency changes; no existing files changed; `package-lock.json` untouched.
Registry (new or changed):
- function: roleToCapabilities, can, ROLE_CAPABILITIES in blend_ode/src/sow_module/shared/roles.ts: role table
- file: blend_ode/src/sow_module/shared/role-table.json: exported role table for backend comparison
- file: blend_ode/src/sow_module/testing/roles.test.ts: 9 role tests
- function: checkJsonSnapshot in blend_ode/src/sow_module/testing/jsonSnapshot.ts: snapshot helper for later tests
- file: sow_module/collab/src/shared/* (vendored) and SHARED.sha256 refreshed
Open questions for the user: Q2 who is sow_admin; should editors really be unable to rename (stricter than today)?
Handoff to next prompt: set up the Node service, the backend package and the combined check script.

## CP-07: Node service, backend package and combined checks (2026-10-30)
Summary: Completed the tooling started earlier. The collab service has package.json (only express, ws, yjs, y-protocols, zod, pino plus dev tooling), strict tsconfig, Vitest, ESLint and scripts typecheck, test, lint, dev. `DataStore` and `InMemoryDataStore` are in place with round-trip, lease and fencing tests. The backend smoke test and the shared-drift tests exist in both repos. `run_checks.sh` now has 7 steps (sync check, Jest, shared tsc, collab tsc, collab lint, Vitest, backend pytest) and all pass.
Decisions/assumptions:
- Fixed `DocumentRecord.sourceRevision` from number to string (the revision is a SHA-256 hash, see contracts.md).
- `appendUpdates(rows, fencingToken)` now rejects a token that is not the current lease token (`StaleFencingError`), implementing ADR-007 at the storage boundary.
- Env names reconciled with contracts.md: `COLLAB_SERVICE_KEY_NODE_TO_API`, `COLLAB_SERVICE_KEY_API_TO_NODE`, `COLLAB_API_BASE_URL`, `COLLAB_NODE_BASE_URL`, `COLLAB_WS_PUBLIC_URL`; header is `X-Sow-Collab-Service-Key` (matches shared/bridge.ts). Default flush interval 250 ms (ADR-006). Nothing is read by code yet; `config.py` untouched.
- Added a minimal `src/server.ts` (health route only) so the `dev` script has a target.
- `sync_shared.sh` also copies `role-table.json` to `tests/fixtures/sow_module_html/` for the later backend comparison. No existing dependencies changed; no Snowflake driver; no secrets.
Registry (new or changed):
- interface: DataStore, DocumentRecord, UpdateRow, SnapshotChunk, Lease, Revocation, AuditEvent, StaleFencingError in sow_module/collab/src/data/store.ts: persistence boundary (documents, updates, snapshots, leases, revocations, audit)
- class: InMemoryDataStore in sow_module/collab/src/data/memory-store.ts (+ test): in-memory fake
- file: sow_module/collab/src/server.ts: health-only server for `npm run dev`
- file: sow_module/collab/.env.example, src/sow_module/ENV.md: variable names only
- file: sow_module/sync_shared.sh, run_checks.sh, shared_hash.py; tests/test_sow_module_smoke.py, tests/test_sow_module_shared_drift.py; blend_ode/src/sow_module/testing/sharedDrift.test.ts
- file: tests/fixtures/sow_module_html/{html-contract.json,role-table.json}: vendored fixtures
Open questions for the user: Q2 who is sow_admin; D-B1; whether `/internal/*` can be blocked at ingress; confirm `PRICING_APP_DEV_USER`.
Handoff to next prompt: spikes begin by connecting to Snowflake from Node (needs the env values you add later).

## CP-08: Snowflake access and capability spike (2026-11-01)
Summary: Wrote the connection and capability spike scripts in full (`snowflakeClient.ts`, `00-connect.ts`, `01-capabilities.ts`), added `snowflake-sdk` (pinned to the patched `^3.4.1`, fixing a known credential-logging advisory) plus `@types/snowflake-sdk` and `dotenv`. No Snowflake credentials for the Node service exist on this machine, so **no live connection was made and no result is reported as measured** — confirmed with the user rather than guessed. `snowflake_findings.md` documents exactly what's needed to run it for real.
Decisions/assumptions:
- User confirmed (2026-11-01): write every CP-08 to CP-11 spike script in full now, mark every measured result PENDING LIVE RUN, do not fabricate numbers. A new root-level `testing_pending_guide.md` tracks every such item across the whole project.
- `SNOWFLAKE_SPIKE_SCHEMA` added as a new env var, explicitly separate from the module's permanent `BLEND_INTERNAL_IT_APPS.NEXDRAFT` schema (ADR-012) — spikes create/drop objects freely and must not touch the real schema.
- The application-table-denial probe in `01-capabilities.ts` is written to expect failure (reading `SOWS` should be denied); a successful read is reported as `UNEXPECTED`.
- `run_checks.sh` still passes 7/7 after adding the dependency.
Registry (new or changed):
- file: sow_module/collab/src/spikes/{snowflakeClient,00-connect,01-capabilities}.ts: connection + capability spike code
- dependency: snowflake-sdk ^3.4.1, @types/snowflake-sdk, dotenv (collab package.json)
- file: sow_module/collab/.env.example: added SNOWFLAKE_SPIKE_SCHEMA and SNOWFLAKE_PRIVATE_KEY note
- file: sow_module/_tracking/snowflake_findings.md: pending-run findings doc
- file: SOW_project/testing_pending_guide.md (new, root level): master tracker for everything needing live verification
Open questions for the user: the 5 items listed in snowflake_findings.md (Node service user/role, warehouse, scratch schema, CREATE SCHEMA permission, private key).
Handoff to next prompt: CP-09 writes the durable-log latency spike (same pending-run status).

## CP-09: Batched durable-log latency spike (2026-11-01)
Summary: Wrote the full latency-measurement spike (`02-durable-log.ts` + shared `percentiles.ts`): the 100-document, 5-minute, 500ms-batch flush simulation with per-update enqueue-to-commit latency, plus single/100/1000-row insert timing, point read, conditional update, statements-per-minute, and a 5 MB chunked-BINARY write/read-back check. Configurable via env for a fast smoke test before the full run. **No live run was performed** (same blocker as CP-08); no latency number, gate result, or tier recommendation is reported — `spike_durable_log.md` has a reporting checklist instead, to be filled in only from real script output.
Decisions/assumptions:
- Table creation tries `CREATE HYBRID TABLE` first and falls back to a standard table automatically, so the same script works regardless of CP-08's hybrid-table finding.
- 1 MB chunk size chosen for the 5 MB binary write as a conservative margin under the documented ~8 MB `BINARY` column limit (to be confirmed for real by CP-08).
- The warehouse-cost estimate is deliberately left as a formula, not a number, in the findings file — it depends on a warehouse size not yet chosen and on active-seconds data the script itself cannot observe (that's in Snowflake's own metering).
- `run_checks.sh` still passes 7/7.
Registry (new or changed):
- file: sow_module/collab/src/spikes/02-durable-log.ts: the architecture-gating latency spike
- file: sow_module/collab/src/spikes/percentiles.ts: p50/p95/p99 helper (shared with CP-10, CP-11)
- file: sow_module/_tracking/spike_durable_log.md: pending-run findings doc with a reporting checklist
Open questions for the user: same CP-08 blockers; additionally, which warehouse size to use for this spike (affects the cost estimate).
Handoff to next prompt: CP-10 writes the lease and fencing spike (same pending-run status).

## CP-10: Lease and fencing spike (2026-11-01)
Summary: Wrote the full lease spike (`03-lease.ts`): a MERGE-based conditional acquire/renew mirroring `crud.py`'s `_lock_sow_row_for_draft_write`, a batched renew-all-rooms statement, a fenced-write probe, a 1-document race and a 500-document race across multiple concurrent Snowflake connections, a zombie-holder test, and a kill-and-failover timing test. **No live run performed** (same blocker as CP-08/CP-09); no acquire latency, failover time, or pass/fail is reported — `spike_lease.md` has a reporting checklist instead.
Decisions/assumptions:
- The script races multiple Snowflake connections from one Node process rather than literally separate OS processes, with a documented recommendation to also run it from two terminals at once once it's live, for extra confidence.
- `acquireWithRetry` retries on transient concurrent-DML errors with backoff, since standard tables use optimistic concurrency (hybrid tables would instead row-lock) — which behavior actually occurs is itself something this spike needs to observe live, not assumed.
- Table creation auto-detects hybrid-table availability the same way CP-09's script does.
- `run_checks.sh` still passes 7/7.
Registry (new or changed):
- file: sow_module/collab/src/spikes/03-lease.ts: lease/fencing spike (acquire, batch renew, fenced write, races, zombie, failover)
- file: sow_module/_tracking/spike_lease.md: pending-run findings doc with a reporting checklist
Open questions for the user: same CP-08 blockers.
Handoff to next prompt: CP-11 writes the crash-durability and client-held-edits spike (same pending-run status).

## CP-11: Crash durability and client-held-edits spike (2026-11-01)
Summary: Wrote the full durability spike (`04-durability.ts`) using real Yjs `Y.Doc` updates (not synthetic bytes): a client that holds every update until `DURABLE`, an owner that persists via a `MERGE`-keyed-on-(deal_id,seq) write (so a resend after a crash is a no-op, not a duplicate), repeated kill-and-resend rounds, and a final check for durably-acked loss (must be 0) and duplicate rows (must be 0). **No live run performed** (same blocker as CP-08/09/10); both required counts are reported as pending, not guessed.
Decisions/assumptions:
- Fixed a type gap discovered while writing this spike: `snowflake-sdk`'s own `.d.ts` restricts bind values to `string | number | boolean | null`, omitting `Buffer`/`Uint8Array` even though the driver accepts them at runtime for `BINARY` columns. Added a `SpikeBind` type and one confined cast in `snowflakeClient.ts` instead of casting at every call site; removed the `as never` workarounds this replaced in `02-durable-log.ts`.
- This is a Phase-closing "STOP AND TEST" checkpoint per the build plan (review CP-09/CP-10/CP-11 together and confirm the storage tier) — that review cannot happen yet since all three spikes are pending live Snowflake access; flagged to the user rather than silently treated as done.
- `run_checks.sh` still passes 7/7.
Registry (new or changed):
- file: sow_module/collab/src/spikes/04-durability.ts: crash-durability spike using real Yjs updates
- file: sow_module/collab/src/spikes/snowflakeClient.ts: added SpikeBind type to correctly type BINARY binds (Buffer/Uint8Array), fixing a gap in snowflake-sdk's own types
- file: sow_module/collab/src/spikes/02-durable-log.ts: removed now-unnecessary `as never` casts after the SpikeBind fix
- file: sow_module/_tracking/spike_durability.md: pending-run findings doc with a reporting checklist
Open questions for the user: same CP-08 blockers; this checkpoint's own stop-and-test review (confirming the storage tier across CP-09/10/11) is deferred until those spikes actually run.
Handoff to next prompt: CP-12 starts the HTML-to-Lexical-to-HTML fidelity spike using synthetic fixtures (user confirmed 2026-11-01, no real draft samples available yet) — this one does not need Snowflake and can run for real now.

## CP-12: Lexical packages, the allowlist, and HTML <-> Lexical fidelity (2026-11-01)
Summary: Added the pinned Lexical 0.44.0 package set to `blend_ode` (not the latest 0.53.0, which requires TypeScript >=5.2; bisected the version history to find the newest release with no such peer requirement, then confirmed empirically that the whole existing project still typechecks clean under TypeScript 4.9.5 — no TypeScript upgrade). Built 23 synthetic HTML fixtures (user confirmed 2026-11-01: no real draft samples available yet) covering short/long documents, lists to depth 3, six table shapes (including the exact billing-schedule and signature-block structures the backend parsers expect), all 7 highlight classes, placeholders, font-size/family, text-align, a link, and CKEditor wrapper/alias tags. Wrote `shared/allowlist.ts` (import/export sanitizers driven by `html-contract.json`) and ran a real headless-Lexical round-trip test (Jest/jsdom: 30/30 assertions pass; plain Node/jsdom script: 23/23 fixtures pass). Zero defects on text, headings, lists, tables or highlights; merged cells survive — full detail and the difference list in `spike_html_fidelity.md`.
Decisions/assumptions:
- Link handling (CP-12's own flagged question): user confirmed 2026-11-01 — links are reduced to plain text, not preserved. Already matched the CP-05 contract's `importUnwrap: ["a", ...]`; confirmed, not changed.
- Two real defects were found and fixed, not anticipated: (1) `@lexical/mark`'s `MarkNode` cannot round-trip `<mark>` HTML — `importDOM()` returns null and `excludeFromCopy()` always excludes it from export; fixed with `SowHighlightNode`, a thin subclass overriding only the export exclusion. (2) Lexical's default `TextNode.exportDOM()` double-wraps formatted text in legacy tags (`<b><strong>`, `<i><em>`, etc.) for copy-paste compatibility; fixed with a custom `TextNode` exporter in `buildHtmlConfig` emitting exactly the contract's canonical tags. A third, smaller issue (`text-align: start` emitted as an explicit default, breaking JSON byte-stability with no semantic effect) is handled in `sanitizeExportedHtml`.
- `html-contract.json` needed no changes — every tag/class/style the fixtures required was already present from CP-05 (expected, since the fixtures were synthesized from the same audit the contract came from; real samples may still surface gaps).
- A fourth finding, not a defect in this module: `@lexical/headless`'s `withDOM()` helper (happy-dom) silently drops every table on import, with no error. `jsdom` used directly instead, which passes all 23 fixtures. Flagged for CP-37 (the Owner's server-side HTML export) in `testing_pending_guide.md` — do not use `withDOM()`'s default for anything containing a table.
- `blend_ode_be/sow_module/collab/tsconfig.json` now excludes the three new DOM/Lexical-dependent shared files (and fixtures) from the Node service's own typecheck — they're vendored for drift-tracking only; the Node service never imports them (it handles raw Yjs updates, not HTML DOM manipulation).
- `npm run check:html-fidelity:node` (new script, `tsx` + `jsdom`, both added as devDependencies) runs the same logic outside Jest, satisfying the "run it in Jest and in Node" requirement.
Registry (new or changed):
- dependency: lexical, @lexical/{react,rich-text,list,table,html,selection,utils,clipboard,mark,yjs,headless} (all 0.44.0), yjs 13.6.33 (blend_ode dependencies); tsx, jsdom (blend_ode devDependencies)
- file: blend_ode/src/sow_module/shared/allowlist.ts: contract-driven import/export sanitizers and Lexical HTMLConfig
- file: blend_ode/src/sow_module/shared/SowHighlightNode.ts: MarkNode subclass fixing the export-exclusion defect
- file: blend_ode/src/sow_module/shared/htmlFidelity.ts: environment-agnostic round-trip + semantic comparison logic
- file: blend_ode/src/sow_module/shared/fixtures/html/synthetic-01..23-*.html: 23 synthetic fixtures
- file: blend_ode/src/sow_module/shared/fixtures/expected/*.json: generated Lexical JSON per fixture, for a future backend parity test
- file: blend_ode/src/sow_module/testing/htmlFidelity.test.ts: 30 Jest assertions
- file: blend_ode/src/sow_module/testing/lexicalTsCheck.ts: TypeScript 4.9.5 compatibility probe for every added package
- file: blend_ode/scripts/check-html-fidelity-node.ts: the plain-Node counterpart (jsdom)
- file: blend_ode/src/sow_module/tsconfig.shared.json: added "dom" to lib (types only, documented why this doesn't violate the no-DOM-runtime-dependency rule)
- file: sow_module/collab/tsconfig.json: excluded the three browser-only shared files from the Node service's typecheck
- file: sow_module/_tracking/spike_html_fidelity.md: Part 1 (fixtures, packages, allowlist, defects found and fixed, per-fixture results, approved-loss list, open items)
- file: SOW_project/testing_pending_guide.md: added the CP-37 happy-dom finding; updated the real-draft-samples entry
Open questions for the user: none new blocking this checkpoint; CP-13 (Python parity and the HTML GO/NO-GO) and CP-37 (which DOM implementation the Owner uses) are downstream follow-ups already flagged above.
Handoff to next prompt: CP-13 tests this same output against the Python sanitizer and exporters and gives the GO or NO-GO.

## CP-13: Python parity, hostile inputs, and the HTML GO/NO-GO (2026-11-01)
Summary: Wrote `src/sow_module/html_allowlist.py` (`sanitize_draft_html`), a new, separate allowlist driven by the same vendored `html-contract.json` the TypeScript side uses — mirrors the existing `_RichTextSanitizer`'s streaming style, generalized to read rules from the JSON contract plus alias renaming. `sync_shared.sh` now also copies the contract into `src/sow_module/` for runtime use. Wrote `tests/test_sow_module_html_parity.py` (81 tests, all real): semantic parity (normalized text + surviving classes) between the Python and TypeScript allowlists on all 23 real fixtures, export acceptance (`build_sow_docx_bytes`/`build_sow_pdf_bytes`, no error) on all 23, a report-only pass through the existing `sanitize_rich_text` (no equality asserted — it's lossy by design), and 6 hostile fixtures (script tag, javascript: link, malformed style, depth bomb, giant table, HTML-in-markdown-text) neutralized on both sides. **Verdict: GO — HTML stays the system of record.** Full reasoning in `spike_html_fidelity.md` Part 2.
Decisions/assumptions:
- "Must equal" (the prompt's wording) is interpreted as semantic parity (text + classes), not byte-for-byte: jsdom (full HTML5 parser) and Python's stdlib `HTMLParser` (streaming, no table content-model) are different engines that will legitimately disagree only on already-malformed markup no real draft HTML produces (e.g. an orphan `<td>` outside a `<table>`). Documented explicitly rather than silently redefined.
- A real defect was found and fixed on the TypeScript side while building the hostile fixtures: a malformed `url(javascript:...)` style broke jsdom's CSS parser badly enough that the per-property removal loop never ran, letting the raw dangerous text survive untouched. Fixed with a raw-text safety-net check in `sanitizeExportedHtml` (also present in the Python version for defense in depth, though its own regex-based style parsing doesn't have the same gap in the same form).
- `application/rich_text.py` was not edited; `sanitize_rich_text` is only called for a side-by-side report (it strips `<mark>`/highlights/placeholders entirely on the public approval page — a known, pre-existing limitation, not caused by this module).
- Installed `python-docx==1.2.0` and `reportlab==5.0.1` into this machine's ad-hoc Python environment — both are already pinned in `requirements.txt`, just not previously installed here. The full backend pytest suite still can't collect in this environment (missing `fastapi` and other dependencies, a pre-existing gap unrelated to this module); only the module-specific tests were run.
- `run_checks.sh`'s backend step now also runs `test_sow_module_html_parity.py`.
Registry (new or changed):
- file: src/sow_module/html_allowlist.py: `sanitize_draft_html`, the backend's own contract-driven allowlist
- file: src/sow_module/html-contract.json: runtime copy, kept in sync by `sync_shared.sh`
- file: tests/test_sow_module_html_parity.py: 81 tests (parity, export acceptance, rich-text report, hostile inputs)
- file: blend_ode/src/sow_module/shared/allowlist.ts: added `sanitizeHtml` (combined import+export pass, independent of Lexical) and the style-attribute safety-net fix
- file: blend_ode/src/sow_module/testing/sanitizeHtmlParity.test.ts: generates the TS-side expected `.sanitized.html` fixtures (23 real + 6 hostile) that the Python test compares against
- file: blend_ode/src/sow_module/shared/fixtures/hostile/*.html (+ expected/): 6 hostile fixtures
- file: sow_module/sync_shared.sh: added the runtime `html-contract.json` copy and its drift check
- file: sow_module/run_checks.sh: backend step now includes the html parity test
- file: sow_module/_tracking/spike_html_fidelity.md: Part 2 (parity definition, export acceptance, rich-text report, hostile-input results, GO/NO-GO verdict)
Open questions for the user: none new. Real draft samples remain the one open item tracked in `testing_pending_guide.md`.
Handoff to next prompt: CP-14 spikes paste and LLM-text intake.

## CP-14: Paste and LLM-text intake spike (2026-11-01)
Summary: Wrote `shared/intake.ts`: one function, `intake({format, content})`, funneling Word paste, backend LLM output, markdown, and plain text through the exact same allowlist CP-13 proved has backend parity. markdown-it runs with `html: false` (raw HTML in markdown source is escaped, a layer of defense before the allowlist runs). 10 synthetic LLM-output fixtures and 5 synthetic Word-paste fixtures were built (user confirmed 2026-11-01: no real samples available, same as CP-12's precedent). Testing against the Word-paste fixtures — not hypothetical input — found and fixed two real defects in `sanitizeImportHtml`/`sanitizeExportedHtml` (CP-12/13's functions): an entirely unrecognized tag (`<o:p>`) survived untouched because the unwrap pass only checked the explicit `importUnwrap` list, not "anything not allowed"; and vendor-prefixed CSS (`mso-bidi-font-weight`) survived because the code trusted the DOM's own CSS parser to enumerate style properties, and jsdom doesn't recognize Microsoft's non-standard property names. Both fixed; all of CP-12's 23 fixtures and CP-13's 81 tests still pass. 400 fuzzed inputs (fixed seed) found no surviving script/event-handler/javascript: after the fixes.
Decisions/assumptions:
- `fast-check@3.23.2` used, not the current 4.x (pure ESM, incompatible with CRA's default Jest transform without ejecting).
- A first fuzzing run (before pinning a seed) found a test false-positive — the `javascript:` detection regex allowed an optional quote, letting it match coincidental plain text. Tightened to require a quote, in both `intake.test.ts` and the Python parity test, which had the same loose pattern.
- Leaked markdown syntax inside already-rendered LLM `html`-format output (e.g. `**bold**` left unconverted) is not auto-corrected by `intake()` — that's a content-quality issue for the agent prompts to fix upstream, not a security/allowlist gap.
- Timing (20-page synthetic paste, well under 100ms) was measured in Jest/jsdom on this machine, explicitly not the reference workstation named in the prompt.
Registry (new or changed):
- file: blend_ode/src/sow_module/shared/intake.ts: the one intake pipeline (html/markdown/text → sanitizeHtml)
- file: blend_ode/src/sow_module/shared/allowlist.ts: fixed default-unwrap-for-unrecognized-tags and replaced DOM-based style filtering with hand-written regex parsing (matching the Python side, which never had either bug)
- file: blend_ode/src/sow_module/shared/fixtures/intake/llm/synthetic-llm-01..10-*.html, intake/word-paste/synthetic-word-01..05-*.html: 15 synthetic fixtures
- file: blend_ode/src/sow_module/testing/intake.test.ts: 25 tests including 400 fuzzed inputs
- dependency: markdown-it 15.0.3, @types/markdown-it (blend_ode dependencies); fast-check 3.23.2 (devDependency)
- file: sow_module/collab/tsconfig.json: excluded intake.ts from the Node service's own typecheck (browser-only, same reason as allowlist.ts/htmlFidelity.ts)
- file: sow_module/_tracking/spike_intake.md: full findings, results, open items
Open questions for the user: none new. Real Word-paste and LLM-output samples remain the one open item (testing_pending_guide.md).
Handoff to next prompt: CP-15 runs the 100-client room spike and closes Phase B.

## CP-08 (update): real Snowflake connection succeeded; grants gap found (2026-11-06)
Summary: User added real Snowflake env values to `blend_ode_be/.env` and `blend_ode/.env`, and asked to confirm `SF_USER=PRICING_APP_DEV_USER` and the `BLEND_INTERNAL_IT_APPS.NEXDRAFT` restriction — both confirmed by reading the file (no secrets printed). Created `sow_module/collab/.env` (git-ignored) copying the Node service's identity from the backend's working credentials, with database/schema set to the module's target per ADR-012. Ran the real, live connection test: `npm run spike:connect` succeeded (`CURRENT_VERSION 10.40.102`, role and warehouse confirmed). Two further read-only diagnostics isolated a real finding: `PRICING_APP_DEV_ROLE` has no `USAGE` grant on `BLEND_INTERNAL_IT_APPS` at all (0 results from `SHOW DATABASES LIKE`), while the same role/key/warehouse can see `RESOURCE_REQUEST_TOOL` (the database the existing app actually uses) without issue — proving the credentials and auth are fine and the gap is specifically a missing grant on the new database. The capability probes (`01-capabilities.ts`) were not run, since they would fail on the same missing-grant error.
Decisions/assumptions:
- Did not run `01-capabilities.ts` (which creates/drops tables) without the database grant — would only reproduce the same error.
- User is requesting the grant from Snowflake administration; `GRANT USAGE ON DATABASE BLEND_INTERNAL_IT_APPS TO ROLE PRICING_APP_DEV_ROLE;` plus the schema grant on `NEXDRAFT` are the exact statements needed (not executed by this session — privileged change, outside standing authorization).
- Also found and fixed: `sow_module/collab/snowflake.log` (an auto-generated runtime log containing no secrets, only connection metadata) was not git-ignored; added `*.log` to `sow_module/collab/.gitignore`. Confirmed no `.env` file is tracked or staged in either repo.
- Noted, not acted on: `blend_ode/.env.development` is a pre-existing tracked file (predates this module) containing a CKEditor license token and an API key in plain text — unrelated to this module, flagged to the user, not modified.
Registry (new or changed):
- file: sow_module/collab/.env (git-ignored, not committed): real Snowflake identity for the Node service
- file: sow_module/collab/.gitignore: added `*.log`
- file: sow_module/collab/src/spikes/check-nexdraft.ts, check-existing-db.ts: read-only diagnostics (SHOW/SELECT only), kept for reuse once the grant lands
- file: sow_module/_tracking/snowflake_findings.md: real measured connect result; grants gap documented as the specific, isolated blocker
Open questions for the user: none — the next step is entirely in the user's hands (requesting the grant).
Handoff to next prompt: CP-15, with the persistence/lease-integrated portions still pending the same grant.

## CP-15: 100-client room spike and Phase B report (2026-11-06)
Summary: Ran the portions of this spike that don't need Snowflake, for real: `05-scale.ts` (100 real Yjs clients over real WebSocket connections to a minimal in-process owner, steady-typing simulation, 30 seconds) measured broadcast fan-out p50 9ms / p95 72ms / p99 170ms across 97,812 deliveries; `06-proxy-hop.ts` measured proxy-hop overhead (hardcoded routing table standing in for the CP-10 lease, since that's still pending the grant) at effectively 0ms added latency on loopback. A real bug was found and fixed while building the proxy spike: the proxy only attached its forwarding handler after its own upstream connection opened, so an early client message was silently dropped and the client hung forever — fixed by buffering pre-open messages. The persistence-under-load and real-lease-failover portions remain unmeasured (same `BLEND_INTERNAL_IT_APPS` grant blocker as CP-08/09/10/11). `phaseB_report.md` records the honest state: **PENDING, not GO** — the one number that was always meant to decide the storage tier (CP-09's durable-ack latency) has never been measured, and nothing here substitutes for it.
Decisions/assumptions:
- User explicitly instructed proceeding to CP-16/17 despite Phase B not reaching a real GO. This is recorded as a deliberate, flagged exception to the plan's own stop-and-test gate, not a silent assumption that the gate was cleared.
- The 100-client numbers are explicitly labeled local/not-production-shaped per the prompt's own instruction: owner and all 100 clients ran in one Node process on one machine.
- No tier default is hardcoded anywhere in code from this; CP-16's migration will use Tier 2 (standard tables) as a clearly-flagged provisional choice, pending the real CP-08/09 results.
Registry (new or changed):
- file: sow_module/collab/src/spikes/05-scale.ts: 100-client broadcast fan-out spike (real, run)
- file: sow_module/collab/src/spikes/06-proxy-hop.ts: proxy-hop overhead spike (real, run)
- file: sow_module/_tracking/spike_scale.md: measured results, explicitly scoped to what was and wasn't tested
- file: sow_module/_tracking/phaseB_report.md: all Phase B spikes summarized; PENDING decision recorded with reasons
Open questions for the user: the `BLEND_INTERNAL_IT_APPS` grant (tracked since CP-08); team size and rollout schedule for the re-baseline note, not yet confirmed.
Handoff to next prompt: CP-16 writes the SOW_COLLAB migration file (not applied), using Tier 2 as a flagged provisional default.

## CP-16: SOW_COLLAB tables migration, written and not applied (2026-11-06)
Summary: Confirmed the migration number with the user before writing anything, per the prompt's explicit requirement. Rather than just asking, checked what's actually verifiable: `git log --all` shows no trace of `migrations/011_*.sql` ever having existed in this repository's history, and a read-only `SHOW TABLES LIKE 'SOW%'` against the schema the existing app actually uses returned exactly the 12 tables migrations 001-010 account for, nothing more. (This also surfaced an unrelated finding: the earlier CP-03 audit's claim that `SOW_NATIVE_*` tables "exist in the live database" is not true in this schema — flagged separately, not resolved here.) User confirmed proceeding with 011. Wrote `migrations/011_sow_collab_live_workspace.sql`: 7 tables (`SOW_COLLAB_DOCUMENT`, `_UPDATE`, `_SNAPSHOT`, `_LEASE`, `_REVOCATION`, `_IDEMPOTENCY`, `_AUDIT`), every one fully qualified to `BLEND_INTERNAL_IT_APPS.NEXDRAFT`, using `CREATE TABLE IF NOT EXISTS` with a `COMMENT` on each, in the style of the existing numbered migrations. **Not executed against Snowflake.** Added `tests/test_sow_module_migration_011.py` (5 tests, text-parsing only, no Snowflake connection): every created table is prefixed `SOW_COLLAB_` and fully qualified; no `ALTER`/`DROP`/`TRUNCATE` anywhere in the file; no existing application table is referenced as an actual statement target (only in prose comments).
Decisions/assumptions:
- **Tier: standard tables (Tier 2), explicitly flagged as provisional.** CP-08's hybrid-table probe and CP-09's latency gate — the two things meant to decide this — have not run (phaseB_report.md: PENDING, not GO). Standard tables chosen as the safer always-available default, not as a measured decision. Revisit once CP-08/09 actually run.
- D-B1 (reuse `SOW_NATIVE_AUDIT_EVENTS` vs. a new audit table) is still Proposed, not decided — `SOW_COLLAB_AUDIT` created as its own table; the migration's comment says so explicitly so a future reader isn't misled into thinking this was a final decision.
- A test-writing bug was found and fixed while building the guard pytest: the first version flagged a harmless mention of `SOW_DRAFT_VERSIONS` inside an inline `COMMENT = '...'` documentation string as if it were a real SQL reference. Fixed by stripping `COMMENT = '...'` clause text (in addition to `--` line comments) before checking for existing-table mentions.
Registry (new or changed):
- file (not applied): migrations/011_sow_collab_live_workspace.sql — 7 new tables, all SOW_COLLAB_*, all in BLEND_INTERNAL_IT_APPS.NEXDRAFT
- file: tests/test_sow_module_migration_011.py: 5 guard tests, text-parsing only
- file: sow_module/collab/src/spikes/check-sow-schema-tables.ts: read-only diagnostic used to check for migration 011 evidence (kept for reuse)
Open questions for the user: the SOW_NATIVE_* table-location discrepancy found above (separate from this checkpoint's scope); D-B1 remains open; the tier remains provisional pending CP-08/09.
Handoff to next prompt: CP-17 writes the grants file and runbook, and asks for explicit approval of both SQL files.

## Tier decision: standard tables, made by the user (2026-11-06)
Summary: After the `BLEND_INTERNAL_IT_APPS`/`NEXDRAFT` `USAGE` grants landed, ran `npm run spike:capabilities` for real (CP-08's remaining probes) against the now-accessible, confirmed-empty `NEXDRAFT` schema. Results: regular `CREATE TABLE` + `BINARY` storage + `TIMESTAMP_TZ` all work for `PRICING_APP_DEV_ROLE`; the real `SOWS` table is correctly denied (intended); `CREATE HYBRID TABLE` specifically is denied (its own separate Snowflake grant, distinct from plain `CREATE TABLE`). Explained to the user why hybrid tables were the original plan (row-level locking for the lease, frequent small writes for the update log). Rather than request the additional grant, **the user decided to proceed with standard tables (Tier 2) as the final decision**, given CP-15's broadcast spike showed nothing pathologically slow. CP-09's durable-ack latency gate — the measurement that would have added a real performance comparison — was not run before this decision. Updated `migrations/011_sow_collab_live_workspace.sql`'s header comment, `README_SOW_COLLAB.md`, `spike_durable_log.md`, `phaseB_report.md`, `snowflake_findings.md`, and `SOW_project/testing_pending_guide.md` to record this plainly: a user decision made with partial information, not a measured side-by-side comparison, with that distinction preserved for anyone reading any of these files later.
Decisions/assumptions:
- The tier is now final, not provisional. CP-09/10/11/15's remaining Snowflake-integrated portions are reclassified from "blocking" to "optional follow-up for extra confidence" — the grants needed to actually run them now exist, so nothing stops them being run later if desired, but nothing downstream waits on them anymore.
- `NEXDRAFT` confirmed clean (0 tables) both before and after today's capability probe run — nothing was left behind.
Registry (new or changed):
- file: migrations/011_sow_collab_live_workspace.sql: header comment rewritten — tier is now a final, dated decision with full reasoning, not "provisional"
- file: migrations/README_SOW_COLLAB.md: status section updated to match
- file: sow_module/_tracking/{snowflake_findings.md,spike_durable_log.md,phaseB_report.md}: updated with today's live capability-probe results and the tier decision
- file: SOW_project/testing_pending_guide.md: CP-08 resolved; CP-09/10/11/15's Snowflake-integrated portions reclassified as optional, non-blocking follow-ups
Open questions for the user: none new. CP-09/10/11's full live runs remain available as optional follow-ups, not required.
Handoff to next prompt: CP-17's migration approval — the updated `011` reflects this final tier decision; re-presenting for approval.

## CP-18: Node Snowflake data layer, implemented and run live against NEXDRAFT (2026-11-06)
Summary: Built `sow_module/collab/src/data/snowflake-store.ts`, a `SnowflakeDataStore implements DataStore` covering every method the interface defines (`getDocument`, `putDocument`, `appendUpdates`, `readUpdates`, `putSnapshot`, `getLatestSnapshot`, `acquireLease`, `renewLeases`, `getLease`, `releaseLease`, `addRevocation`, `listRevocations`, `appendAudit`), using `snowflake-sdk`'s connection pool (via `generic-pool`, added as an explicit dependency), key-pair auth, bind variables only, a per-connection `ALTER SESSION SET STATEMENT_TIMEOUT_IN_SECONDS` (the one place a value is interpolated rather than bound, since `ALTER SESSION` does not accept bind parameters — guarded by an explicit `Number.isFinite`/`> 0` check first), and a retry-on-transient-error wrapper (regex-matched: ECONNRESET, ETIMEDOUT, EPIPE, socket hang up, connection down, warehouse suspended/resuming). Added `createDataStore()` factory reading `SOW_STORE` (`memory` default, `snowflake` opt-in) and `loadSnowflakeStoreConfigFromEnv()`. Extracted the existing in-memory contract tests into a shared, reusable suite (`src/data/contract-tests.ts`, `runDataStoreContractTests(label, makeStore, dealIdFactory?, timeoutMs?)`) so the same behavioral contract runs against both `InMemoryDataStore` and `SnowflakeDataStore` without duplicating test bodies; `memory-store.test.ts` now calls the shared suite plus keeps its own implementation-specific deterministic-clock tests. Wrote `src/data/snowflake-store.integration.test.ts`, gated behind `RUN_SNOWFLAKE_INTEGRATION=1` (skipped by default, including in `run_checks.sh`'s `collab tests (Vitest)` step) so a normal test run never touches Snowflake: it creates 6 temporary `SOW_COLLAB_TEST_<random-12-hex>_*` tables inside `BLEND_INTERNAL_IT_APPS.NEXDRAFT` (no separate scratch schema exists — the same deviation already recorded for the CP-08 spikes), runs the full shared contract suite against them via `SnowflakeDataStore`'s `tablePrefix` option, and drops all 6 in an `afterAll` regardless of pass/fail. Wrote `src/data/snowflake-store.sql-safety.test.ts`, a static source scan (no Snowflake connection) asserting every SQL template literal only interpolates `this.table(...)`/`placeholders`/the validated `seconds` value (never request data), that the one non-bound interpolation is preceded by the finite-positive-integer check, and that every `this.exec(...)` call site includes a binds array argument.
**Ran the integration test live, with the user's explicit approval after an explanation of exactly what it does** (creates 6 brand-new temp tables, writes/reads only inside them, drops all 6 afterward, touches nothing else). First live run surfaced a real bug: `appendUpdates`/`putSnapshot` bound a raw `Buffer` directly into a `MERGE ... USING (SELECT ? AS ...)` subquery, and the driver silently mis-typed it, sending `JSON.stringify(buffer)` (`'{"type":"Buffer","data":[1]}'`) instead of binary data, which Snowflake then rejected as "not a legal hex-encoded value." Fixed by hex-encoding the buffer in JS (`Buffer.from(bytes).toString('hex')`) and converting explicitly in SQL with `TO_BINARY(?, 'HEX')`, removing the ambiguity. Second live run hit vitest's default 5-second per-test timeout (each real Snowflake round trip takes ~0.3-1.3s and one test makes 6+ sequential calls) — added a `timeoutMs` parameter to the shared suite (default 5000, the live integration file passes 30000). Third live run: **all 6 contract tests passed for real against Snowflake.** Verified afterward with a direct `SHOW TABLES IN SCHEMA BLEND_INTERNAL_IT_APPS.NEXDRAFT` that the schema is back to 0 tables — nothing left behind.
Decisions/assumptions:
- The SQL-concatenation-safety test is a static regex/source scan of `snowflake-store.ts`, not a runtime SQL-injection test — appropriate since every value in this file already goes through bind variables or `this.table()` (a fixed prefix + literal suffix, never request-controlled).
- `SOW_COLLAB_IDEMPOTENCY` (created by migration 011) has no corresponding `DataStore` method yet — not created by the integration test's temporary-table setup, since nothing in CP-18's scope writes to it.
- Full `run_checks.sh` (all 7 checks across both repos) re-run clean after this change: 93 backend pytest, 68 frontend Jest, 11 collab Vitest (6 live-integration tests correctly skipped by default).
Registry (new or changed):
- file: sow_module/collab/src/data/snowflake-store.ts: `SnowflakeDataStore` (full `DataStore` implementation), `loadSnowflakeStoreConfigFromEnv`, `createDataStore`
- file: sow_module/collab/src/data/contract-tests.ts: `runDataStoreContractTests` — shared behavioral contract for any `DataStore`
- file: sow_module/collab/src/data/memory-store.test.ts: rewritten to call the shared suite plus its own deterministic-clock-specific tests
- file: sow_module/collab/src/data/snowflake-store.integration.test.ts: live Snowflake integration test, opt-in only (`RUN_SNOWFLAKE_INTEGRATION=1`), temp-table lifecycle
- file: sow_module/collab/src/data/snowflake-store.sql-safety.test.ts: static guard against string-built SQL
- dependency: sow_module/collab/package.json: added `generic-pool` (dependencies), `@types/generic-pool` (devDependencies)
Open questions for the user: none. CP-09/10/11's full live lease/durability spikes remain optional follow-ups, unaffected by this checkpoint.
Handoff to next prompt: CP-19 — FastAPI session endpoint, ticket, and feature flag.

## CP-19: FastAPI session endpoint, ticket, and feature flag (2026-11-06)
Summary: Added the browser-facing `POST /pricing_tool/sows/{sow_id}/collab/session` endpoint, following contracts.md sections 1-3 exactly (the authoritative, already-written spec from CP-05, cross-checked against `bridge.ts`'s `SessionResponse`/`TicketClaims` and `roles.ts`'s `ROLE_CAPABILITIES`). Added `FE_SOW_LEXICAL_EDITOR` to `src/common/config.py`'s `FeaturesList`, plus the 5 env vars named in `src/sow_module/ENV.md` (`COLLAB_TICKET_SECRET`, `COLLAB_SERVICE_KEY_NODE_TO_API`, `COLLAB_SERVICE_KEY_API_TO_NODE`, `COLLAB_NODE_BASE_URL`, `COLLAB_WS_PUBLIC_URL`), all `os.getenv(...)` with no default, so the feature stays fully disabled (session endpoint returns 403) until the user provides real values. Wrote `src/sow_module/collab_session.py` (`resolve_role`, `mint_ticket`, `build_session_response`, `collab_feature_configured`): tickets are HS256 JWS (via `jose.jwt`, matching the project's existing JWT style in `pricing_tool/dependencies.py`), signed with `COLLAB_TICKET_SECRET` (never `JWT_SECRET`), 30-second lifetime, claims exactly matching `TicketClaims` (`jti` random 128-bit, `aud="sow-collab"`, `sow_id`, `actor` as an `ActorRef`-shaped dict, `role`, `iat`, `exp`). Wrote `src/sow_module/router.py`: `authorize_sow_lexical_editor` (same pattern as the existing `authorize_sow_native_editor` in `sow_studio/dependencies.py` — isolates this pilot from both the production editor and the other native-editor pilot), the session endpoint itself (feature-not-configured -> 403, `assert_deal_access` -> 403 on no deal relationship, otherwise mints a ticket). Registered the new router in `src/main.py` under the same `/pricing_tool` prefix `sow_studio_router` already uses, confirmed via `app.openapi()` that the route lands at `/pricing_tool/sows/{sow_id}/collab/session`.
Added a Python mirror of the frontend's role table: `src/sow_module/role_table.py` (loads the vendored `role-table.json`, exposes `role_to_capabilities`/`can`, fails closed on an unknown role) plus `tests/test_sow_module_role_table_parity.py` (5 tests: roles/capabilities match the vendored fixture exactly, strictly-increasing privilege, `sow_admin` is the only role that can finalize, unknown role fails closed). Extended `sync_shared.sh` to vendor a runtime copy of `role-table.json` into `src/sow_module/` (mirroring the existing `html-contract.json` runtime-copy pattern) with the same `--check` drift detection, and re-ran it clean.
**Role mapping decision, explicitly flagged as an assumption, not a new resolution of the open question**: contracts.md section 7 records that "finalize today is gated by the same check as editing" and "`FE_ADMIN` is the only existing elevated flag" — Q2 (decisions.md) on who exactly counts as `sow_admin` is still open. `resolve_role` therefore maps `FE_ADMIN` -> `sow_admin`, everyone else who passes `assert_deal_access` -> `editor`. Guest sessions (the `commenter` role, endpoint 2 in contracts.md) are explicitly deferred to CP-60 per the build plan; this checkpoint only builds endpoint 1.
Wrote `tests/test_sow_module_collab_session.py` (6 tests): feature flag off -> 403; feature not configured (env unset) -> 403; no deal access -> 403; a Blend participant gets a `editor` ticket with `checkpoint` but not `finalize`; `FE_ADMIN` gets a `sow_admin` ticket with `finalize`/`manage_access`; the ticket's `exp - iat == 30` and a ticket with a manually-expired `exp` fails `jose`'s own decode-time expiry check. Confirmed `test_sow_http_e2e.py` (uses the same `authorize_sow_access` dependency-override pattern), `test_native_collaboration.py`, `test_sow_revision_integrity.py`, `test_sow_rich_text.py` all still pass unchanged (48/48). Full suite: 534 passed, the same 9 pre-existing failures as the established baseline (date-dependent template tests and a duration-rounding test, all unrelated to this work), confirming no regression. `run_checks.sh` extended to include the two new test files; full 7/7 checks green.
Decisions/assumptions:
- `wsUrl` in the session response is `COLLAB_WS_PUBLIC_URL` directly (a single public URL), not a per-document lease lookup — CP-10's lease and any multi-instance routing layer don't exist yet. Documented here so CP-25+ (once a real lease/routing exists) revisits this rather than assuming today's single-URL behavior is final.
- The "finalize-capable = sow_admin" mapping uses `FE_ADMIN` only, consistent with contracts.md's own "fact from the code" and not inventing a new deal-owner-based rule the contract doesn't actually specify.
Registry (new or changed):
- file: src/common/config.py: added `FE_SOW_LEXICAL_EDITOR` feature flag; `COLLAB_TICKET_SECRET`, `COLLAB_SERVICE_KEY_NODE_TO_API`, `COLLAB_SERVICE_KEY_API_TO_NODE`, `COLLAB_NODE_BASE_URL`, `COLLAB_WS_PUBLIC_URL` env vars
- file: src/sow_module/collab_session.py: `resolve_role`, `mint_ticket`, `build_session_response`, `collab_feature_configured`
- file: src/sow_module/router.py: `authorize_sow_lexical_editor` dependency; endpoint: `POST /pricing_tool/sows/{sow_id}/collab/session`
- file: src/sow_module/role_table.py: `role_to_capabilities`, `can`, `ROLES`, `CAPABILITIES`
- file: src/sow_module/role-table.json: runtime-vendored copy (sync_shared.sh)
- file: sow_module/sync_shared.sh: vendors and drift-checks the runtime role-table.json copy
- file: src/main.py: registered `sow_module_router` under `/pricing_tool`
- file: tests/test_sow_module_role_table_parity.py, tests/test_sow_module_collab_session.py
- file: sow_module/run_checks.sh: added the two new test files to the backend pytest line
Open questions for the user: decisions.md Q2 (who exactly counts as sow_admin) remains open; this checkpoint's `FE_ADMIN`-only mapping is a documented stand-in, not a resolution.
Handoff to next prompt: CP-20 — FastAPI internal endpoints (resolve-access, document-state, checkpoint) under `/internal/sow-collab/*`, service-key protected, using `html_allowlist.sanitize_draft_html` and the existing `save_sow_draft` path.

## CP-20: FastAPI internal endpoints (resolve-access, document-state, checkpoint) (2026-11-06)
Summary: Built the three `/internal/sow-collab/*` calls only the Node collaboration service may make (contracts.md section 4, types mirrored from `bridge.ts` exactly, same as CP-19). Wrote `src/sow_module/collab_internal.py` with pydantic request/response models using `alias_generator=to_camel` (so the wire format matches `bridge.ts`'s camelCase `ResolveAccessRequest`/`DocumentStateRequest`/`CheckpointRequest` exactly, while Python code stays snake_case) and three functions: `resolve_access`, `get_document_state`, `checkpoint`, plus `notify_access_changed` (the FastAPI -> Node hook). Wrote `src/sow_module/router_internal.py`: a `verify_collab_service_key` dependency (constant-time `hmac.compare_digest` against `COLLAB_SERVICE_KEY_NODE_TO_API`, header `X-Sow-Collab-Service-Key` -- the exact constant `bridge.ts` defines as `SERVICE_KEY_HEADER`), and the three routes, all under `include_in_schema=False` so they never appear in the public OpenAPI schema. Registered at the FastAPI root (no `/pricing_tool` prefix, matching `bridge.ts`'s absolute `BRIDGE_PATHS`), confirmed via `app.openapi()` that `/internal/*` paths are absent from the schema while still live and reachable (confirmed a bare POST without a key returns 401).
**Access-control design, reusing existing functions without copying their logic (constraint)**: `resolve_access` is the one call that evaluates a real actor's access -- it fetches the actor's real features via the existing `get_role_and_features` (the same function `authorize_token` already uses), builds a `SimpleNamespace` identity from them, and calls the real `assert_deal_access` unmodified. This was a deliberate fix after noticing a draft version would have passed an empty features list for the actor, which would make `assert_deal_access`'s `FE_ADMIN` bypass silently never trigger for an actual admin who isn't also a deal collaborator -- exactly the scenario the CP-20 acceptance test "matches `assert_deal_access` for FE_ADMIN" is there to catch. `document-state` and `checkpoint`, per contracts.md's own gate column, carry no further per-actor check beyond the service key -- they use a synthetic `SimpleNamespace` service identity with `FE_ADMIN` (same technique `native_collaboration.py`'s guest paths already use for guests, who also have no real `User`) purely to satisfy `assert_deal_access`/`save_sow_draft`'s existing signatures, never written anywhere as an attribution. `checkpoint`'s actual attribution (`CREATED_BY_EMAIL`/`CREATED_BY_NAME`) comes from the request's `attributedTo` field (the last human editor in the window), per contracts.md section 4.
`checkpoint` fetches the current draft first (via the real `get_latest_sow_draft`) and passes its `comments`/`addedAddIns`/`openItems` through unchanged into `save_sow_draft`, matching contracts.md's "so the editor module changes content only" -- passing nothing would have silently wiped existing comments. It explicitly rejects a finalized SOW before attempting a save (`409 sow_locked`), skips the write entirely and returns `unchanged: true` when the sanitized HTML equals the stored content (avoiding a no-op revision bump), and maps `save_sow_draft`'s remaining `409`/`428` to `{code: stale_revision | revision_required, message}` structured errors, satisfying "on a 409 or 428 return a structured error so the Owner can refresh the source revision and retry once."
**Explicitly flagged scope decision**: `fencingToken` and `upToSeq` are accepted in `CheckpointRequest` (so the eventual Node client doesn't need redesigning once a stronger check exists) but not independently re-verified against `SOW_COLLAB_LEASE` -- FastAPI's existing Snowflake connection targets `RESOURCE_REQUEST_TOOL`/`PRICING_TOOL_SANDBOX_HYBRID`, not `BLEND_INTERNAL_IT_APPS.NEXDRAFT` where the lease lives, and building a second Snowflake connection pool purely for this check is out of scope for the literal CP-20 task list (which does not mention fencing-token verification at all -- only contracts.md's earlier, more detailed design does). The Node service already enforces fencing before any batch is durably written (CP-18's `appendUpdates`), which is the practical protection against a stale/zombie writer; this is recorded here, not silently assumed, per ADR-012's own note that the fencing token (not a shared transaction) is the real cross-schema safety mechanism.
Added `notify_access_changed(sow_id, reason, actor_id=None)`: POSTs to the Node service's `/internal/access-changed` with `COLLAB_SERVICE_KEY_API_TO_NODE`; returns immediately (no-op) if the Node service isn't configured; catches and logs any exception so a failure never blocks the caller. Wired into `revoke_guest_invitation` (`native_collaboration.py`) -- a single added line after its existing commit, its return value unchanged, confirmed by rerunning `test_native_collaboration.py`.
Wrote `tests/test_sow_module_collab_internal.py` (11 tests): missing/wrong service key -> 401; checkpoint on a finalized SOW -> 409 `sow_locked`; checkpoint saves and returns a new version/revision, calling `save_sow_draft` with the correct `expected_version`/`expected_revision`; HTML is sanitized (script tag and `onclick` stripped) while a highlight class (`marker-yellow`) and a placeholder marker (`sow-studio-placeholder-marker`) survive; checkpoint with unchanged HTML skips the write entirely; `resolve_access` matches a real/simulated `assert_deal_access` for an owner, a collaborator, a stranger, and `FE_ADMIN` (the `FE_ADMIN` case specifically confirms `role == "sow_admin"` and the `finalize` capability). Confirmed `test_sow_http_e2e.py`, `test_native_collaboration.py`, `test_sow_revision_integrity.py`, `test_sow_rich_text.py` all still pass unchanged (48/48). Full suite: 555 passed, the same 9 pre-existing failures as the established baseline -- no regression. `run_checks.sh` extended; full 7/7 checks green, 115 backend tests.
Decisions/assumptions:
- `ResolveAccessResponse`'s `locked` and `document-state`'s `locked` are both `status == "Finalized"` only (not also "pending sign-offs", which the raw CP-20 prompt text mentions but `bridge.ts`'s typed `DocumentStateResponse`/`ResolveAccessResponse` don't carry a separate field for) -- followed the vendored shared type as authoritative over the prose task description, consistent with the same choice made in CP-19.
- Guest actors (`ActorRef.kind == "guest"`) are rejected from `resolve_access` with `reason: "feature_disabled"` -- guest sessions are explicitly deferred to CP-60; nothing here builds guest support.
Registry (new or changed):
- file: src/sow_module/collab_internal.py: `ResolveAccessRequest`/`resolve_access`, `DocumentStateRequest`/`get_document_state`, `CheckpointRequest`/`checkpoint`, `notify_access_changed`, `SERVICE_KEY_HEADER`
- file: src/sow_module/router_internal.py: `verify_collab_service_key`; endpoints: `POST /internal/sow-collab/resolve-access`, `POST /internal/sow-collab/document-state`, `POST /internal/sow-collab/checkpoint` (all `include_in_schema=False`)
- file: src/main.py: registered `sow_module_internal_router` at root (no prefix)
- file: src/sow_studio/native_collaboration.py: `revoke_guest_invitation` now calls `notify_access_changed(f"sow-{deal_id}", "guest_revoked")` after its existing commit; return value unchanged
- file: tests/test_sow_module_collab_internal.py: 11 tests
- file: sow_module/run_checks.sh: added the new test file to the backend pytest line
Open questions for the user: none new.
Handoff to next prompt: CP-21 -- Node Express skeleton, ticket verification, CI-style checks.

## CP-21: Node service skeleton, ticket verification, CI-style checks (2026-11-06)
Summary: Built the collab service shell: `src/server.ts` (Express `/healthz`, RFC 9457 error handler via `http-problem.ts`, a `ws` server on `/collab` sharing the HTTP server, graceful shutdown on SIGTERM/SIGINT), `src/ticket.ts` (HS256 ticket verification matching FastAPI's `jose.jwt.encode` output exactly — signature, `aud`, `exp`, plus a per-process `ReplayGuard` for single-use `jti`), `src/bridge/fastapi-client.ts` (typed calls to the 3 internal endpoints using `shared/bridge.ts`'s types, per-call timeout via `AbortController`, and a small failure-count circuit breaker), and `src/config.ts` (env loader matching `.env.example`'s existing names). Origin is checked at the HTTP `upgrade` event before the WebSocket handshake completes; an unlisted origin never reaches `wss`. On connect, the server waits for one `HELLO` message (5s timeout) and closes 4401 on a missing/malformed/expired/replayed/mis-signed ticket — no further protocol logic yet (that's CP-24).
Added CI-style checks since neither repo has an automated test-running pipeline: `scripts/check-module-boundary.sh` (greps for the collab service importing shared code from outside its own vendored `src/shared`, and for the frontend's `src/sow_module` importing collab code), wired into `run_checks.sh` as `collab module boundary`; `sentinel-secret.test.ts` (plants a fake secret as the ticket secret/service key, exercises the rejection and FastAPI-client-failure paths, asserts the value never appears in captured log output or a thrown error's message).
Tests: `ticket.test.ts` (7 — valid, tampered, wrong secret, expired, replayed, malformed, wrong audience), `fastapi-client.test.ts` (4 — success, non-2xx, timeout, circuit breaker opens after repeated failures and short-circuits without calling the server again), `server.test.ts` (5 — healthz, valid ticket keeps the socket open, tampered/expired ticket closes 4401, disallowed origin rejected before upgrade). All pass; `run_checks.sh` 8/8 (added the boundary check as a new row).
Decisions/assumptions: the module-boundary check is a grep-based script, not a bundler/lint-plugin rule — matches the project's existing shell-script check style (`sync_shared.sh --check`) rather than adding new tooling.
Registry (new or changed):
- file: sow_module/collab/src/server.ts: `createCollabService`, `/healthz`, WS upgrade/HELLO handling
- file: sow_module/collab/src/ticket.ts: `verifyTicket`, `ReplayGuard`, `TicketError`
- file: sow_module/collab/src/bridge/fastapi-client.ts: `FastApiClient`, `FastApiError`, `CircuitOpenError`
- file: sow_module/collab/src/config.ts, src/http-problem.ts
- file: sow_module/collab/scripts/check-module-boundary.sh
- file: sow_module/run_checks.sh: added `collab module boundary`
Open questions for the user: none new.
Handoff to next prompt: CP-22 — Document Owner rooms, lease and persist queue (Phase D begins).

## CP-22: Document Owner: rooms, lease and persist queue (2026-11-06)
Summary: Turned the CP-09/CP-10/CP-11 spikes into production code. `src/owner/room.ts` (`Room`: one in-memory `Y.Doc` per open SOW, tracks connected client ids, `applyUpdate`/`encodeStateAsUpdate`), `src/owner/room-manager.ts` (`RoomManager`: creates a room on first access, `closeIfEmpty` removes it once its last client leaves), `src/owner/lease-manager.ts` (`LeaseManager`: thin wrapper over the existing `DataStore.acquireLease`/`renewLeases`/`releaseLease` — `renewAll()` is already a single batched call across every lease this process holds, since `DataStore.renewLeases` is keyed by `holderId`, not per document), `src/owner/persist-queue.ts` (`PersistQueue`: buffers updates per document, flushes on a timer via `DataStore.appendUpdates`, assigns a per-document monotonic `seq`, calls `onDurable(dealId, upToSeq)` after a committed flush, and `onFenced(dealId)` — dropping that document's pending rows — when `appendUpdates` throws `StaleFencingError`).
Re-proved the two acceptance checks against this production code (not the spikes), using `InMemoryDataStore` rather than live Snowflake (fast, and the store's `StaleFencingError` contract is identical either way): a durable-acknowledged edit survives a simulated crash (flush, then read straight from the store as a fresh process would); a paused "zombie" owner holding a stale fencing token has its write rejected and dropped (`onFenced` fires, nothing lands in the store) once a newer owner's lease has taken over.
Tests: `owner/durability.test.ts` (2 — the two acceptance checks above), `owner/room-manager.test.ts` (3 — create/reuse, close-when-empty, applying a real external Yjs update). All pass; collab suite now 34 passed, 6 skipped (unchanged live-Snowflake integration tests).
Decisions/assumptions: `PersistQueue.flush()` makes one `appendUpdates` call per dirty document per tick (the existing `DataStore` contract validates one fencing token per call), not a single call spanning multiple documents — matches the interface built in CP-18 without changing it. A document with no fencing token yet (lease not acquired) is left buffered rather than dropped, since CP-23 (seeding) and CP-24 (protocol wiring) haven't connected a room to a lease yet.
Registry (new or changed):
- file: sow_module/collab/src/owner/room.ts: `Room`
- file: sow_module/collab/src/owner/room-manager.ts: `RoomManager`
- file: sow_module/collab/src/owner/lease-manager.ts: `LeaseManager`
- file: sow_module/collab/src/owner/persist-queue.ts: `PersistQueue`
Open questions for the user: none new.
Handoff to next prompt: CP-23 — snapshots, reopening and seeding from the latest draft.

## CP-23: Document Owner: snapshots, reopening and seeding (2026-11-06)
Summary: `src/owner/snapshot-manager.ts` (`SnapshotManager`): `takeSnapshot` chunks `Y.encodeStateAsUpdate(doc)` at a configurable byte size (default 256KB) via `DataStore.putSnapshot`, storing `contentHtml`/`contentHash` on chunk 0 only; `reopen` loads the latest snapshot's chunks (ordered, reassembled), applies them to a fresh `Y.Doc`, then applies every update after the snapshot's `upToSeq` from `DataStore.readUpdates`. `src/owner/seed.ts` (`SeedCoordinator`): on a document's first open (`store.getDocument` returns null), fetches the latest draft via FastAPI's `document-state` and records the sanitized HTML plus source version/revision on the Yjs doc under a `seed` map, then writes a `SOW_COLLAB_DOCUMENT` row — never touching `SOWS`/`SOW_DRAFT_VERSIONS` (document-state is a read). Concurrent first-opens for the same document are deduped by an in-process `Map<dealId, Promise>`, which is the real serialization point: only one owner process holds a document's lease at a time, so two simultaneous opens of a not-yet-live document are, in practice, always two WebSocket connections racing on the same process.
**Explicitly flagged scope decision**: seeding stores the draft HTML as a marker, not a pre-built Lexical node tree. Converting HTML into Lexical's Yjs-bound shared-type structure is owned by the frontend's shared code (`blend_ode/src/sow_module/shared`, per ADR-008/009) and needs `@lexical/headless`/jsdom, neither of which is a dependency of this Node service (and adding them here would duplicate logic this module is specifically supposed to vendor, not reimplement). The first editor-capable client to open a freshly-seeded document performs the HTML->Lexical conversion locally and applies it as a normal Yjs update, which becomes the document's real update history from that point on. Flagged for CP-25 to confirm against the actual shared-type shape the browser binds to.
Tests: `snapshot-manager.test.ts` (3 — reopen reconstructs snapshot+tail correctly, a large state splits into multiple chunks and reassembles byte-correct, reopening a document with no snapshot/updates returns null), `seed.test.ts` (3 — seeds a new document from the latest draft, two concurrent first-opens call FastAPI's document-state exactly once, seeding a document that already has a `SOW_COLLAB_DOCUMENT` row calls FastAPI zero times and leaves the existing row and the Yjs doc's seed map untouched). All pass; collab suite now 40 passed, 6 skipped. Full `run_checks.sh` 8/8.
Registry (new or changed):
- file: sow_module/collab/src/owner/snapshot-manager.ts: `SnapshotManager`
- file: sow_module/collab/src/owner/seed.ts: `SeedCoordinator`, `DocumentStateFetcher`
Open questions for the user: none new (the seeding-shape flag above is for CP-25, not a blocking question now).
Handoff to next prompt: CP-24 — WebSocket protocol, roles and validation boundary.

## CP-24: WebSocket protocol, roles and validation boundary (2026-11-06)
Summary: Wired the full WebSocket protocol together. `src/owner/connection-registry.ts` (`ConnectionRegistry`): tracks live connections per document for broadcast fan-out, kept separate from `Room`'s own client-count bookkeeping so `Room` stays testable without a real socket. `src/owner/connection.ts` (`handleAuthenticatedConnection`): on a verified `HELLO`, calls `resolve-access` (confirms role/capabilities server-side — the ticket's role is only a hint, per contracts.md), closes 4403 on denial, 4410 if the SOW is locked; on first client, acquires the lease (4301 with the current holder's address if another owner has it), reopens from the latest snapshot or seeds from the draft; sends `READY` and an initial `SYNC` frame; routes `AWARENESS` frames straight through, routes `SYNC` frames through `canEdit()` (closes 4403 for a viewer/commenter that sends one) then the validation boundary before applying/enqueueing/broadcasting; handles `REAUTH` (re-verifies the ticket, re-runs `resolve-access`, sends `ROLE_CHANGED` if the role actually changed) and `CHECKPOINT_NOW` (see the flagged gap below); releases the lease when the room empties.
`src/owner/validation.ts` (`validateUpdate`): applies the candidate update to a disposable shadow copy of the room's document (never mutates the live one), then walks it using `shared/allowlist.ts`'s own `isTagAllowed`/`isClassAllowed`/`isStylePropertyAllowed`/`isAttributeAllowed` — the same module the frontend and FastAPI both use, per the "one allowlist module only" constraint — rejecting an unregistered tag, a disallowed class/style/attribute, nesting past `MAX_NODE_DEPTH` (64), or a payload over `MAX_PAYLOAD_BYTES` (1MB, also set as `ws`'s own `maxPayload`). **A real bug found and fixed while building this**: a Yjs root type received only via a binary update materializes as a generic, untyped `AbstractType` until something re-requests it with its real constructor — a naive `instanceof Y.XmlElement`/`Y.XmlFragment` walk over `doc.share`'s raw entries silently matched nothing and let every update through. Fixed by re-requesting each root key as `Y.XmlFragment` (catching the thrown mismatch for any root that genuinely isn't one) before walking it. Confirmed live with a minimal reproduction script before writing the fix into the real code.
**Dependency change, needed to make this possible**: `shared/allowlist.ts` imports `lexical` (via its `SowHighlightNode.ts` dependency, which also needs `@lexical/mark`; `allowlist.ts` itself separately touches `@lexical/headless`/`@lexical/html`/`@lexical/list`/`@lexical/table` through sibling files in the same vendored folder). None of these were installed in the collab service before now. Added `lexical`, `@lexical/headless`, `@lexical/html`, `@lexical/list`, `@lexical/mark`, `@lexical/table`, all pinned to `0.44.0` to match `blend_ode`'s versions exactly. `tsconfig.json` updated: added `DOM` to `lib` (types only — nothing here runs in a browser), set `noImplicitOverride: false` (the vendored `SowHighlightNode.ts` predates that flag and must compile as delivered, never hand-edited), and removed `allowlist.ts`/`SowHighlightNode.ts` from the exclude list (its old comment — "the Node service never imports them" — is now out of date and was corrected). `htmlFidelity.ts`/`intake.ts` stay excluded; nothing here needs HTML<->Lexical conversion or paste/LLM intake normalization, only the allowlist's plain predicates. Incidental finding, not fixed (out of scope): `npm audit` flags `happy-dom` (transitive via `@lexical/headless`, high severity) — already a known, tracked gap from CP-12/CP-37 (`sow_docs/guides/testing_pending_guide.md`); this code never calls `@lexical/headless`'s `withDOM()` helper that touches it, so the vulnerable path is dormant, not exercised.
**Explicitly flagged scope decision, carried over from CP-23's seeding note**: the validator's tag/class walk assumes the live document's Lexical content lives under a `Y.XmlFragment`-shaped root. This was not verified against real traffic — the browser-side `@lexical/yjs` binding doesn't exist yet (CP-25). If the real binding uses a different shared-type shape (e.g. a `Y.Map`-keyed structure), the tag/class/attribute checks will find nothing to walk and pass trivially; size and malformed-frame checks still apply regardless since they don't depend on the shape. Revisit once CP-25 produces real traffic to test this against.
**`CHECKPOINT_NOW` gap, explicitly flagged, not hidden**: responds with a `service_unavailable` error — exporting HTML from the live Yjs document requires the same Lexical<->Yjs conversion CP-23's seeding note already flagged as owned by the frontend, not available server-side. Checkpointing will be wired once that conversion exists.
Tests (CP-24's three required integration tests, with real Yjs clients, plus supporting unit tests): `server.test.ts` adds — a malicious update (an XmlElement tagged `script`) is rejected at the boundary and never reaches a second connected client; a view-mode (viewer-role) client that sends a content update is closed with 4403; `REAUTH` with a role change sends `ROLE_CHANGED` on a live socket. `validation.test.ts` (8 — allowed tag, unregistered tag, disallowed class, allowed class, depth bomb, oversized payload, malformed/undecodable update, confirms the original document is never mutated). All pass; collab suite now 51 passed, 6 skipped (live-Snowflake integration, unaffected). Full `run_checks.sh` 8/8.
Registry (new or changed):
- file: sow_module/collab/src/owner/validation.ts: `validateUpdate`, `MAX_PAYLOAD_BYTES`, `MAX_NODE_DEPTH`
- file: sow_module/collab/src/owner/connection.ts: `handleAuthenticatedConnection`, `broadcastDurable`, `dealIdFromSowId`
- file: sow_module/collab/src/owner/connection-registry.ts: `ConnectionRegistry`
- file: sow_module/collab/src/server.ts: full dependency wiring (store, FastAPI client, rooms, leases, persist queue, snapshots, seeds, registry) behind the existing HELLO-verification flow
- file: sow_module/collab/src/bridge/fastapi-client.ts: added `FastApiClientLike` interface so tests can inject a fake without a real HTTP server
- dependency: sow_module/collab/package.json: added `lexical`, `@lexical/headless`, `@lexical/html`, `@lexical/list`, `@lexical/mark`, `@lexical/table` @ 0.44.0
- file: sow_module/collab/tsconfig.json: DOM lib, `noImplicitOverride: false`, `allowlist.ts`/`SowHighlightNode.ts` no longer excluded
Open questions for the user: none new. The shared-type-shape assumption and the `CHECKPOINT_NOW` gap are flagged for CP-25, not blocking questions now.
Handoff to next prompt: CP-25 — LexicalDraftEditor shell with the same props as DraftEditor (Phase D's frontend half begins; this closes out the user's requested CP-21 through CP-24 stopping point).

## CP-25: LexicalDraftEditor shell with the same props as DraftEditor (2026-11-06)
Summary: Built the frontend editor shell in `blend_ode/src/sow_module/editor/`, not wired into any route. `LexicalDraftEditor.tsx` (default export) takes props typed as `React.ComponentProps<typeof DraftEditor>`, so parity with `DraftEditor` is enforced by the compiler, not by hand. Renders paragraph and heading nodes only, through Lexical's `CollaborationPlugin`, with the undo history from `@lexical/yjs` (no `HistoryPlugin`), an error boundary, a loading state and a Retry state. `SowCollabProvider.ts` implements Lexical's `Provider` contract over one WebSocket, speaking the Node service's wire format exactly as CP-24 built it (HELLO with the ticket, binary `[kind][payload]` frames, READY/DURABLE/ROLE_CHANGED/REAUTH_REQUIRED/ERROR). `useCollabSession.ts` requests the session through the existing `apiToken` axios instance; the ticket lives in React state only (verified by a test that nothing is written to localStorage, sessionStorage or the URL). `lazy.tsx` is the only import path for screens, so Lexical is code-split. `SowLexicalHarness.tsx` renders the editor for `?sow=sow-<id>`; it is not mounted by any route.
**Blocked-draft safety**: a draft containing anything the shell cannot represent (tables, lists, highlights, font styling, links) is NOT opened in an editor. It shows a notice naming the unsupported tags. An earlier draft of this code rendered an empty read-only editor next to that notice, which looks exactly like a blank SOW. Fixed, and a test asserts the editor is absent.
**Local persistence**: `y-indexeddb` stores the document locally. The provider holds all outbound updates until local storage has loaded, so the stored state is not re-sent to the server on every refresh. After the server's initial sync it sends the full local state once, so edits made offline reach the server (the server applies updates idempotently).
Verified (automated, not manual):
- `sow_module/editor` tests: 28 pass across 4 suites. Provider: HELLO with the ticket, server updates applied without echo, local edits sent as SYNC frames, view-mode never sends content, ROLE_CHANGED stops sending, DURABLE clears the unacknowledged count, outbound held until local storage loads, malformed server frames reported without throwing, disconnect, and edits surviving a refresh (real y-indexeddb against fake-indexeddb). Hook: session requested for the SOW, ticket kept out of storage and the URL, 403 mapped to a clear message, reload. Component: loading, error with Retry, unsupported-draft notice, missing SOW, session refused, live editor editable, read-only locks it, Save disabled until a change. Harness: prompts for a SOW, and opens the editor for one.
- Existing `DraftEditor.test.tsx`: 22/22 pass, unchanged from baseline.
- Existing `DraftEditorScreen.integration.test.tsx`: 60 fail / 3 pass, identical to the pre-change baseline. It was already failing before this checkpoint (jsdom TypeError "Failed to execute 'contains' on 'Node'"); not caused or fixed here.
- Typecheck of the whole frontend: clean. ESLint on the new folder: 0 errors, 0 warnings.
- Production build (`CI=false npm run build`): succeeds. Main chunk 565 kB gzipped. No Lexical or collaboration code appears in any output file, because nothing imports the editor yet. Bundle delta for the app: 0 kB by construction. The lazy split only takes effect once the harness or a screen mounts the editor.
NOT verified (say so when demoing): two real browsers typing together — requires the Node service running and a real FastAPI session; not run here. Real-browser rendering and the CKEditor-era flows — not run here.
Deviations from the CP-25 text, all flagged:
1. Session request is POST `/pricing_tool/sows/{sow_id}/collab/session`, not GET. The CP-19 endpoint is POST (contracts.md section 1).
2. SYNC frames carry raw Yjs update bytes, not y-protocols sync envelopes. The CP-24 server treats SYNC payloads as raw updates; changing that is a server change. Awareness does use y-protocols encoding.
3. "Keep local updates until DURABLE" is only approximated. DURABLE carries an upToSeq but not which client updates it covers, so the client clears its whole unacknowledged count on DURABLE. A frame sent just before DURABLE arrives can be counted as durable early. Exact per-update acknowledgement needs an echoed client update id in the protocol (a proposed change, not made).
4. CP-24's validator assumes the document's content lives under a Y.XmlFragment root. That shape is still unverified against real Lexical traffic; this checkpoint's own tests do not exercise it end to end.
5. Comments (`onCommentChangeHandlerReady`, `comments`) are accepted for prop parity but not implemented; the Save button passes `null` for comments.
6. Save calls the existing `onSave` (the current save path), not a socket checkpoint. CHECKPOINT_NOW is still unavailable (it needs HTML export from the live document through the same shape flagged in 4).
Files changed in an existing location (disclosed, not hidden): `blend_ode/package.json` (added `y-protocols` 1.0.7, `y-indexeddb` 9.0.12 as runtime deps; `fake-indexeddb` 6.2.7 as dev dep; added a `jest` block with a `moduleNameMapper` for one `@lexical/react` sub-path that Jest 27 cannot resolve, and a `transformIgnorePatterns` that lets Jest load the ESM-only `lib0`/`yjs`/`y-protocols`/`y-indexeddb` packages); `blend_ode/package-lock.json` (53 added lines, no removed lines). No application file was modified.
Open questions for the user: (a) the harness needs a route to be seen in a browser, and mounting it means one line in `App.tsx`, which this checkpoint's own constraints forbid; decide whether to allow it for the demo. (b) confirm POST vs GET for the session endpoint is intended (the code uses POST, matching CP-19).
Manual two-browser note: NOT performed (not run in this environment).
Handoff to next prompt: CP-26 — toolbar framework and basic formatting.

## Follow-up fixes and demo routes (2026-11-07 to 2026-11-09)
Summary: After CP-25, fixed the editor import (`LexicalErrorBoundary` is a named export; commit 485d3cc), made the collab service start on Windows (`main()` guard compared file URLs incorrectly; blend_ode_be 166120c), and stopped the pricing pages crashing when a deal has no project start date (both pages now return right after the fatal-error redirect; blend_ode 8853099). Added temporary demo routes in `blend_ode/src/App.tsx` (`/sow-sandbox`, `/sow-lexical-harness`), marked "TEMP demo route, do not commit", and not committed. The sandbox is a local Lexical page with no login, backend or SOW data: Undo/Redo, Normal/H1-H3, Bold/Italic/Underline/Strikethrough/Code, bullet and numbered lists, and Print (A4). Sandbox changes are not committed.
Incident: during the session, `blend_ode/src/App.tsx` in the working tree was cut to a single line by something outside this session. The damaged copy was saved, the file was restored from git, and only the two temporary routes were re-added. Not committed.
Verification: typecheck, ESLint on the editor folder, and production build pass. Editor tests: 28 pass. The sandbox and print output were not checked in a browser.
Open items: the live harness needs the backend running and the user's account to have `FE_SOW_LEXICAL_EDITOR` (or `FE_ADMIN`) and access to the deal. The 403 for deal 64986161471 is a real access denial. `/pricing_tool/hubspot_deals_sync_info` returns 500, which hides the CORS headers. The backend traceback is still needed to fix it.
Handoff to next prompt: CP-26 — toolbar framework and basic formatting in the live editor.

## CP-26 to CP-28: toolbar, formatting parity, list depth (2026-11-09)
- blend_ode `cedf2a1` CP-26: toolbar (bold, italic, underline, strikethrough, H1-H3, alignment, undo/redo), capability-gated.
- blend_ode `c1513aa` CP-27: highlight, font size, font family from html-contract.json. Import now keeps span font-size and font-family. blend_ode_be `e29ecf7` re-vendors shared; synthetic-15/16 expected JSON updated (style "" to the real style). Exported HTML fixtures unchanged.
- blend_ode `70d5fdb` CP-28: list buttons, Tab cap at depth 3, paste flattened to remaining depth, import flatten in allowlist.ts. blend_ode_be `1b2d13d` re-vendors shared.
- Verified: frontend 108 tests pass (sow_module), backend sow_module 115 pass (before CP-28 sync; no Python change in CP-28), typecheck clean, build passes.
- Not done: server-side list depth in `html_allowlist.py` (streaming sanitizer; needs a tree-based lift to match TS). CP-29 (tables) and CP-30 (merges, properties, export parity) not started.

## CP-29 and CP-30: tables, merges, cell properties (2026-11-09)
- blend_ode `4db5c7a`: table insert (size picker), row and column insert and delete (delete asks to confirm), header row, column width presets, merge and split, cell shading, vertical alignment, Tab between cells. Stored on the nodes, so they go through Yjs and HTML export.
- Allowlist bug fixed: the contract group "table-box" was never expanded, so widths, shading and borders were stripped on export. Fixed in TS and Python (blend_ode_be `bce5042`, plus shared re-vendor). Lexical's default cell width, border, vertical-align and header background are stripped on import, because the exporter writes them back on every cell.
- Verified: frontend 116 sow_module tests pass, backend 116 sow_module tests pass, typecheck and build clean.
- Not done: drag-to-resize handles (width presets instead), table border and padding dialogs, python-docx and reportlab support for merged cells (export parity), a two-client sync test for widths, real-browser check of the table UI, and server-side list depth in `html_allowlist.py`.

## Table visibility fix, CP-31 to CP-33 (2026-11-09)
- Table fix: empty cells had no size or border, so the table looked absent and could not be clicked. Added table and list styles in `sowx.css`. Insert works from an unfocused editor (places the table at the end). The table menu stays visible outside a table, with a hint. Checked in headless Chrome: insert, type into a cell, menu, find panel.
- CP-31: paste and drop run through `intake`. Ctrl+Shift+V pastes plain text. Removals show an antd message; details for sow_admin only. One paste is one update (`tag: paste`, discrete). Server re-check: existing `validation.test.ts` covers disallowed tags and classes; no new server test for paste.
- CP-32: placeholder rule (keep while either bracket remains, unwrap otherwise) as Lexical transforms, incl. table cells. Find and replace panel; replace all is one update. Read-only roles can find, not replace. Limit: matches stay inside one text run.
- CP-33: block ids in node state (`createState`), exported as `data-block-id` on top-level blocks (paragraph, heading, list, table), read back on import. Paste strips copied ids. Decision to record for the user: ids are in exported HTML, because checkpoint reopen needs them. Property tests with fast-check: 40 random runs.
- Not verified: two-client Yjs sync of block ids (Lexical syncs node state; not run here). Browser check covers table, find, and paste only.

## CP-34 to CP-36: page chrome, action bar, editor API, add-ins (2026-11-09)
- CP-34: watermark (sign-offs pending), Blend logo, provisional banner (same wording), read-only label. Historical version = static render of sanitized HTML, no room opened. Watermark and banner sit beside the editor, never in saved HTML (test checks this). Limit: the watermark is not clipped by a resize observer as in DraftEditor; it covers the editor box.
- CP-35: Save, Download (DOCX, PDF), Copy link with DraftEditor's labels. Status: Saving until the server confirms, then Saved. Leave-page warning only while changes are unconfirmed by the server (pure rule tested; the event itself not run in jsdom).
- CP-36: editor API (getHtml, setHtml, insertContent with cursor/end/afterBlock/replaceBlock, focus, scrollToBlock). Programmatic changes use their own tag and never fire onContentChange. Add-ins call the existing services unchanged through `addIns.ts`. The add-in marker (data-add-in-id) is carried on blocks and list items. Existing add-in tests: 64 pass.
- Verified: frontend sow_module tests 163 pass, typecheck and build clean.
- Not done or not verified: the live harness in a real browser (needs backend and session); two-client sync; scroll-to-add-in visuals; the readOnly server refusal (backend already gates viewers; not re-tested here).

## CP-37: Owner checkpoint to the draft system (2026-11-09)
Summary: The Owner exports the live document to HTML from the Lexical v1 Yjs shape (verified with a spike), passes it through the shared allowlist, and calls FastAPI checkpoint. Saves on explicit request, after a quiet period, and when the last client leaves. Also fixed the CP-24 validation boundary, which never checked real Lexical content (it walked a fragment root).
Decisions/assumptions: Base revision is read from document-state immediately before each send. On 409 or 428, refresh and retry once; if the revision moved, do not retry (see CP-38). Unknown block types fail closed. Add @lexical/yjs and @lexical/rich-text 0.44.0 and happy-dom 20.14.6 to the collab service (user asked for "what is good").
Registry (new or changed):
- file: sow_module/collab/src/owner/yjs-html.ts: Yjs-to-HTML export (Lexical v1 shape) and the shared-allowlist sanitizer
- class / function: sanitizedRootHtml, UnknownBlockError in yjs-html.ts: export and fail-closed error
- class / function: runCheckpoint in sow_module/collab/src/owner/checkpoint.ts: one checkpoint with refresh and retry
- class / function: CheckpointCoordinator in sow_module/collab/src/owner/checkpoint-coordinator.ts: timer, explicit, last-client-leave; one run at a time
- class / function: validateUpdate (walkText) in sow_module/collab/src/owner/validation.ts: walks the Lexical root
- endpoint: none new (uses the existing internal checkpoint)
- feature: CHECKPOINT_NOW and automatic checkpoints: built (Node side)
Open questions for the user: none for CP-37.
Handoff to next prompt: CP-38 screen integration.

## CP-38: Screen integration behind the flag and other writers (2026-11-09)
Summary: DraftEditorScreen shows the Lexical editor for users with FE_SOW_LEXICAL_EDITOR (or FE_ADMIN with REACT_APP_SOW_LEXICAL_EDITOR=true). In that mode Save and "Save and continue" go to the Owner checkpoint, the screen's autosave is off, and the screen takes the new version and revision from the answer. If another path changed the draft, the Owner refuses to overwrite and tells every client.
Decisions/assumptions: The Owner's known revision comes from the collab row (persisted after each checkpoint). Explicit saves get a CHECKPOINTED reply even when nothing changed. Version label is "vN", matching crud.py. Admin access depends on a build flag.
Registry (new or changed):
- file: sow_module/editor/flag.ts: isLexicalEditorEnabled
- file: blend_ode/src/pricing-tool/components/DraftEditorScreen.tsx: changed lines: imports; after draftRevisionRef (flag, API ref, DraftEditorView); top of handleSave (checkpoint path); autosave condition; DraftEditor opening tag now DraftEditorView with onReady
- function: requestCheckpoint in SowCollabProvider.ts: CHECKPOINT_NOW, resolved by CHECKPOINTED
- function: checkpoint in editorApi.ts (SowEditorApi)
- feature: flagged Lexical editor in the screen: built
Open questions for the user: (a) "reload from document-state" after an outside change is a notice, not an automatic reload. A reset of the in-memory room needs a design (see Phase D report). (b) Confirm the flag design: FE_ADMIN needs REACT_APP_SOW_LEXICAL_EDITOR=true in the build.
Handoff to next prompt: CP-39 freeze and regression.

## CP-39: Core API freeze and Phase D regression (2026-11-09)
Summary: Wrote the frozen API and the Phase D regression report with a two-user script. Automated suites: no new failures. The two-user run itself has not been done.
Decisions/assumptions: Pre-existing failures are listed separately (see phaseD_report.md).
Registry (new or changed):
- file: sow_docs/tracking/core_api_freeze.md: frozen routes, WebSocket messages, props, close codes (Core API 1.0)
- file: sow_docs/tracking/phaseD_report.md: regression results and manual test script
Open questions for the user: run the two-user script and record results in phaseD_report.md.
Handoff to next prompt: CP-40 presence and chunked sync.
