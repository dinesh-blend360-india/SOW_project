# Runbook: live SOW editor (local and staging)

Status: written 2026-11-13 from the code. Not yet rehearsed end to end. Items marked (verify) must be checked once on a real environment.

## Services
- **FastAPI** (`blend_ode_be`): `uvicorn` on the usual port. Serves the session endpoints, the internal Owner endpoints and the native editor.
- **Owner / collab service** (`blend_ode_be/sow_module/collab`): `npm run dev` (watch) or `npm start` after `npm run build` (verify the build script). Serves WebSocket on `/collab` and `/healthz`.
- **Frontend** (`blend_ode`): `npm start` for development. The live editor needs `REACT_APP_SOW_LEXICAL_EDITOR=true` (admins only) or the user flag `FE_SOW_LEXICAL_EDITOR`.
- Stop: stop the Owner first (it saves open rooms on shutdown, within `COLLAB_SHUTDOWN_TIMEOUT_MS`), then FastAPI, then the frontend.

## Environment variables
From `blend_ode_be/sow_module/collab/.env.example` and `blend_ode_be/src/common/config.py`. Values are secrets unless stated.
- `COLLAB_TICKET_SECRET` (secret): signs the 30-second WebSocket tickets. Must match between FastAPI and the Owner.
- `COLLAB_SERVICE_KEY_API_TO_NODE` and `COLLAB_SERVICE_KEY_NODE_TO_API` (secret): service keys for the internal calls in each direction.
- `COLLAB_NODE_BASE_URL` (FastAPI): where FastAPI reaches the Owner.
- `COLLAB_API_BASE_URL` (Owner): where the Owner reaches FastAPI.
- `COLLAB_WS_PUBLIC_URL`: the WebSocket address the browser uses.
- `ALLOWED_ORIGINS`, `PORT`, `LOG_LEVEL`.
- Timing: `COLLAB_CHECKPOINT_QUIET_MS` (30000), `COLLAB_FASTAPI_TIMEOUT_MS` (5000), `COLLAB_LEASE_RENEWAL_INTERVAL_MS`, `COLLAB_SHUTDOWN_TIMEOUT_MS`, `COLLAB_SNAPSHOT_*`, `COLLAB_ACCESS_RECHECK_INTERVAL_MS`, `COLLAB_REVOCATION_POLL_INTERVAL_MS`.
- Snowflake (secret): `SNOWFLAKE_*` as in the collab `.env.example`. The Owner uses only `SOW_COLLAB_*` tables.
- `COLLAB_PURGE_REDACTIONS` (default off): when `true`, a checkpoint of a room with redacted comments also purges older update rows. Do not turn on without approval.
- `ENABLE_SOW_EMAIL_SENDING`: mentions send email only when `true`.

## Snowflake outage
- Symptom: the Owner logs `snowflake_unreachable`; new rooms refuse to open; open rooms go read-only after the lease window.
- Action: nothing to do in the app. Edits in open rooms are kept on the screen (IndexedDB) and sent when the Owner recovers.
- Check: `/healthz`; then open a room once it is back and confirm the offline edits arrive (verify).

## Stuck lease
- Symptom: a room says it is read-only or cannot open with "owner elsewhere".
- Check: the `SOW_COLLAB_LEASE` row for the deal (holder, expiry, fencing token). Wait for the lease to expire; leases renew every `COLLAB_LEASE_RENEWAL_INTERVAL_MS`.
- Do not delete the lease row by hand unless no Owner process is running (verify).

## Stuck flush (finalize refused)
- Symptom: finalize says "Live edits could not be saved before finalizing".
- Check: the Owner is running and reachable from FastAPI (`/healthz`). Then retry finalize. The flush endpoint is `POST /internal/room/flush`.
- If the draft moved on in another session (409), reload the draft and finalize again.

## Rollback to CKEditor
- Not built yet (CP-73). Until then: turn off `FE_SOW_LEXICAL_EDITOR` for the user, or unset `REACT_APP_SOW_LEXICAL_EDITOR` for the build. HTML is the system of record, so the CKEditor screen shows the last saved version. Save in the live editor before switching (verify).

## Key rotation
- Rotate `COLLAB_TICKET_SECRET` and the two service keys together, in FastAPI and the Owner, during a quiet period. Open sessions end within 30 seconds (ticket lifetime) and reconnect.
- Rotate the Snowflake key through the normal Snowflake process. Never commit `.env` files (see pending.txt A4).

## Logs and metrics
- Logs carry ids, counts, revisions and durations only, never document text (tested: observability.test.ts).
- Audit events for live sessions go to the existing audit table: `session_joined`, `session_left`, `checkpoint_created`, `access_changed`, `room_locked`, and `content_inserted` (CP-49).
- Metric lines: `metric=checkpoint_ms` with value and labels. Other metrics listed in CP-66 (durable-ack latency, queue depth, flush time, Snowflake statements, lease renewals and losses, rooms and sockets) are not emitted yet.

## Retention (decision needed)
- The retention period for live updates and the new audit rows is not decided. Live updates are pruned by compaction. Audit follows the existing policy. Owner decision pending (pending.txt).
