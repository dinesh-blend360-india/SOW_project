# Repository Map

Three repositories, each with its own Git history and remote. See `LOCAL_DEVELOPMENT_SETUP.md`
for how to clone all three into the layout this document assumes.

```
SOW_project/                       ROOT REPO (this one)
  agent-handoff/                   this documentation system
  sow_docs/                        SOW editor module's tracking/build-plan/reference docs
  documentation/                   narrative docs — MOVED into sow_docs/narrative/, pointer left behind
  blend_ode/                       FRONTEND REPO (git-ignored by root; clone separately)
  blend_ode_be/                    BACKEND REPO (git-ignored by root; clone separately)
```

---

## `blend_ode_be` — backend (FastAPI)

**Purpose**: Serves the Pricing Tool and SOW Studio APIs, owns the Snowflake connection, issues
the app's own JWTs after a Microsoft SSO exchange, and (new, in progress) hosts the FastAPI side of
the Lexical/Yjs collaboration module plus the module's own Node.js service inside this same repo.

**Remote**: `origin` → `github.com/dinesh-blend360-india/blend_ode_be`, `upstream` →
`github.com/BLEND360/blend_ode_be`. Current dev branch (verified): `dinesh_thamill_blend360india_sow_editor`.

**Directory structure** (verified):
```
blend_ode_be/
  src/
    main.py                    FastAPI app, middleware, router registration
    common/                    config.py (env vars, feature flags), database.py (Snowflake engine),
                                send_mail.py, aws_s3.py, openair_api.py, middlewares.py
    admin/routers.py           /admin/* — exports, pricing-version workflow
    auth/routers.py            /auth/* — MSAL login exchange, user/group/feature RBAC admin
    pricing_tool/              /pricing_tool/* — the original Pricing App (54 endpoints)
      dependencies.py          authorize_token() — the core auth dependency
      crud.py                  get_role_and_features() — the RBAC source of truth
    sow_studio/                /pricing_tool/* — SOW Studio (60 endpoints across 3 files)
      routers.py                intake, draft, review, lifecycle, export (45 endpoints)
      api/native.py             native-editor pilot (pre-Lexical), guest flow (11 endpoints)
      api/public.py             unauthenticated public approval/share pages (4 endpoints)
      native_collaboration.py   native-editor persistence/access logic
      crud.py                   save_sow_draft(), assert_deal_access(), get_latest_sow_draft() —
                                 reused directly by the new sow_module (never copied)
    sow_module/                 the in-progress Lexical/Yjs collaboration module (Python side)
      router.py                 public session endpoint
      router_internal.py        service-key-protected internal endpoints
      collab_session.py          ticket minting, role resolution
      collab_internal.py         resolve-access / document-state / checkpoint logic
      role_table.py              Python mirror of the frontend's role-table.json
      html_allowlist.py          the module's own HTML allowlist (never sanitize_rich_text)
  sow_module/                   tracking/docs/scripts for the module — MOVED to sow_docs/, pointer left
    collab/                      the Node.js "Document Owner" service (Express + ws + Yjs, Vitest)
      src/shared/                 VENDORED copy of blend_ode's src/sow_module/shared/ — never edit by hand
    sync_shared.sh, shared_hash.py, run_checks.sh
  migrations/                   001 through 012, sequential SQL (SOW Studio + sow_module's SOW_COLLAB_*)
  db/migrations/                a SEPARATE, smaller migrations folder (002, 003 only) — UNKNOWN
                                 relationship to migrations/; flagged in KNOWN_ISSUES_AND_BLOCKERS.md
  tests/                        60 pytest files, flat under tests/, plus tests/fixtures/
  deployment/                   IAM policy JSON for App Runner
  Dockerfile, requirements.txt, Pipfile
  .github/workflows/main.yml    manual (workflow_dispatch) build+push+deploy, no test/lint step
```

**Database ownership**: `blend_ode_be` is the only writer to the application's core Snowflake
tables (`SOWS`, `SOW_DRAFT_VERSIONS`, `HUBSPOT_DEALS`, `PRICING_DEALS`, `USERS`, `GROUPS`,
`FEATURES`, etc., in the app's main schema — `SF_DATABASE`/`SF_SCHEMA` env vars, historically
`RESOURCE_REQUEST_TOOL`/`PRICING_TOOL_SANDBOX_HYBRID`). The `sow_module`'s Node service owns a
**separate** set of tables (`SOW_COLLAB_*`) in a different schema
(`BLEND_INTERNAL_IT_APPS.NEXDRAFT`) and must never write the tables above — see
`SYSTEM_ARCHITECTURE.md` and `sow_docs/tracking/decisions.md` ADR-003/012.

**Where to implement what**:
- A new Pricing Tool endpoint/feature → `src/pricing_tool/`.
- A new SOW Studio endpoint/feature (existing CKEditor-era flow) → `src/sow_studio/`.
- Anything touching the Lexical/Yjs live-editor module → `src/sow_module/` (FastAPI side) or
  `sow_module/collab/` (Node side) — read `blend_ode_be/sow_module/CLAUDE.md` first; it has its own
  stricter standing rules (never touch `SOW_NATIVE_*`/`SOWS`/`SOW_DRAFT_VERSIONS` from the Node
  service, bind-variable-only SQL, never apply a migration without explicit approval).
- A new Snowflake migration → `migrations/`, next sequential number; never apply it without asking.

---

## `blend_ode` — frontend (Create React App)

**Purpose**: The browser UI for both the Pricing Tool and SOW Studio, plus the Lexical/Yjs editor (behind a feature flag) and its
shared TypeScript contracts.

**Remote**: `origin` → `github.com/dinesh-blend360-india/blend_ode`, `upstream` →
`github.com/BLEND360/blend_ode`. Current dev branch (verified):
`dinesh_thamill_blend360india_sow_editor`.

**Directory structure** (verified):
```
blend_ode/
  src/
    App.tsx                    routes (react-router-dom v6)
    index.tsx                  entry point; maintenance-mode gate
    config/
      ApiService.ts             axios instances: api, apiToken, sowStudioApi
      AgentsApiService.ts       axios instances for a separate "Blend Agents" service
      MsalConfig.ts             Azure AD client/tenant config (TRACKED — see KNOWN_ISSUES)
    context/                   UserContext, CurrencyContext (React Context, no Redux/Zustand/RQ)
    components/                 shared UI: ProtectedRoute, LoginRoute, ErrorBoundary, AdminPage, ...
    layout/                    AppLayout, MaintenanceLayout
    pricing-tool/
      features/                 route-level pages (homePage, createPricing, versionManagement, ...)
      components/                reusable UI incl. DraftEditor.tsx (CKEditor 5), NativeDraftEditorScreen.tsx
      sow-studio/                newer, layered SOW Studio UI: pages/, model/, steps/, api/, shared/
      hooks/, interfaces/, services/
    sow_module/                 SHARED CONTRACTS for the Lexical/Yjs editor (source of truth)
      shared/                    roles.ts, role-table.json, protocol.ts, errors.ts, bridge.ts,
                                  allowlist.ts, html-contract.ts/.json, SowHighlightNode.ts, intake.ts
      testing/                   drift test, fixture-based fidelity/parity tests
      tsconfig.shared.json       strict, isolated from CRA/DOM runtime
  .env.development, .env.production   TRACKED, contain real secrets — see KNOWN_ISSUES_AND_BLOCKERS.md
  package.json
  .github/pull_request_template.md    the only CI-adjacent file — no actual CI workflow exists
```

**Where to implement what**:
- A new Pricing Tool page/component → `src/pricing-tool/features/` or `components/`.
- A new SOW Studio page/component → `src/pricing-tool/sow-studio/` (the newer, layered structure;
  its own README warns against adding to the older `pricing-tool/components/index.ts` barrel).
- Any change to the Lexical/Yjs shared contract types → `src/sow_module/shared/` **only**, then run
  `blend_ode_be/sow_module/sync_shared.sh` to vendor the change into the backend repo. Never hand-edit
  the vendored copy at `blend_ode_be/sow_module/collab/src/shared/`.
- The actual browser-side Lexical editor component and its WebSocket client do not exist yet —
  this is the next major frontend task for the collaboration module (see `TASK_BOARD.md`).

---

## Root repository (`SOW_project/`)

**Purpose**: Houses this agent-handoff documentation system and the SOW editor module's own
tracking/build-plan/reference docs (`sow_docs/`), without owning or duplicating either app's
source code. Both `blend_ode/` and `blend_ode_be/` are excluded via `.gitignore` (see the root
`.gitignore` for the exact, user-maintained list — it also excludes a few local-only scratch files
and a folder named `project-memory-starter`, which this documentation system does not reference per
explicit instruction, and a folder named `dont-check-others`, whose contents were not inspected).

**Directory structure** (verified, root repo's own tracked content only):
```
SOW_project/
  agent-handoff/              this system
  sow_docs/
    build_plan/                SOW_EDITOR_BUILD_PROMPTS_v1.md, SOW_EDITOR_RUN_GUIDE_v1.md
    tracking/                   prompt_log.md, decisions.md, contracts.md, test_log.md, audits, spike reports
    guides/                     testing_pending_guide.md, ENV_REMINDER.md
    narrative/                   README.md, backend.md, frontend.md (human-readable architecture writeups)
    reference/                   ENV.md (sow_module env var names), sow_module_README.md
  CLAUDE.md                   root-level agent entry point (this task creates it)
  AGENTS.md                   root-level agent entry point for other coding agents (this task creates it)
  .gitignore
```

The root repository was not yet a Git repository as of this task's start; see
`agent-handoff/SESSION_LOG.md` for when and how it was initialized.
