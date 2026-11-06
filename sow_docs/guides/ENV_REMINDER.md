# Reminder: Snowflake env values (fill in later)

Not committed (this folder is not a git repo). No secrets belong in this file.

## Allowed target, and nothing else

- Database.schema: `BLEND_INTERNAL_IT_APPS.NEXDRAFT`
- `SF_USER`: `PRICING_APP_DEV_USER` (unconfirmed: no real `.env` existed when this was written)

## When you add the env files, check

1. `SF_DATABASE=BLEND_INTERNAL_IT_APPS` and `SF_SCHEMA=NEXDRAFT` in `blend_ode_be/.env` (the code reads `SF_USER`, `SF_DATABASE`, `SF_SCHEMA`, `SF_WAREHOUSE`, `SF_ROLE`, `SF_ACCOUNT`, and the key-pair settings in `src/common/config.py`). `.env.example` still shows the old `RESOURCE_REQUEST_TOOL` / `PRICING_TOOL_SANDBOX_HYBRID`; do not copy those.
2. The existing app tables (`SOWS`, `SOW_DRAFT_VERSIONS`, `SOW_CHANGE_LOG`, `SOW_STATUS_HISTORY`, `SOW_SIGNOFFS`, and the rest) are queried by unqualified name. If the session schema is `NEXDRAFT`, those tables must exist in `NEXDRAFT` or SOW Studio breaks. Confirm they are there before pointing the app at it.
3. `PRICING_APP_DEV_USER` has `USAGE` on the database and schema, `SELECT/INSERT/UPDATE/DELETE` on the existing SOW tables, and `CREATE TABLE` in `NEXDRAFT` for the `SOW_COLLAB_*` migration. Check with `SELECT CURRENT_USER(), CURRENT_ROLE();` and `SHOW GRANTS TO USER PRICING_APP_DEV_USER;`.
4. The Node collab service gets its own env file at `blend_ode_be/sow_module/collab/.env` (names defined in CP-06; see `.env.example` there). Same database, schema and user restriction.
5. Frontend `blend_ode/.env` holds no Snowflake values at all.
6. The module migration (CP-17) is never applied without your explicit approval.

## Rules that stay in force

- Module SQL uses fully qualified names `BLEND_INTERNAL_IT_APPS.NEXDRAFT.<TABLE>` and bind variables.
- Secrets only in env files that are git-ignored; only `.env.example` files are created by Claude.
- `insecure_mode: True` in `common/database.py` is reported, not changed.
