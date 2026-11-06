# Contracts (CP-05)

Status: **Proposed**. Types live in `blend_ode/src/sow_module/shared/` (source of truth) and are vendored to `sow_module/collab/src/shared/`. This document is the prose companion; if the two disagree, fix the types and this file together.

Parties: **Browser** (React `LexicalDraftEditor`), **Owner** (Node collaboration service, one instance owns a document at a time), **FastAPI** (`blend_ode_be`, the only writer of `SOW_DRAFT_VERSIONS`), **Snowflake**.

## 1. Endpoints and who may call them

| # | Endpoint | Hosted by | Caller | Credential | Gate (must all pass) |
|---|---|---|---|---|---|
| 1 | `POST /sows/{sow_id}/collab/session` | FastAPI | Browser (Blend user) | JWT (`authorize_token`) | `authorize_sow_access` (FE_SOW_STUDIO, FE_SOW_STUDIO_PILOT or FE_ADMIN) + `FE_SOW_LEXICAL_EDITOR` (or FE_ADMIN) + `assert_deal_access` |
| 2 | `POST /sow-native/guest/{sow_id}/collab/session` | FastAPI | Browser (guest) | Guest session bearer token (`authorize_guest_session`) | Session not revoked or expired; invitation tied to this SOW. Feature gating for guests: see open question G1 |
| 3 | `GET wss://<owner>/ws/sow/{sow_id}` | Node | Browser | Ticket sent in first message `HELLO` (not in the URL) | Ticket valid (section 3) + Owner re-confirms through endpoint 4 |
| 4 | `POST /internal/sow-collab/resolve-access` | FastAPI | Node only | Service key header | Key valid. Returns role, capabilities, locked. Never exposed through the public ingress |
| 5 | `POST /internal/sow-collab/document-state` | FastAPI | Node only | Service key header | Key valid. Returns latest HTML, version, revision, status |
| 6 | `POST /internal/sow-collab/checkpoint` | FastAPI | Node only | Service key header | Key valid + fencing token is the current lease token + HTML passes `html_allowlist.py`. Calls `save_sow_draft` |
| 7 | `POST /internal/access-changed` | Node | FastAPI only | Service key header | Key valid. FastAPI calls it on guest revoke, role change, deal access change, finalize |

Service key: header `X-Sow-Collab-Service-Key`. Proposal: one key per direction (Node-to-FastAPI, FastAPI-to-Node), compared in constant time, rotated through env vars only. Names only are recorded here; values never appear in any file.

Internal endpoints (4-7) must not be reachable from the internet. Whether `deployment/` can restrict them is unknown (open question Q-N1).

## 2. Session endpoint (1 and 2)

Request: empty body. Response `SessionResponse`:

```
{ ticket, wsUrl, expiresAt, role, capabilities }
```

Steps on FastAPI: authenticate, check feature flag, `assert_deal_access`, resolve role (section 6), refuse with `access_denied` / `feature_disabled` / `sow_not_found`, mint ticket, return `wsUrl` of the instance that owns the document (lease lookup) or the router in front of it.

## 3. Ticket

- Format: compact JWS, HS256. Claims `TicketClaims`: `jti`, `aud="sow-collab"`, `sow_id`, `actor`, `role`, `iat`, `exp`.
- Lifetime: **30 seconds** (`exp = iat + 30`). Bound to one SOW, one actor, one role.
- `jti` is unique (random 128-bit). The Owner accepts each `jti` once and remembers it for at least 60 seconds.
- Signing secret: `COLLAB_TICKET_SECRET`, **a different secret from `JWT_SECRET`**, shared only by FastAPI and the Owner. A ticket never works as a login token and a login token never works as a ticket (distinct `aud`, distinct key).
- The role inside the ticket is a hint. The Owner confirms with `resolve-access` on connect and every 60 seconds.

## 4. Internal calls (4 to 7)

Types are in `bridge.ts`.

- **resolve-access** `{sowId, actor}` -> `{allowed:true, role, capabilities, locked}` or `{allowed:false, reason}`.
- **document-state** `{sowId}` -> `{html, version, revision, status, locked}`. `revision` is `_draft_revision(row)` (crud.py:1443). Used to seed a document with no stored Yjs state and to re-base after a conflict.
- **checkpoint** `{sowId, html, baseRevision, baseVersion, upToSeq, fencingToken, attributedTo, reason}` -> `{version, revision, savedAt, unchanged}`. FastAPI: verifies the fencing token against the lease row, validates HTML with `html_allowlist.py`, then calls the existing `save_sow_draft` with `expected_revision=baseRevision` and `create_version=False`, passing the current `comments`, `added_add_ins` and `open_items` through unchanged (so the editor module changes `content` only). Errors: `stale_revision` (409), `sow_locked` (409), `validation_failed` (422), and a fencing failure returns `owner_elsewhere` (409).
- **access-changed** `{sowId, actorId?, reason}` -> `{affectedConnections}`. The Owner re-runs resolve-access for the matching connections immediately and closes with 4403 when access is gone.

Checkpoint attribution: `attributedTo` is the last human editor in the window; it becomes `CREATED_BY_EMAIL` / `CREATED_BY_NAME`. Open question Q-C1: how FastAPI builds a `current_user` for `save_sow_draft` from an `ActorRef` (guests have no `User`; guests are commenters and never trigger content writes themselves, but a window can contain only guest suggestions).

## 5. WebSocket lifecycle

1. Browser calls endpoint 1 (or 2), gets the ticket, opens `wsUrl`.
2. Within **5 s** (`HELLO_TIMEOUT_MS`) the browser sends `HELLO {protocol, ticket, lastDurableSeq?}`. No HELLO: close 4401.
3. Owner verifies signature, `aud`, `exp`, `jti` unused; calls `resolve-access`. Failures: 4401 bad ticket, 4403 access removed.
4. Owner checks the lease. Not the owner: close **4301** with the owning instance id as the reason; the browser requests a new session (a fresh ticket, since this one is spent).
5. SOW locked (finalized): close **4410**; the browser shows the existing read-only HTML view.
6. Owner sends `READY {sessionId, role, capabilities, version, revision}`, then Yjs sync (binary frames, first byte `FRAME_KIND.SYNC`). Awareness uses `FRAME_KIND.AWARENESS`.
7. Live: updates are held in memory, flushed in batches to `SOW_COLLAB_*`; after each committed batch the Owner sends `DURABLE {upToSeq}`. On a checkpoint the Owner sends `CHECKPOINTED {version, revision, upToSeq}`; the browser stores the new revision (this keeps the existing CKEditor-era `expected_revision` flow consistent).
8. Every 60 s the Owner re-runs `resolve-access` for each connection. Change of role: `ROLE_CHANGED`. Removal: close 4403. Finalize: close 4410.
9. Re-auth: at the session age cap (proposal 8 h, same as guest sessions) or after a service-key rotation the Owner sends `REAUTH_REQUIRED {deadlineMs}`; the browser fetches a fresh ticket and sends `REAUTH {ticket}`; no answer in time: close 4401.
10. Save button: browser sends `CHECKPOINT_NOW` (needs capability `checkpoint`); the Owner checkpoints and answers `CHECKPOINTED`.
11. Close: the last client leaving triggers a checkpoint (`last_client_left`); shutdown triggers one (`shutdown`).

Close codes: 4401 bad ticket, 4403 access removed, 4410 locked, 4301 owner hint. Server `ERROR` messages carry a `CollabErrorCode` and `fatal`.

## 6. Concurrency with the existing save path

During the feature-flag transition some users still run CKEditor and call `POST /pricing_tool/drafts/save` directly. Their saves change `revision`. The Owner's next checkpoint then fails with `stale_revision`; the Owner calls `document-state`, and (proposal) keeps the Yjs document as the winner for flagged SOWs and re-bases its `baseRevision`. Which side wins when both paths write the same SOW is open question Q-C2; the safest rule is that a SOW with an active Owner session rejects CKEditor saves with a clear message.

## 7. Role mapping (PROPOSED, needs user confirmation)

| Role | Capabilities | Who (proposal) |
|---|---|---|
| viewer | read | Blend users who can see the deal but may not edit, and everyone when the SOW is locked or a historical version is open |
| commenter | read, comment, suggest | **Guests** (invited reviewers). Cannot resolve threads (the native editor already forbids guests resolving or editing existing threads) |
| editor | read, comment, suggest, edit, resolve_thread, checkpoint | **Blend participants** on the deal who pass the editing gate today. Cannot rename the SOW (added in CP-06) |
| sow_admin | everything above + rename, manage_access, finalize | The people allowed to finalize today (see the fact below) |

Fact from the code: finalize today is gated by `authorize_sow_access` + `assert_deal_access` + status "Approved" (audit_backend.md section 7). That is the **same gate as editing**, so no finer "finalizer" role exists in the code yet. "Those allowed to finalize today" therefore equals "all editors", unless a deal-level role restricts it. `FE_ADMIN` is the only existing elevated flag. The mapping of `sow_admin` is therefore a question, not a fact (question Q1 in `decisions.md`).

`roleToCapabilities` in `roles.ts` is a stub that returns no capabilities until CP-06 implements and tests this table.

## 8. HTML contract

Machine-readable: `shared/html-contract.json` (typed by `html-contract.ts`). Derived from real CKEditor output found in the audits and tests: tags `p h1-h6 ul ol li table colgroup col thead tbody tr td th strong i u s mark span br`; `mark` classes `marker-yellow marker-green marker-pink marker-blue pen-red pen-green sow-studio-placeholder-marker sow-studio-manual-placeholder`; `span` styles `font-size` (10/12/14/18/24/32 px) and `font-family`; `text-align` on blocks; attributes `data-add-in-id` (blocks and tables), `colspan`, `rowspan`. `figure` (`class="table"`), `div`, `section`, `font`, `a` are unwrapped on import and never emitted; `b`, `em`, `strike`, `del` map to `strong`, `i`, `s`.

Invariants required by existing backend parsers (audit_backend.md section 14): literal brackets; no wrapper between `table` and `tr`; two-cell signature rows; exact billing-schedule header text; fixed-fee sentence inside one `p`; headings stay headings. The contract file lists the unverified items (table inline styles, `colgroup`, `text-align` on cells) that must be checked against real saved drafts.

## 9. Environment variable names (no values anywhere in the repo)

`COLLAB_TICKET_SECRET`, `COLLAB_SERVICE_KEY_NODE_TO_API`, `COLLAB_SERVICE_KEY_API_TO_NODE`, `COLLAB_WS_PUBLIC_URL`, `COLLAB_API_BASE_URL` (Node side), `COLLAB_NODE_BASE_URL` (FastAPI side). Final names are confirmed in CP-06 and `ENV.md`.
