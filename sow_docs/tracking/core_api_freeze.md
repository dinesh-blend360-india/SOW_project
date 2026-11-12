# Core API freeze

Version stamp: **Core API 1.0, frozen 2026-11-09** (`PROTOCOL_VERSION = 1`, `CHECKPOINT` contract as in `blend_ode_be/sow_module/collab/src/shared`).

Any change to the items below needs a note in `prompt_log.md` and a new version stamp.

## HTTP routes

| Route | Caller | Notes |
|---|---|---|
| `POST /pricing_tool/sows/{sow_id}/collab/session` | Frontend | Feature-gated (`FE_SOW_LEXICAL_EDITOR`, or `FE_ADMIN` when enabled). Returns a 30-second ticket. |
| `POST /internal/sow-collab/resolve-access` | Node collab service | Service-key protected. Hidden from public OpenAPI. |
| `POST /internal/sow-collab/document-state` | Node collab service | Service-key protected. Read only. |
| `POST /internal/sow-collab/checkpoint` | Node collab service | Service-key protected. The only write path into `SOW_DRAFT_VERSIONS`. Returns 409 (stale revision) or 428 (revision required). |

## WebSocket (`/collab`)

Client to server:
- `HELLO` (text JSON): `{ type, protocol, ticket }`
- `REAUTH` (text JSON): `{ type, ticket }`
- `CHECKPOINT_NOW` (text JSON): `{ type }`. Requires the `checkpoint` capability.
- Binary frames: `[kind][payload]`, kind `0` = SYNC (raw Yjs update), kind `1` = AWARENESS (y-protocols).

Server to client:
- `READY`, `DURABLE`, `ROLE_CHANGED`, `REAUTH_REQUIRED`, `CHECKPOINTED { version, revision, upToSeq? }`, `ERROR { code, message, fatal }`
- `ERROR` codes used: `stale_revision`, `sow_locked`, `validation_failed`, `service_unavailable`, `access_denied`, and the protocol codes already defined.

Close codes: `4401` bad ticket, `4403` access removed (do not reconnect), `4410` locked (fall back to read-only HTML), `4301` owner hint (request a new session).

## Browser components and props

- `LexicalDraftEditor` (default export) takes the same props as `DraftEditor`, plus `onReady?: (api: SowEditorApi) => void`.
- `lazy.tsx` default export `LazyLexicalDraftEditor`: the only import path for screens.
- `SowEditorApi`: `getHtml()`, `setHtml(html)`, `insertContent({ format, content, target })`, `checkpoint()`, `focus()`, `scrollToBlock(blockId)`.
- `insertContent` targets: `cursor`, `end`, `afterBlock { blockId }`, `replaceBlock { blockId }`.
- `onContentChange(html)` fires only for author edits, never for programmatic ones.
- `isLexicalEditorEnabled(userDetailsJson, buildFlag)` decides who sees the Lexical editor.

## Owner (Node) module boundary

- Writes only `SOW_COLLAB_*` tables (plus the collab row's source revision). Never `SOWS`, `SOW_DRAFT_VERSIONS`, `HUBSPOT_DEALS`, `SOW_NATIVE_*`.
- Checkpoints only through `POST /internal/sow-collab/checkpoint`.
- Validation boundary: `validateUpdate` (walks the Lexical Yjs shape) and `sanitizedRootHtml` (shared allowlist).

## Configuration

- `COLLAB_CHECKPOINT_QUIET_MS` (default `30000`): quiet period before an automatic checkpoint.
- `COLLAB_FASTAPI_TIMEOUT_MS` (default `5000`).
- Build flag `REACT_APP_SOW_LEXICAL_EDITOR=true` (admins only, with `FE_ADMIN`).

## Change log

- **2026-11-12 (CP-49, non-breaking addition):** client text message `CONTENT_INSERTED`
  `{type, idempotencyKey, contentHash, target, source, model?, requestId?}`. `contentHash` is a lowercase
  SHA-256 hex of the sanitized HTML, never the content. `target` is cursor, end, afterBlock, replaceBlock
  or document; `source` is llm, addin or api. Requires capability `edit` (otherwise `access_denied`);
  a malformed message gets `protocol_error` (non-fatal). The Owner writes one `content_inserted` audit
  row per idempotency key. No new server message. Existing messages are unchanged.

- **2026-11-11 (CP-40, non-breaking addition):** binary frame kind `2` = CHUNK added to `FRAME_KIND`
  (one 256 KB piece of a larger SYNC payload, in order, max 16 MB assembled). Close code `1013`
  (room full, more than 100 connections) added. Config `COLLAB_CHECKPOINT_QUIET_MS` (default 30000)
  documented. Existing frame kinds and close codes are unchanged.

- **2026-11-12 (CP-51, non-breaking additions):** server text message `CONVERSION_REPORT {report}` (to SOW
  admins only, once after READY, only when opening an older draft changes it; names and counts only). Client
  text message `DRAFT_META {openItems?, addedAddIns?, sowName?}` (needs `edit`; the name needs `rename`;
  open items at most 200, each at most 4,000 characters of JSON, known fields only). Internal checkpoint
  request gains optional `openItems`, `addedAddIns`, `sowName`; absent means keep the stored values, and the
  backend retries once with the stored values if it rejects the reported ones. No existing field changed.
