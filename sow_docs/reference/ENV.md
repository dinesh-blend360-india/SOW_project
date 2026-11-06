# SOW module: backend environment variables

Names only. These are not read yet; CP-19 adds them to `src/common/config.py`
and the feature stays disabled when they are unset.

| Variable | Purpose |
|---|---|
| `COLLAB_TICKET_SECRET` | HMAC secret for the 30-second WebSocket ticket. Must differ from `JWT_SECRET`. |
| `COLLAB_SERVICE_KEY_NODE_TO_API` | Key the Node service sends in `X-Sow-Collab-Service-Key` on `/internal/sow-collab/*`. |
| `COLLAB_SERVICE_KEY_API_TO_NODE` | Key FastAPI sends in `X-Sow-Collab-Service-Key` on the Node `/internal/access-changed` hook. |
| `COLLAB_NODE_BASE_URL` | Private URL of the Node collaboration service (used by `notify_access_changed`). |
| `COLLAB_WS_PUBLIC_URL` | Public WebSocket URL returned to the browser by the session endpoint. |

Feature flag (a user feature, not an environment variable): `FE_SOW_LEXICAL_EDITOR`.

Snowflake: the backend keeps using its existing `SF_*` variables. The module's tables live in `BLEND_INTERNAL_IT_APPS.NEXDRAFT` with user `PRICING_APP_DEV_USER` only (decisions.md ADR-012; see `SOW_project/ENV_REMINDER.md`).
