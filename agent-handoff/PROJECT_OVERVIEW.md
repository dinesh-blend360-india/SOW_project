# Project Overview

## What this is

A backend/frontend application pair for Blend360, built around two connected products on a shared
Snowflake database:

1. **Pricing Tool** — creates and manages pricing deals, syncs with HubSpot deals, integrates with
   OpenAir for resourcing/booking data, and gates most actions behind a role/feature-flag system.
2. **SOW Studio** — generates, reviews, and finalizes Statements of Work (SOWs) tied to those
   deals: AI-assisted drafting and risk review ("Deal Desk" agents), a rich-text editor (CKEditor
   today), sign-off/approval workflow, finalize-and-export (DOCX/PDF), and client-facing
   approval/share pages.

A third initiative, in active development as of this writing, is replacing/extending SOW Studio's
editor with a **live, multi-user collaborative editor** (Lexical + Yjs), built as its own module
(`sow_module/` in both repos) rather than inside the existing CKEditor code paths. See
`sow_docs/tracking/decisions.md` and `sow_docs/build_plan/SOW_EDITOR_BUILD_PROMPTS_v1.md` for that
initiative's own detailed plan — this overview only summarizes it; it is not a restatement of this
project's purpose as a whole.

## Problem statement

- Pricing and SOW authoring today involve multiple disconnected tools (HubSpot, a pricing
  spreadsheet-like flow, email-based review, manual document assembly) — this app consolidates
  that into one authenticated internal tool.
- SOWs currently go through single-editor, save-and-reload authoring (CKEditor) with a manual
  review/sign-off process. The in-progress editor migration's problem statement (narrower, scoped
  to that initiative): give multiple Blend people real-time, Google-Docs-style concurrent editing
  on one SOW, while keeping the existing HTML-based save/finalize/export pipeline as the system of
  record (see `sow_docs/tracking/decisions.md` ADR-001/002).

## Proposed solution (high level, verified from code)

- A FastAPI backend (`blend_ode_be`) fronting Snowflake, with Microsoft SSO login exchanged for the
  app's own JWT, and a feature-flag/role system (`FeaturesList`, `USERS`/`GROUPS`/`FEATURES` tables
  in Snowflake) gating almost every endpoint.
- A Create React App frontend (`blend_ode`) using antd for UI, consuming that API.
- AI-assisted SOW drafting and legal/commercial risk review via Anthropic Claude (directly or
  through AWS Bedrock via the `strands-agents` SDK) — the "Deal Desk" agent pipeline.
- The in-progress editor module adds a **separate Node.js "Document Owner" service**
  (Express + `ws` + Yjs) that holds live documents in memory, persists durably to its own
  Snowflake tables (`SOW_COLLAB_*`, schema `BLEND_INTERNAL_IT_APPS.NEXDRAFT`), and talks to FastAPI
  through a small set of internal, service-key-protected endpoints rather than connecting to the
  application's main database directly. See `SYSTEM_ARCHITECTURE.md`.

## Major features (verified from route inventory — see `REPOSITORY_MAP.md`/`INTEGRATION_CONTRACTS.md`)

- HubSpot deal browsing and sync, pricing deal/version CRUD, pricing templates, resource requests,
  OpenAir bookings (Pricing Tool).
- SOW intake, AI draft generation, suggested-version review, risk-flag evaluation, sign-off,
  publish/finalize, DOCX/PDF export, public approval/share pages (SOW Studio).
- A native-editor pilot (pre-Lexical) with its own guest-invitation and checkpoint flow — an
  earlier, narrower experiment than the current Lexical/Yjs module; both exist in the code today.
- Admin exports and a base-price-version workflow; user/group/feature RBAC management (`/admin`,
  `/auth`).
- (In progress) Live multi-user SOW editing via the Lexical/Yjs module.

## Users and use cases

- **Internal Blend360 staff** (sales, delivery, solution leads, SOW authors) — the entire app is
  behind Microsoft SSO; there is no public self-service signup. UNKNOWN beyond this whether there
  are distinct named user populations (e.g. a specific "SOW Writer & Deal Desk" team) beyond what
  `common/send_mail.py`'s template content ("Thanks, SOW Writer & Deal Desk") implies.
- **External clients**, narrowly: only through the public, token-gated `sow-approval`/`sow-share`
  pages (`sow_studio/api/public.py`) — read/approve access to one SOW, not general app access.
  The in-progress editor module's plan also mentions a future external/guest access model (secret
  key, approval-request based) — not built yet (`sow_docs/tracking/decisions.md`).

## Technology choices (verified)

| Layer | Technology |
|---|---|
| Backend framework | FastAPI 0.141.1, Python 3.11 |
| Backend auth | Microsoft Entra ID (MSAL) + the app's own HS256 JWT |
| Database | Snowflake (SQLAlchemy engine for connection pooling; raw SQL via `snowflake-connector-python`'s `DictCursor`, not an ORM query layer) |
| AI | Anthropic Claude, via AWS Bedrock (`strands-agents` SDK) by default, or directly via `ANTHROPIC_API_KEY` |
| Frontend framework | React (Create React App) — see `REPOSITORY_MAP.md` for exact versions once confirmed |
| Frontend UI library | antd |
| Frontend rich-text editor (current) | CKEditor 5 |
| New collaboration module | Lexical (editor framework) + Yjs (CRDT) + a Node.js/Express/`ws` service, vendoring shared TypeScript from the frontend repo |
| Hosting/deploy | Docker containers on AWS App Runner, pushed via ECR, deployed with a manual (`workflow_dispatch`) GitHub Actions job |

## Project scope

In scope today: the Pricing Tool, SOW Studio, and the Lexical/Yjs live-collaboration editor
module (in progress). Not in scope for this handoff system to document as if finished: any part of
the live-collaboration module not yet built — see `CURRENT_PROJECT_STATE.md` and `TASK_BOARD.md`
for exactly what is and isn't done.

## Current development phase

As of 2026-11-11: the Lexical/Yjs editor module has completed CP-00 through CP-39 of its build plan,
including the Owner checkpoint and the flagged screen integration. CP-40 (presence and chunked sync) is
partly done. The module is not deployed, and the two-user manual test has not been run. See
`CURRENT_PROJECT_STATE.md` for the authoritative, continuously-updated status.
