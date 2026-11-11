# Local Development Setup

## Prerequisites

| Tool | Version | Verified how |
|---|---|---|
| Python | 3.11 | `blend_ode_be/Dockerfile` (`python:3.11-slim`); no `.python-version` file, so this is the only pin found |
| Node.js | UNKNOWN — no `.nvmrc`, no `engines` field in either `package.json` | Use a reasonably current LTS (20+) until confirmed; the collab module's `package.json` declares `"engines": {"node": ">=22"}` for that one subproject only |
| Pipenv | required per `blend_ode_be/README.md` | — note: both a `Pipfile` and a `requirements.txt` exist; Docker and (implicitly) CI use `requirements.txt` — which is authoritative for local dev is **unconfirmed**; if `pipenv install` and `pip install -r requirements.txt` disagree, treat `requirements.txt` as the one proven to work (it's what's actually deployed) |
| npm | whatever ships with your Node install | `blend_ode` uses `package-lock.json` (npm), not yarn/pnpm |
| Docker | optional, only needed to build/run the production container image | `blend_ode_be/README.md` |
| Git | — | three separate repositories, see below |

## Repository cloning and folder placement

This root folder is **not itself a dependency** of either app — it only holds documentation. The
two application repos must be cloned as siblings directly inside it:

```bash
# from the parent directory where you want the project to live
git clone <root-repo-url> SOW_project
cd SOW_project

git clone https://github.com/<your-fork-or-org>/blend_ode.git
git clone https://github.com/<your-fork-or-org>/blend_ode_be.git
```

Resulting layout (must match for the root repo's `.gitignore` to correctly exclude them, and for
the SOW module's `sync_shared.sh` and `run_checks.sh` scripts to find both repos by relative path):

```
SOW_project/
    CLAUDE.md
    AGENTS.md
    .gitignore
    agent-handoff/
    sow_docs/
    blend_ode/
    blend_ode_be/
```

Verified remotes for both app repos (yours may differ if you're working from a fork):
- `blend_ode`: `origin` → `blend360-india/blend_ode` fork, `upstream` → `BLEND360/blend_ode`
- `blend_ode_be`: `origin` → `blend360-india/blend_ode_be` fork, `upstream` → `BLEND360/blend_ode_be`

Check out the branch you intend to continue work on in each app repo (verified example branch in
use during this project: `dinesh_thamill_blend360india_sow_editor` — confirm with your own remote
before assuming this branch exists there).

## Dependency installation

### Backend (`blend_ode_be`)
```bash
cd blend_ode_be
pip install -r requirements.txt
# or, per the README: pipenv install
```

### Frontend (`blend_ode`)
```bash
cd blend_ode
npm install
```

### SOW collaboration module's Node service (`blend_ode_be/sow_module/collab`)
```bash
cd blend_ode_be/sow_module/collab
npm install
```
This is a separate Node project from the frontend — it has its own `package.json`, and is not yet
wired into the main app's dev startup (it isn't started by anything else automatically).

## Environment configuration

See `ENVIRONMENT_AND_SECRETS.md` for the full variable list and what each one does — this section
only covers where to put them.

- **Backend**: create `blend_ode_be/.env` (not tracked; a `.env.example` exists in that repo to
  copy from) with the Snowflake, Microsoft SSO, and integration credentials. Ask the project owner
  for real values — none are reproduced in this documentation.
- **Frontend**: `blend_ode/.env.development` already exists and is tracked — **do not add new real
  secrets to it**; it already has a known exposure problem (see `KNOWN_ISSUES_AND_BLOCKERS.md`).
  Until that's remediated, treat any value already in it as compromised/rotatable rather than safe
  to extend.
- **SOW collab module's Node service**: `blend_ode_be/sow_module/collab/.env` (gitignored, copy
  from `.env.example` in that folder) — needs its own Snowflake key-pair credentials, separate from
  the backend's main `.env`. See `sow_docs/reference/ENV.md` for the exact variable names.

## Database setup

No local database setup is needed — both the main application and the SOW collaboration module
connect to shared, hosted Snowflake (no local Snowflake emulator exists or is expected). You need:
1. A Snowflake user with access to the application's main schema (ask for `SF_*` credentials).
2. For SOW collaboration module work specifically: access to `PRICING_APP_DEV_USER` /
   `PRICING_APP_DEV_ROLE` scoped to `BLEND_INTERNAL_IT_APPS.NEXDRAFT` — this is a distinct,
   narrower credential, not the same as the main `SF_*` ones. See
   `sow_docs/tracking/decisions.md` ADR-012.

**Migrations**: live in `blend_ode_be/migrations/` (numbered `001`–`012`) and a separate, smaller
`blend_ode_be/db/migrations/` (`002`, `003` only — relationship between the two folders is
unconfirmed, see `KNOWN_ISSUES_AND_BLOCKERS.md`). **Never apply a migration to Snowflake without
explicit approval from whoever owns the database** — this is a standing rule
(`blend_ode_be/sow_module/CLAUDE.md`), not a suggestion, and it applies project-wide: there is one
shared Snowflake instance, treat it as shared.

## Backend startup

```bash
cd blend_ode_be
uvicorn src.main:app --reload
# or: fastapi dev src/main.py
```
Swagger UI: `http://127.0.0.1:8000/docs` (confirmed in the repo's own README).

## Frontend startup

```bash
cd blend_ode
npm start
```
Dev server: `http://localhost:3000` (inferred from `REACT_APP_MSAL_REDIRECT_URI=http://localhost:3000/callback`
in the tracked `.env.development` — not independently confirmed by running it in this task).

Set `REACT_APP_API_URL` (in `.env.development`, already present) to point at your local backend if
it isn't already `http://127.0.0.1:8000` or equivalent.

## SOW collaboration module's Node service startup (not started by the main app's dev script)

```bash
cd blend_ode_be/sow_module/collab
npm run dev
```
This starts the service in isolation. The browser editor reaches it through the session endpoint, which
returns `COLLAB_WS_PUBLIC_URL`; both must be configured for a real session. The Lexical editor is shown
only with the feature flag (see `ENVIRONMENT_AND_SECRETS.md`). The service is also useful for its own
test suite and its spike scripts (`npm run spike:*`).

## Build and verification commands

| Command | Where | What it does |
|---|---|---|
| `python -m pytest tests/ -q` | `blend_ode_be/` | Full backend test suite (60 files). Verified passing: 555 passed, 9 pre-existing failures unrelated to the SOW module work done so far (date-dependent template tests, a duration-rounding test) — see `KNOWN_ISSUES_AND_BLOCKERS.md`, do not assume a 10th failure is pre-existing without checking |
| `npm test` | `blend_ode/` | Jest via `react-scripts`, interactive watch mode by default |
| `npm run build` | `blend_ode/` | Production build |
| `bash sow_module/run_checks.sh` | `blend_ode_be/` | Runs all SOW-module-specific checks across both repos in one shot: shared-code sync check, frontend Jest (sow_module tests only), frontend shared typecheck, collab service typecheck/lint/Vitest, backend module pytest subset. Verified passing, 7/7, as of the last session in this repo. |
| `npm run typecheck` / `npm run lint` / `npm test` | `blend_ode_be/sow_module/collab/` | TypeScript/ESLint/Vitest for the Node collab service specifically |

No backend lint command/config was found (no `.flake8`/`ruff.toml`/lint section in any config file)
— there is currently nothing to run for backend lint beyond what an IDE might do locally.

## Common setup errors and their solutions

| Symptom | Likely cause | Fix |
|---|---|---|
| Frontend fails to start with a Babel/CKEditor-related error | `prestart` runs `ensure-babel-private-property-plugin.js` and `cleanup-stale-ckeditor-builds.js` to work around a known CKEditor5/Babel toolchain quirk | Let the `prestart` script run (don't skip it with `--ignore-scripts`); if it still fails, check that script's own output for what it expects |
| Backend can't connect to Snowflake | Missing or malformed `SF_PRIVATE_KEY` (must be a PEM, converted internally to DER/PKCS8) | Confirm the key format matches what an existing working `.env` uses; ask the project owner rather than guessing at key format |
| SOW collab module's Node service can't see `BLEND_INTERNAL_IT_APPS.NEXDRAFT` | Using the backend's main `SF_*` credentials instead of the module's own `PRICING_APP_DEV_USER`-scoped ones | Use the separate `.env` in `sow_module/collab/`, not the backend's main `.env` — these are deliberately different credentials (ADR-012) |
| `npm test` in `blend_ode` fails only on `DraftEditor.test.tsx` env-related assertions | That test manipulates `process.env.REACT_APP_CKEDITOR_*` directly, and those vars are not consistently set outside `.env.development` | Confirm you're running from a checkout with `.env.development` present, or isolate the one test |
| A SOW module Snowflake test is skipped, not failing | Integration tests against the real `BLEND_INTERNAL_IT_APPS.NEXDRAFT` schema are gated behind `RUN_SNOWFLAKE_INTEGRATION=1` and real credentials, by design — they never run by default | Set that env var and provide real credentials only if you intend to touch live Snowflake, and only with the care the module's `CLAUDE.md` requires |
