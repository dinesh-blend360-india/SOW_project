# Current Project State

Last updated: 2026-11-06, by the session that created this handoff system. Keep this file current
as development progresses — it should always reflect reality, not a plan.

## Architecture status

- **Pricing Tool and SOW Studio** (the pre-existing, CKEditor-era application): built and in
  production use. Not modified by the work described below; this handoff system documents it but
  did not change its behavior.
- **SOW collaboration module** (Lexical + Yjs live-editing, replacing CKEditor for SOW authoring):
  **in active development**, checkpoints CP-00 through CP-20 of its own build plan
  (`sow_docs/build_plan/SOW_EDITOR_BUILD_PROMPTS_v1.md`) are complete.

## Completed functionality (SOW collaboration module)

- Shared TypeScript contracts (roles/capabilities, protocol, HTML allowlist, bridge types) —
  `blend_ode/src/sow_module/shared/`, vendored into `blend_ode_be` with a drift check.
- Backend HTML allowlist (`html_allowlist.py`) with parity tests against the TypeScript version.
- Snowflake schema for the module's own tables (`SOW_COLLAB_DOCUMENT/UPDATE/SNAPSHOT/LEASE/
  REVOCATION/IDEMPOTENCY/AUDIT`), written as `migrations/011` and `012` (grants) — **not yet
  applied to Snowflake**; written and tested, awaiting explicit approval to run.
- `SnowflakeDataStore`: the Node service's full data-access layer against those tables, live-tested
  against real (temporary) Snowflake tables.
- FastAPI session endpoint (`POST /sows/{sow_id}/collab/session`) — feature-flagged, mints a
  30-second ticket.
- FastAPI internal endpoints (`resolve-access`, `document-state`, `checkpoint`) — service-key
  protected, reusing the existing `save_sow_draft`/`assert_deal_access` without modifying them.
- A Python mirror of the role/capability table, parity-tested against the TypeScript source.

## In-progress / not yet started (SOW collaboration module)

- The Node Express + `ws` server itself that actually hosts live WebSocket connections (CP-21+ in
  the build plan) — not built.
- Ticket verification on the Node side, and the WebSocket lifecycle described in
  `sow_docs/tracking/contracts.md` section 5 (HELLO handshake, lease ownership check, Yjs sync
  frames, awareness, reauth, checkpoint-on-last-client-leaves) — not built.
- The browser-side Lexical editor component and its WebSocket client — not built; no route in
  `blend_ode` renders a Lexical editor yet.
- Guest/external access to the live-collaboration module (commenter role) — explicitly deferred to
  CP-60 of the module's build plan; not started.
- A two-browser live-editing proof (the module's own first real UI milestone, CP-25) — blocked on
  the above.
- Independent re-verification of the checkpoint fencing token against the live lease from FastAPI —
  explicitly out of scope for now (see `KNOWN_ISSUES_AND_BLOCKERS.md` #7).

## Remaining tasks

See `TASK_BOARD.md` for the itemized, status-tracked list.

## Recent changes (most recent first — see `CHANGELOG.md` for the full history)

1. This `agent-handoff/` documentation system was created at the project root (2026-11-06).
2. The SOW module's tracking/reference/narrative docs were moved from inside `blend_ode_be` (and a
   standalone `documentation/` folder) into a new root-level `sow_docs/` folder, with pointers left
   at each original location — **this move is currently staged but not committed** in
   `blend_ode_be`; see `SESSION_LOG.md` for the exact uncommitted state and the open question about
   whether to commit it as-is or move `sow_docs/tracking/` back inside `blend_ode_be` so it's
   versioned with the code it documents.
3. CP-20 (FastAPI internal endpoints) committed in `blend_ode_be` (`408b0f8`).
4. CP-19 (FastAPI session endpoint, ticket, feature flag) committed (`dc0218b`).
5. CP-18 (Node Snowflake data layer) committed (`427f479`) — found and fixed a real binary-bind bug
   via a live integration test.

## Important implementation details worth knowing before continuing

- The SOW module's Node service connects to Snowflake with **different credentials and a different
  schema** than the main FastAPI app — do not assume one `.env` covers both.
- `checkpoint()`'s HTML sanitization must always go through `sow_module.html_allowlist
  .sanitize_draft_html`, never the pre-existing `sanitize_rich_text` — the latter strips
  highlights/placeholders the editor needs to preserve.
- The role-to-capability mapping is parity-tested across TypeScript and Python — if you change one
  side, the drift test will fail until you change the other (or re-run `sync_shared.sh`).
- Nothing in the SOW module writes to `SOWS`, `SOW_DRAFT_VERSIONS`, `HUBSPOT_DEALS`, or
  `SOW_NATIVE_*` except through the existing `save_sow_draft` call inside the `checkpoint` internal
  endpoint — this is a hard rule, not a current-state observation.

## Known limitations

See `KNOWN_ISSUES_AND_BLOCKERS.md` for the full, itemized list — most notably the committed secrets
in `blend_ode` (#1) and the lack of CI in either repo (#2), both of which predate and are unrelated
to the SOW module work but are relevant to anyone continuing development here.

## Update 2026-11-09
- **CP-21 to CP-25 are complete.** The collab service shell, rooms, snapshots, WebSocket protocol, and the frontend editor shell are committed.
- **The live editor is not reachable for a demo yet.** It needs the backend, the collab service, and a user with `FE_SOW_LEXICAL_EDITOR` (or `FE_ADMIN`) and access to the deal.
- **A local sandbox page exists** (`/sow-sandbox`, temporary, uncommitted) with formatting, lists, and A4 print.
- **Open:** backend 500 on `/pricing_tool/hubspot_deals_sync_info`; the CORS message is a side effect of that 500.
- **Next:** CP-26 (toolbar in the live editor).
