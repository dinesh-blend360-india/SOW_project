# Integration Contracts

All verified directly from route decorators, dependency code, and the shared TypeScript contract
files — not inferred. Where a detail is unconfirmed, it says so.

## Frontend-to-backend communication, overall

The frontend uses **three separate axios instances** against potentially different base URLs, not
one unified API client:

| Instance | Base URL env var | Auth | Used for |
|---|---|---|---|
| `api` / `apiToken` (`src/config/ApiService.ts`) | `REACT_APP_API_URL` | `apiToken` attaches `Authorization: Bearer <login_token>` via a request interceptor reading `localStorage` | Pricing Tool and most SOW Studio calls |
| `sowStudioApi` (`src/config/ApiService.ts`) | `REACT_APP_SOW_STUDIO_API_URL`, falling back to the relative path `/api/sow-studio` if unset | same JWT interceptor | SOW Studio-specific calls — note the fallback means this currently rides whatever host serves the frontend unless the env var is explicitly set somewhere in deployment, which was not confirmed |
| `agentApi` / `agentApiToken` / `agentFormApiToken` (`src/config/AgentsApiService.ts`) | `REACT_APP_BLEND_AGENTS_API_URL` | a **static shared API key** (`REACT_APP_AGENTS_API_KEY`), not the user's JWT | A separate "Blend Agents" backend service, distinct from `blend_ode_be` |

A near-duplicate axios wrapper exists at `src/pricing-tool/services/nativeEditorApi.ts` with the
same `REACT_APP_SOW_STUDIO_API_URL` fallback logic — flagged as possible duplication, not yet
consolidated.

**CORS** (backend, `src/main.py`, verified): `allow_origins` is an explicit list —
`[FRONT_END_URI, "https://eiaphtx37p.us-east-1.awsapprunner.com", BLEND_ID_URI]` (`FRONT_END_URI` is
derived from `REDIRECT_URI`'s scheme+host) — `allow_credentials=True`, all methods and headers
allowed, and `X-SOW-Review-ID`/`X-Process-Time` are exposed response headers.

## Authentication contract

1. Frontend completes an OAuth authorization-code flow against Microsoft Entra ID (how exactly the
   code is obtained in the browser was not fully confirmed — see `SYSTEM_ARCHITECTURE.md`).
2. Frontend calls `POST /auth/token` with the code.
3. Backend exchanges it via MSAL for a Microsoft Graph token, calls Graph `/v1.0/me`, then issues
   **its own JWT** — HS256, signed with `JWT_SECRET`, 15-day expiry (`create_jwt_token` in
   `pricing_tool/dependencies.py`).
4. Frontend stores this JWT in `localStorage` under the key `login_token` and attaches it as
   `Authorization: Bearer <token>` on every subsequent call (via the axios interceptors above).
5. Backend decodes and validates it per-request via `authorize_token(required_permissions=None)`
   (a FastAPI dependency factory), which also resolves the caller's roles/features/deal context
   via `get_role_and_features()` before the route body runs.

**401 behavior** (frontend, `ProtectedRoute.tsx`, verified): on a 401 from any call, the frontend
clears the stored token and redirects to `/auth-error`.

## REST API surface (verified via router inventory — see `REPOSITORY_MAP.md` for file paths)

All routers below except `sow_module`'s internal router are mounted under `/pricing_tool`, `/auth`,
or `/admin` as shown; none expose an API version prefix (no `/v1/` etc. found anywhere).

| Router | Mount | Endpoints | Representative paths |
|---|---|---|---|
| `pricing_tool` | `/pricing_tool` | 54 | `GET /hubspot_deals`, `POST /create_pricing_deal`, `POST /save_pricing_deal_version`, `POST /lock_pricing_deal`, `POST /sync_deals_from_hubspot`, `POST /save_openair_bookings` |
| `sow_studio` (routers.py) | `/pricing_tool` | 45 | `/sow_intake*`, `/drafts/*`, `/sows/{sow_id}/{state,change-log,version-history,approvals,submit-for-approval,publish,finalize,msa-ticket,final-docx}` |
| `sow_studio.api.native` | `/pricing_tool` | 11 | `/sows/{sow_id}/native-editor/{bootstrap,state,checkpoint,reload-latest,invitations}` (pre-Lexical pilot) |
| `sow_studio.api.public` | `/pricing_tool` | 4 | `GET/POST /sow-approval/{token}(/draft)`, `/sow-share/{token}` — **unauthenticated**, token-gated |
| `sow_module` | `/pricing_tool` | 1 | `POST /sows/{sow_id}/collab/session` — feature-flagged, JWT-authenticated |
| `sow_module` internal | *(root, no prefix)* | 3 | `POST /internal/sow-collab/{resolve-access,document-state,checkpoint}` — service-key only, `include_in_schema=False` |
| `auth` | `/auth` | 16 | `POST /token`, CRUD for `users`/`groups`/`features`/`hubspot_roles` |
| `admin` | `/admin` | 8 | `/exports/*`, `/pricing-versions`, `/regions`, base-price-version 3-step workflow |

For the full endpoint list of the older routers (`pricing_tool`, `sow_studio`), read the
`@router.*` decorators directly in each `routers.py` — this document does not re-list all 120+
endpoints; it records the pattern and the ones most relevant to cross-cutting integration.

## The SOW editor module's contracts (the newest, most precisely specified integration surface)

This module has its own authoritative, machine-checked contract files, which this document
indexes rather than duplicates:

- **TypeScript source of truth**: `blend_ode/src/sow_module/shared/bridge.ts` (wire shapes),
  `roles.ts` (`Role`, `Capability`, `ActorRef`, `ROLE_CAPABILITIES`), `protocol.ts` (WebSocket
  message shapes, close codes).
- **Prose companion**: `sow_docs/tracking/contracts.md` — read this for the full WebSocket
  lifecycle, not reproduced here.
- **Python mirror**: `blend_ode_be/src/sow_module/role_table.py` (kept in parity-tested sync with
  `role-table.json` via `tests/test_sow_module_role_table_parity.py`).

### Session endpoint

`POST /pricing_tool/sows/{sow_id}/collab/session` — browser, JWT-authenticated.

Gate (all must pass): `authorize_sow_access` (FE_SOW_STUDIO / FE_SOW_STUDIO_PILOT / FE_ADMIN) →
`FE_SOW_LEXICAL_EDITOR` or `FE_ADMIN` → the feature actually being configured
(`COLLAB_TICKET_SECRET` + `COLLAB_WS_PUBLIC_URL` both set) → `assert_deal_access`.

Response (`SessionResponse`, camelCase on the wire):
```json
{ "ticket": "<HS256 JWS>", "wsUrl": "wss://...", "expiresAt": "2026-11-06T12:00:30Z",
  "role": "editor", "capabilities": ["read", "comment", "suggest", "edit", "resolve_thread", "checkpoint"] }
```

Ticket claims (`TicketClaims`): `jti` (random 128-bit), `aud: "sow-collab"`, `sow_id`, `actor`
(`ActorRef`), `role`, `iat`, `exp` (`iat + 30` seconds). Signed with `COLLAB_TICKET_SECRET` —
**a different secret from `JWT_SECRET`**; a login token never works as a ticket and vice versa.

### Internal endpoints (Node service → FastAPI only, never public)

All three require header `X-Sow-Collab-Service-Key` (constant-time compared against
`COLLAB_SERVICE_KEY_NODE_TO_API`) and are excluded from the OpenAPI schema.

| Endpoint | Request | Response |
|---|---|---|
| `POST /internal/sow-collab/resolve-access` | `{sowId, actor: ActorRef}` | `{allowed:true, role, capabilities, locked}` or `{allowed:false, reason}` |
| `POST /internal/sow-collab/document-state` | `{sowId}` | `{sowId, html, version, revision, status, locked}` |
| `POST /internal/sow-collab/checkpoint` | `{sowId, html, baseRevision, baseVersion, upToSeq, fencingToken, attributedTo, reason}` | `{version, revision, savedAt, unchanged}` |

`checkpoint`'s error responses carry a structured `{code, message}` detail on 409
(`stale_revision` or `sow_locked`) — the Owner is expected to call `document-state` and retry once
after a `stale_revision`.

### FastAPI → Node hook

`notify_access_changed(sow_id, reason, actor_id=None)` posts to the Node service's
`POST /internal/access-changed` with `COLLAB_SERVICE_KEY_API_TO_NODE` in the same header. Never
raises — a failed notification never blocks the FastAPI action that triggered it (e.g. revoking a
guest invitation still succeeds even if the Node service is unreachable).

### WebSocket (implemented; behind the feature flag)

- Endpoint on the Owner: `wss://<owner>/collab` (the path is exact; requests to any other path are dropped).
  The ticket is sent in the first `HELLO` message, never in the URL. Protocol version `1`.
- Client to server: `HELLO {type, protocol, ticket}`, `REAUTH {type, ticket}`, `CHECKPOINT_NOW {type}`, `CONTENT_INSERTED {type, idempotencyKey, contentHash, target, source, model?, requestId?}`
  (CP-49: audit notice for inserted content; needs `edit`; hash only, never content or prompts).
  Binary frames: `[kind][payload]`, kind `0` = SYNC (Yjs update), `1` = AWARENESS, `2` = CHUNK (one
  piece of a larger SYNC payload; 256 KB pieces, in order).
- Server to client: `READY`, `DURABLE`, `ROLE_CHANGED`, `REAUTH_REQUIRED`, `CHECKPOINTED {version, revision, upToSeq?}`,
  `ERROR {code, message, fatal}`.
- Close codes: `4401` bad ticket, `4403` access removed, `4410` locked, `4301` owner hint, `1013` room full
  (more than 100 connections).
- Full message list and the frozen surface: `sow_docs/tracking/core_api_freeze.md`.

## Shared data models

- **`Role`** (`'viewer' | 'commenter' | 'editor' | 'sow_admin'`) and **`Capability`** (9 values,
  `role-table.json`) — identical in TypeScript (`roles.ts`) and Python (`role_table.py`), parity
  enforced by a test on each side.
- **`ActorRef`** (`{kind: 'user'|'guest', id, email, name}`) — used in both the ticket and the
  internal `resolve-access`/`checkpoint` requests.
- **HTML contract** (`html-contract.json`): the allowlisted tag/attribute/class set for SOW
  content, enforced identically (parity-tested) by `shared/allowlist.ts` (TypeScript) and
  `html_allowlist.py` (Python). This is the only HTML sanitizer ever used for checkpoint content;
  the pre-existing `sanitize_rich_text` is a different, stricter, lossy sanitizer used only by the
  public approval page and must never be used here (`sow_docs/tracking/decisions.md` ADR-009).

## Error response conventions

No project-wide error envelope was found — most endpoints raise `fastapi.HTTPException(status_code,
detail=<string>)` and let FastAPI's default `{"detail": "..."}` JSON body stand. The newer
`sow_module` internal `checkpoint` endpoint is the one place a **structured** detail object
(`{code, message}`) is used instead of a bare string — not yet a project-wide convention.

## API versioning

None found. No `/v1/` prefix, no `Accept`/version header scheme, anywhere in either repo.

## CORS requirements

See the CORS section above — any new frontend origin must be added to `common/config.py`'s
`FRONT_END_URI`/`BLEND_ID_URI` resolution or the explicit allow-list in `main.py`, not assumed to
work automatically.
