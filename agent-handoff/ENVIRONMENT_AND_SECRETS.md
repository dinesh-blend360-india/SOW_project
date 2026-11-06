# Environment and Secrets

Never copy an actual `.env` file's contents into this document, into chat, or into any committed
file. Values below are placeholders only. For the real values, ask whoever owns each credential —
this document records what's needed and why, not what the values are.

## ⚠️ Known secret-exposure issue — read before touching `blend_ode`'s env files

**`blend_ode/.env.development` and `.env.production` are tracked in Git and contain what appear to
be real, working secrets**, not placeholders — confirmed by direct inspection:
- A plaintext database password (`DB_DSN`).
- A `REACT_APP_AGENTS_API_KEY` value (different per environment, both committed).
- A CKEditor Premium trial license token in `.env.development` (decodes as a real JWT with `exp`,
  `jti`, and a live `usageEndpoint`).
- `blend_ode/src/config/MsalConfig.ts` also has a hardcoded Azure AD `clientId` and tenant GUID,
  tracked in Git.

`.gitignore` was later updated to list these files, but `.gitignore` does not retroactively untrack
already-committed files or scrub history — they remain in the repo's history today. See
`KNOWN_ISSUES_AND_BLOCKERS.md` for the full writeup and suggested remediation. **This handoff
system does not fix this** (out of scope per this task's own restrictions) — it is recorded here so
nobody mistakes `.env.development` for a safe example file to read from or copy into a new `.env`.

There is currently **no `.env.example` file in `blend_ode`** — a real gap, since without one, a new
developer's only way to discover required variable names is to read the tracked (secret-containing)
`.env.*` files. This document's tables below exist specifically to close that gap without exposing
the actual values.

## `blend_ode_be` environment variables

A `.env.example` **does exist** in this repo and covers most of the table below. Two gaps found:
`HUBSPOT_SOW_STATUS_PROPERTY` and `CKEDITOR_CLOUD_TOKEN_TIMEOUT_SECONDS` are listed in it but have
no matching `os.getenv()` call anywhere in `src/` (likely stale); the 5 `COLLAB_*` variables below
are **not yet added** to it (a gap from CP-19 of the SOW editor module's build — worth fixing).

| Variable | Purpose | Default if unset |
|---|---|---|
| `CLIENT_ID`, `CLIENT_SECRET`, `TENANT_ID`, `REDIRECT_URI` | Microsoft Entra ID / MSAL app registration for the login exchange | none — required for login |
| `JWT_EXPECTED_AUDIENCE` | Declared, no confirmed call site using it | none |
| `JWT_SECRET` | Signs/verifies the app's own login JWT (HS256, 15-day expiry) | none — required |
| `SF_ACCOUNT`, `SF_USER`, `SF_DATABASE`, `SF_SCHEMA`, `SF_WAREHOUSE`, `SF_ROLE`, `SF_PRIVATE_KEY` | Snowflake key-pair connection for the main application schema | none — required |
| `SF_PASSWORD` | Declared but unused (key-pair auth is what's actually active) | n/a |
| `HUBSPOT_TOKEN`, `HUBSPOT_URL`, `HUBSPOT_DRAFT_SOW_URL_PROPERTY` | HubSpot API writes (deal property updates) and deal link construction | `HUBSPOT_DRAFT_SOW_URL_PROPERTY` defaults to `"draft_sow_"` |
| `HUBSPOT_SYNC_API_URL`, `HUBSPOT_SYNC_API_KEY` | A HubSpot sync-related API — exact call site not fully traced | none |
| `ENVIRONMENT` | Environment label | none |
| `RESOURCE_REQUEST_TOOL_URL` | Link to the sibling Resource Request Tool app | none |
| `RRT_TENANT_ID`, `RRT_CLIENT_ID`, `RRT_CLIENT_SECRET` | Also reused for Microsoft Graph email sending (`send_mail.py`) | none |
| `RRT_EMAIL_TO`, `RRT_EMAIL_CC`, `RRT_EMAIL_BCC`, `RRT_EMEA_EMAIL_TO` | Resource-request notification routing | none |
| `LOW_MARGIN_NOTIFICATION_EMAILS` | Comma-separated alert recipients for low client margin | `""` (disables the alert) |
| `PIPELINE_IDS` | Comma-separated HubSpot pipeline IDs the app filters deals to | none |
| `WORKATO_API_TOKEN` | Likely used by an external sync job; exact call site not confirmed in this repo | none |
| `AGENT_API_TOKEN`, `AGENT_API_URL` | Teams error-notification / diagnose-error agent | none (feature silently no-ops if unset) |
| `S3_BUCKET_NAME`, `MAX_SOW_MATERIAL_UPLOAD_MB` | SOW intake material storage | `MAX_SOW_MATERIAL_UPLOAD_MB` defaults to `25` |
| `ENABLE_SOW_EMAIL_SENDING` | Gates whether SOW-related emails actually send | `false` |
| `SOW_NOTIFICATION_TEST_RECIPIENT`, `SOW_NOTIFICATION_SENDER_EMAIL` | Email routing for SOW notifications | sender defaults to `resourcerequest@blend360.com` |
| `SOW_SIGNOFF_DEMO_MODE` | Demo-mode toggle for the sign-off flow | `false` |
| `BEDROCK_MODEL` | Default AI model ID for Deal Desk agents via Bedrock | `us.anthropic.claude-sonnet-4-6` |
| `SOW_SOURCE_CACHE_ENABLED`, `SOW_NARRATIVE_PROMPT_CACHE`, `SOW_NARRATIVE_COMPACT_PROMPT` | AI prompt/caching toggles | all default `true` |
| `MAX_TOKENS`, `TEMPERATURE`, `TOP_P` | AI generation parameters | none (provider defaults apply) |
| `AGENT_RES_INDEX` | Unclear exact purpose from config alone | `0` |
| `ANTHROPIC_CLIENT_MODE` | `strands` (via Bedrock) or `direct` (api.anthropic.com) | `strands` |
| `ANTHROPIC_API_KEY` | Required only when `ANTHROPIC_CLIENT_MODE=direct` | none |
| `ANTHROPIC_MODEL_HAIKU`, `ANTHROPIC_MODEL_SONNET`, `ANTHROPIC_MODEL_OPUS` | Named model IDs used by the Deal Desk agents | hardcoded model-ID defaults in code |
| `DEAL_DESK_EXTRACTION_PRIMARY_MODEL`, `DEAL_DESK_EXTRACTION_FALLBACK_MODEL`, `DEAL_DESK_LEGAL_REVIEW_MODEL`, `DEAL_DESK_ORCHESTRATOR_MODEL`, `DEAL_DESK_FIX_MODEL` | Which model each Deal Desk agent role uses | defaults to one of the `ANTHROPIC_MODEL_*` constants |
| `DEAL_DESK_MODEL_TIMEOUT_SECONDS` | Timeout for a Deal Desk model call | `60` |
| `DEAL_DESK_PARALLEL_REVIEW` | Whether legal/commercial review runs in parallel | `true` |
| `BLEND_ID_URI` | A frontend origin allowed by CORS | `http://localhost:3000` |
| `SOW_PUBLIC_SHARE_BASE_URL`, `SOW_APPROVAL_BASE_URL` | Base URLs used when building public share/approval links | derived from `FRONT_END_URI` if unset |
| `SWITCHBOARD_TICKET_API_URL`, `SWITCHBOARD_TICKET_API_TOKEN` | MSA ticket creation | none |
| `HS_PRODUCT_ID` | A HubSpot product ID reference | none |
| `OPEN_AIR_USERNAME`, `OPEN_AIR_PASSWORD`, `OPEN_AIR_COMPANY_ID`, `OPEN_AIR_API_KEY`, `OPEN_AIR_API_NAMESPACE`, `OPEN_AIR_CLIENT_NAME`, `OPEN_AIR_BASE_URL` | OpenAir XML API connection | none |
| `ENABLE_OPENAIR_AUTO_CREATE` | Auto-create OpenAir bookings on resource-request email | `false` |
| `COLLAB_TICKET_SECRET` | Signs the SOW collaboration module's short-lived WebSocket tickets — **must differ from `JWT_SECRET`** | none — feature stays disabled until set |
| `COLLAB_SERVICE_KEY_NODE_TO_API` | Header value the Node service must send to call FastAPI's `/internal/sow-collab/*` | none — feature stays disabled until set |
| `COLLAB_SERVICE_KEY_API_TO_NODE` | Header value FastAPI sends when calling the Node service's `/internal/access-changed` | none |
| `COLLAB_NODE_BASE_URL` | Private URL of the Node collaboration service | none |
| `COLLAB_WS_PUBLIC_URL` | Public WebSocket URL returned to the browser by the session endpoint | none — feature stays disabled until set |

## `blend_ode` environment variables

No `.env.example` exists (see the warning above). This table is the closest thing to one; create a
real `.env.example` with placeholder values as a follow-up (see `TASK_BOARD.md`).

| Variable | Purpose |
|---|---|
| `REACT_APP_API_URL` | Base URL of the main `blend_ode_be` API |
| `REACT_APP_BLEND_AGENTS_API_URL` | Base URL of a separate "Blend Agents" service |
| `REACT_APP_AGENTS_API_KEY` | Static shared bearer token for the Agents API (not per-user) — **treat as a secret** |
| `REACT_APP_SOW_STUDIO_API_URL` | Optional override for SOW Studio API calls; falls back to the relative path `/api/sow-studio` if unset |
| `REACT_APP_SOW_STUDIO_USE_MOCKS` | Dev/testing toggle to use mock SOW Studio data instead of live calls |
| `REACT_APP_MSAL_REDIRECT_URI` | OAuth/MSAL redirect URI, differs per environment |
| `REACT_APP_MAINTENANCE_MODE` | Renders a maintenance page instead of the app when set |
| `REACT_APP_ENV` | Environment label shown in the app shell |
| `REACT_APP_SOW_SIGNOFF_DEMO_MODE` | Demo-mode toggle for the sign-off screen |
| `REACT_APP_CKEDITOR_ENABLE_CLOUD_SERVICES`, `REACT_APP_CKEDITOR_CLOUD_TOKEN_URL`, `REACT_APP_CKEDITOR_LICENSE_KEY` | Present in `.env.development` but **not referenced in any runtime source file found** — `DraftEditor.tsx` hardcodes `licenseKey: 'GPL'`. Likely dead config; confirm with whoever owns CKEditor licensing before relying on or removing these. |
| `GENERATE_SOURCEMAP` | CRA built-in — disables production sourcemaps |
| `BROWSER` | CRA built-in — controls whether `npm start` auto-opens a browser |

`DB_DSN` also appears in both tracked `.env.*` files but has **no reference anywhere in `src/`** —
it appears to be a leftover from a backend `.env` accidentally merged into these frontend files,
and is itself part of the secret-exposure issue above, not a variable the frontend actually uses.

## External services requiring configuration

| Service | Configured via | Local vs. production |
|---|---|---|
| Microsoft Entra ID (SSO) | `CLIENT_ID`/`CLIENT_SECRET`/`TENANT_ID`/`REDIRECT_URI` (backend), `MsalConfig.ts` + `REACT_APP_MSAL_REDIRECT_URI` (frontend) | Redirect URI must match the environment's actual frontend URL |
| Snowflake | `SF_*` (backend's main schema), separate `SNOWFLAKE_*` vars for the collab module's Node service (see `sow_docs/reference/ENV.md`) | Same account, different schema per module; see ADR-012 |
| HubSpot | `HUBSPOT_*` | — |
| AWS (S3, Bedrock, ECR/App Runner deploy) | `S3_BUCKET_NAME`, AWS credentials (not an app env var — deploy-time AWS CLI config per the backend README) | Deploy credentials are separate from any app `.env` |
| Anthropic | `ANTHROPIC_API_KEY` (only if not using Bedrock) | — |
| OpenAir, Switchboard, Workato | their respective var groups above | — |

## Local vs. production configuration

- Frontend: `.env.development` vs. `.env.production` (CRA's standard convention) — both currently
  tracked in Git, which is the issue flagged above.
- Backend: no equivalent `.env.development`/`.env.production` split was found — a single `.env`
  (gitignored, not tracked) is implied by the README's setup instructions. Deployment-time values
  are presumably injected by AWS App Runner's own environment configuration, not committed
  anywhere — this was not directly confirmed (no App Runner config file found in `deployment/`
  beyond an IAM policy).
