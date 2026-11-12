# SOW Editor Module: Claude Code Build Prompts (Version 4, matched to the real codebase and checked against it)

**Date:** 2026-10-30
**Replaces:** v1, v2 and v3. v4 fixes mismatches found when the prompts were checked against the real repositories: the Python sanitizer drops classes and styles, the two folders are separate git repositories, the swap point is inside `DraftEditorScreen`, and migrations 011 to 013 are missing from the repo.
**Based on:** the combined code of `sow_project/blend_ode` (frontend) and `sow_project/blend_ode_be` (backend), plus the four plan files.
**Prompts:** 75 (CP-00 to CP-74). Deployment block is separate and later.

---

## 1. What the real code changed (read this first)

| Finding in your code | Effect on the plan |
|---|---|
| Frontend is **Create React App + TypeScript + antd + MSAL + Jest**, backend is **FastAPI + Snowflake (connector + SQLAlchemy, key-pair) + pytest** | Module lives in **three places** (Section 3). Tests: Jest (frontend), pytest (backend), Vitest (new Node service) |
| CRA **cannot import files outside `src/`**, and the two repos cannot see each other in CI or Docker | Shared TypeScript lives in `blend_ode/src/sow_module/shared/` (source of truth). `sync_shared.sh` copies it into the Node service and the backend tests, and a hash check fails if a copy drifts |
| Draft content is **HTML** in `SOW_DRAFT_VERSIONS`. Finalize, sign-off, publication, DOCX/PDF export and quality checks read it (CP-04 verifies which of them parse the HTML and how they treat classes and styles) | **HTML stays the system of record.** Lexical/Yjs is the live working state. Saving = a **checkpoint** that exports HTML and calls the existing `save_sow_draft` path |
| An **evaluation "native editor"** already exists (`NativeDraftEditorScreen`, `native_collaboration.py`, tables `SOW_NATIVE_*`): live workspace + explicit checkpoint, comment threads, suggestions, guest invitations, mention emails, audit events | We copy its **checkpoint pattern** and **reuse its guest, mention-delivery and audit tables**. The relationship to it is decision D-B1 (CP-54) |
| Access is `assert_deal_access` on **HubSpot deal fields** (owner, leads, collaborators, pricing contributors) or `FE_ADMIN`; features are `FE_SOW_STUDIO`, `FE_SOW_STUDIO_PILOT`, `FE_SOW_NATIVE_EDITOR`, `FE_ADMIN` | **No host adapter to invent.** The FastAPI bridge calls the real functions. New feature flag `FE_SOW_LEXICAL_EDITOR` |
| `DraftEditor.tsx` (CKEditor) has a fixed props contract (`content` HTML, `onSave`, `comments`, `readOnly`, `hasPendingSignOffs`, `isProvisionalTemplate`, `onDownload`, `onCopyLink`, `addInNavigation`...) | `LexicalDraftEditor` implements the **same props**, so `DraftEditorScreen` swaps it behind the flag |
| Real CKEditor features: bold, italic, underline, strikethrough, highlight (yellow, green, pink, blue markers; red and green pens; a `sowPlaceholder` marker), find and replace, headings, font size (10, 12, 14, default, 18, 24, 32), 9 font families, alignment, lists, **tables with merge cells, properties, column resize**, undo and redo | D18 is answered by the code. **Merged cells are required** (v1 said split them). Find and replace must be built |
| The existing `sanitize_rich_text` (`rich_text.py`) **drops every `class` and `style`**, has no `mark` tag, and is called only by the public approval route (`api/public.py`). CKEditor drafts carry highlight classes, placeholder markers and font styles | **Never run checkpoints through it.** A new module-owned `html_allowlist.py` reads one shared `html-contract.json` (derived from real CKEditor output) and is parity-tested against the TypeScript allowlist. The old sanitizer stays untouched (decision D-B3, CP-05) |
| `blend_ode` and `blend_ode_be` are **two separate git repositories**; `SOW_project` is not a repo; only `blend_ode_be` has CI workflows, a Dockerfile and `deployment/` | Tracking files and the Node collaboration service live in the **backend repo** (`blend_ode_be/sow_module/`). Shared TypeScript is owned by the frontend repo and **vendored** (copied, with a hash check) into the backend repo (decision D-B4) |
| `SowStudioPage.tsx` always renders `DraftEditorScreen`; `NativeDraftEditorScreen` is exported but not routed; `FE_SOW_NATIVE_EDITOR` is used on the frontend only in a test | One swap point, inside `DraftEditorScreen` where `<DraftEditor` is rendered |
| `database.py` connects with `insecure_mode: True` | Reported at CP-69; never copied into the Node service |
| LLM runs in the **Python backend** (Bedrock via Strands). Output becomes **draft HTML** | LLM-into-canvas means **HTML** first. Markdown and text are secondary |
| DOCX/PDF export (`python-docx`, `reportlab`), versions, finalize, sign-off, publication, notifications already exist | v1 phases 7 to 10 shrink to **compatibility and lock integration**, not rebuilds |
| Migrations are plain SQL in `blend_ode_be/migrations/NNN_*.sql`, applied manually. The repo stops at **010**; code refers to `013_sow_native_editor.sql`; **011 to 013 and the DDL of the `SOW_NATIVE_*` tables are not in the repo** | CP-03 asks you where they are and for the real table definitions. CP-16 asks you to confirm the migration number before writing. **No migration is applied to the shared database without your OK** |
| Backend docs and config mention **App Runner** (confirmed in `README_SOW_STUDIO_*`) | Collaboration needs WebSockets. Deployment block must verify that App Runner supports them, otherwise use ECS |

---

## 2. How to use this file

- Run prompts **in order**, each in a **fresh Claude Code session** (`/clear`). Every prompt reads `sow_docs/tracking/prompt_log.md`.
- **Open VS Code at `sow_project/`** (the folder holding both `blend_ode` and `blend_ode_be`).
- Paste from `BEGIN PROMPT` to `END PROMPT`.
- Each prompt ends **READY FOR NEXT PROMPT** or **STOP AND TEST**. At STOP AND TEST: run the checks, add a row to `sow_docs/tracking/test_log.md`, make the git commit, continue.
- **Claude Code never commits.** It only suggests the message.
- Model: strongest available for Phases B, D, E. A lighter model is fine for Phases G, H, J.
- `blend_ode` and `blend_ode_be` are **two separate git repositories** (`SOW_project` is not a repo). Each STOP AND TEST says which repos changed; commit in each repo that changed, with the same message.

## 3. Where things live

```
sow_project/                           (not a git repo)
  blend_ode/                           FRONTEND REPO
    src/sow_module/                    NEW. React: LexicalDraftEditor, plugins, panels
      shared/                          SOURCE OF TRUTH: allowlist, html-contract.json, node schema, roles,
                                       protocol, fixtures (pure TS, no CRA imports)
  blend_ode_be/                        BACKEND REPO
    src/sow_module/                    NEW. FastAPI: collab router, internal endpoints, html_allowlist.py
    sow_module/                        NEW. Tracking, docs, Node collaboration service
      (moved) sow_docs/tracking/  prompt_log.md  test_log.md  (+ reports)
      collab/                          Node + Express + ws + Yjs "Document Owner" (TypeScript, Vitest)
        src/shared/                    VENDORED copy of the frontend shared/ (by sync_shared.sh, hash-checked)
      sync_shared.sh  run_checks.sh  CLAUDE.md  README.md
    tests/fixtures/sow_module_html/    VENDORED fixtures for pytest
    migrations/                        new SOW_COLLAB_* SQL (number confirmed at CP-16)
```

Why vendoring: the two repositories are separate, so the backend repo's CI and Docker build cannot read the frontend repo. The frontend owns the shared code; everything else holds a copy that a hash check keeps honest.

## 4. Standing rules (copied into `blend_ode_be/sow_module/CLAUDE.md` by CP-00)

- TypeScript strict, no `any` without a written reason. Python follows the style already in `blend_ode_be`.
- **HTML is the system of record.** The only path into `SOW_DRAFT_VERSIONS` is the existing `save_sow_draft` logic, called by the FastAPI internal checkpoint endpoint.
- The Node collab service writes **only** to `SOW_COLLAB_*` tables. It never touches `SOWS`, `SOW_DRAFT_VERSIONS`, `HUBSPOT_DEALS` or `SOW_NATIVE_*`.
- All SQL uses **bind variables**. No string-built SQL.
- Do not edit existing CKEditor or native-editor files except at the swap points named in a prompt. Never delete them before CP-74.
- No secrets in code or in the browser. Only `.env.example` files are created.
- `shared/` imports nothing from CRA, antd, Node or the DOM.
- Edit shared code **only** in `blend_ode/src/sow_module/shared/`, then run `sync_shared.sh`. Never edit a vendored copy by hand.
- The existing `sanitize_rich_text` is never used on checkpoints and never edited; the module uses its own `html_allowlist.py`.
- **Never apply a SQL migration to Snowflake without explicit user approval (CP-17).** There is one database; treat it as shared.
- If a fact is missing, stop and ask the user. Do not guess.
- Append to tracking files. Never overwrite them.

## 5. `prompt_log.md` entry format (every prompt appends one)

```
## <CP-ID>: <title> (<ISO date>)
Summary: <2 to 4 lines>
Decisions/assumptions: <bullets or none>
Registry (new or changed):
- file: <path>: <purpose>
- class / function / component: <name> in <path>: <purpose>
- endpoint: <METHOD path>: <purpose>
- table / migration: <name>: <purpose>
- feature: <F-ID or name>: <state>
Open questions for the user: <bullets or none>
Handoff to next prompt: <one line>
```

## 6. Phase map

| Phase | Name | Prompts |
|---|---|---|
| A | Readiness: scaffold, audits, contracts, tooling | CP-00 to CP-07 |
| B | Spikes (blocking) | CP-08 to CP-15 |
| C | Backend bridge: tables, data layer, FastAPI endpoints, Node skeleton | CP-16 to CP-21 |
| D | Editor core and checkpoint integration | CP-22 to CP-39 |
| E | Collaboration hardening and scale | CP-40 to CP-47 |
| F | Content intake: LLM, app data, legacy documents | CP-48 to CP-53 |
| G | Mentions, comments, suggestions, guests | CP-54 to CP-61 |
| H | Versions, finalize lock, export parity, audit | CP-62 to CP-66 |
| I | Hardening | CP-67 to CP-71 |
| J | Cutover tooling | CP-72 to CP-74 |

---

# PHASE A: Readiness (CP-00 to CP-07)

---

## CP-00: Scaffold the module, tracking files and rules

**BEGIN PROMPT**

**From previous prompt:** none. This is the first prompt. The workspace root is `sow_project/`, which contains two separate git repositories: `blend_ode` (frontend) and `blend_ode_be` (backend).

**Goal:** Create the module folders and the tracking system and record verified repository facts. Write no feature code.

**Tasks:**
1. Inspect both repositories. Report for each: current branch and whether the working tree is clean. Frontend: package manager and lockfile, and the exact versions of React, TypeScript, `react-scripts`, antd, `ckeditor5` and `@ckeditor/ckeditor5-react` from `package.json`. There is no `.nvmrc`, so **ask the user which Node version they run** and record it. Backend: Python version (ask if not stated), the FastAPI version in `requirements.txt`, what is in `.github/workflows`, the `Dockerfile` and `deployment/`, and the highest numbered file in `migrations/` and `db/migrations/`.
2. Verify these facts and correct any that are wrong: frontend is Create React App with TypeScript, antd and MSAL, tested with Jest; CKEditor 5 is used in `src/pricing-tool/components/DraftEditor.tsx`; `SowStudioPage.tsx` renders `DraftEditorScreen`; backend is FastAPI with Snowflake through `snowflake-connector-python` and SQLAlchemy with key-pair auth in `src/common/database.py`; feature flags are in `src/common/config.py`; the SOW id convention is `sow-<deal_id>`.
3. Ask the user to confirm this placement (the default): tracking, docs and the Node collaboration service live in the **backend repo** under `blend_ode_be/sow_module/` (it has CI, a Dockerfile and deployment config); the frontend component and the shared TypeScript live in the frontend repo under `blend_ode/src/sow_module/`. Record the answer. Do not create a git repo at `sow_project`.
4. Create:
   ```
   sow_docs/tracking/prompt_log.md
   sow_docs/tracking/test_log.md
   blend_ode_be/sow_module/CLAUDE.md
   blend_ode_be/sow_module/README.md
   blend_ode_be/sow_module/collab/          (with a .gitkeep)
   blend_ode_be/src/sow_module/             (with an __init__.py)
   blend_ode/src/sow_module/shared/         (with a .gitkeep)
   ```
5. `CLAUDE.md` must contain: the standing rules and the log-entry format from the top of the prompt file (paste them in), the layout from the prompt file (two repos, vendoring), the repo facts from tasks 1 and 2, the rule that CRA cannot import outside `src/`, and the rule that the existing `sanitize_rich_text` is never used or edited.
6. `test_log.md`: header and an empty table with columns Date, CP-ID, What was tested, Result, Notes, Commit hash (frontend), Commit hash (backend).
7. `prompt_log.md`: header and the CP-00 entry. Record the repo facts and the placement answer under "Decisions/assumptions".

**Constraints:**
- Create nothing outside the listed paths. Install nothing. Do not edit existing application files.
- Mark anything you cannot verify as "UNKNOWN: ask user".

**Expected outcome:** The folder skeleton and tracking files exist, and the log holds verified repository facts.

**Acceptance checks:**
- All listed paths exist. `CLAUDE.md` includes the rules, the log format and the layout.
- The CP-00 entry states every version found and the placement decision.

**Logging:** Append the CP-00 entry.

**Next prompt will:** audit how CKEditor and the draft screen work today.

**Closing line:** Print the verified facts and UNKNOWNs, then print exactly: **STOP AND TEST: checkpoint CP-00: check the folders and tracking files, then commit in the backend repo (tracking and skeleton) and the frontend repo (the `shared/` placeholder).** Suggested commit message: `chore(sow_module): scaffold module folders and tracking`.

**END PROMPT**

---

## CP-01: Audit the CKEditor draft editor and the parity table

**BEGIN PROMPT**

**From previous prompt:** the scaffold and the repo facts are in `prompt_log.md` and `blend_ode_be/sow_module/CLAUDE.md`; read both first.

**Goal:** Document exactly what the current CKEditor component offers, so `LexicalDraftEditor` can match it feature by feature. Read-only; change no application file.

**Tasks:**
1. Read `blend_ode/src/pricing-tool/components/DraftEditor.tsx` fully. Record the complete props contract and the CKEditor plugin list and toolbar order.
2. Record every format option: font sizes (10, 12, 14, default, 18, 24, 32), the 9 font families, every highlight option and its HTML class (`marker-yellow`, `marker-green`, `marker-pink`, `marker-blue`, `pen-red`, `pen-green`, `sow-studio-placeholder-marker`), table features (merge cells, table and cell properties, column resize), find and replace.
3. Record the SOW-specific behavior: the placeholder-marker auto-unwrap rule, the watermark and logo, the provisional banner, the read-only mode and label, the Save, Download (doc, pdf) and Copy link buttons, the unsaved-changes indicator, and `addInNavigation`.
4. Read `__tests__/DraftEditor.test.tsx` and note how CKEditor is mocked and which behaviors the tests protect.
5. Start `sow_docs/tracking/audit_frontend.md` with: the props table, and the parity table with columns CKEditor feature, HTML it produces, Lexical plan, Decision (built, approved-loss, new). Leave "TBD" where a product decision is needed.

**Constraints:**
- Read-only. Give file paths and line ranges for every claim.
- Mark anything unclear as "not found in code: ask user".

**Expected outcome:** `audit_frontend.md` part 1 contains a complete props table and parity table.

**Acceptance checks:**
- Every CKEditor plugin and highlight option appears in the parity table.
- Each placeholder, watermark and read-only rule has a file reference.

**Logging:** Append the CP-01 entry; list every TBD under open questions.

**Next prompt will:** audit how `DraftEditorScreen` saves, autosaves and tracks revisions, and find the swap point.

**Closing line:** Print the TBD list, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-02: Audit the draft screen: save, autosave, revisions and the swap point

**BEGIN PROMPT**

**From previous prompt:** `audit_frontend.md` part 1 exists with the props and parity tables; read it and `prompt_log.md` first.

**Goal:** Understand how the editor is used inside the screen, so the Lexical editor can replace it without breaking saving. Read-only.

**Tasks:**
1. Read `DraftEditorScreen.tsx` and its tests. Record how `content`, `comments`, `onSave`, and the revision and version values flow; **how autosave works**; how `expected_revision` and `expected_version` are tracked; how a historical version becomes read-only; how Download and Copy link are implemented.
2. Confirm and record the swap point. Known so far: `sow-studio/pages/SowStudioPage.tsx` renders `DraftEditorScreen` unconditionally (around line 2688); `NativeDraftEditorScreen` is exported from `components/index.ts` but not routed; `SowNativeGuestReview` is routed at `/sow-native-guest` in `App.tsx`; `FE_SOW_NATIVE_EDITOR` appears on the frontend only in a test. Find **how the frontend reads feature flags** (look at `UserContext.tsx`, `AppLayout.tsx` and the `user_details` value the tests put in storage). Record exactly where inside `DraftEditorScreen.tsx` the `<DraftEditor` element is rendered with its props: that is the swap point.
3. Read `services/sowStudioAddInInsertion.ts`, `sowStudioAddInPresence.ts`, `sowStudioIntakeDraftSync.ts`, `sowStudioOpenItems.ts` and `sow-studio/steps/draft/addInNavigation`. Note which of them read or write draft HTML directly (these must keep working against Lexical).
4. Append to `audit_frontend.md`: a text diagram of the save and autosave flow, the swap point with line numbers, the list of services that touch draft HTML, and a risks list (especially two writers saving the same revision).

**Constraints:** Read-only. Cite file paths and line ranges.

**Expected outcome:** `audit_frontend.md` is complete, including the save flow and the swap point.

**Acceptance checks:**
- The autosave and revision flow is described step by step with references.
- The swap point has exact file and line references.

**Logging:** Append the CP-02 entry.

**Next prompt will:** audit the backend: access, drafts, finalize and the native editor.

**Closing line:** Print the risks list, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-03: Audit backend access, drafts, finalize and the native editor

**BEGIN PROMPT**

**From previous prompt:** `audit_frontend.md` is complete; read it and `prompt_log.md` first.

**Goal:** Map the backend functions the new module must reuse or must not break. Read-only.

**Tasks:**
1. In `blend_ode_be/src/sow_studio/crud.py` read `assert_deal_access`, `save_sow_draft`, `get_latest_sow_draft`, `get_sow_draft_version`, `_lock_sow_row_for_draft_write`, `_draft_revision`, and the finalize and status code paths. Record how `SOWS.STATUS` (for example `Finalized`) and versions work, what "update in place" means, and who may finalize or rename.
2. Read `native_collaboration.py`, `api/native.py` and `dependencies.py`. Record the `SOW_NATIVE_*` columns used, the guest invitation and token flow, the mention delivery flow, the audit events, and how `SimpleNamespace` is used to act as a user.
3. Find the real migration files for the native tables. Known so far: `migrations/` stops at 010, `db/migrations/` has only 002 and 003, and `native_collaboration.py` mentions `013_sow_native_editor.sql`. Migrations 011 to 013 and the table definitions of the `SOW_NATIVE_*` tables are **not in the repo**. List the highest number found, then ask the user: do 011 to 013 exist on another branch or were they applied directly in Snowflake, and can they run `SELECT GET_DDL('TABLE', 'SOW_NATIVE_DOCUMENTS')` (and the same for the other `SOW_NATIVE_*` tables) and paste the result, so the real columns are known.
4. Read `common/database.py` (note that it connects with `insecure_mode: True`; report it, do not change it), `common/config.py`, `common/middlewares.py` and `pricing_tool/dependencies.py` (where `authorize_token` lives): how `get_db` works, how JWT auth works, what `current_user` contains, CORS and middleware behavior. Also read `sow_studio/api/public.py` (the unauthenticated approval routes) and note which routes it exposes and what it sanitizes.
5. Start `sow_docs/tracking/audit_backend.md` with: a reuse table (function, what we use it for), a do-not-break list, the who-can-finalize rule, the next free migration number, and open questions.

**Constraints:** Read-only. File paths and line ranges for every claim.

**Expected outcome:** `audit_backend.md` part 1 covers access, drafts, finalize, native tables and the next migration number.

**Acceptance checks:**
- The checkpoint path (`save_sow_draft`) is described with its revision and version rules.
- The next migration number is stated.

**Logging:** Append the CP-03 entry.

**Next prompt will:** audit everything that reads draft HTML and the LLM paths.

**Closing line:** Print the open questions, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-04: Audit HTML consumers and the LLM paths

**BEGIN PROMPT**

**From previous prompt:** `audit_backend.md` part 1 exists; read it and `prompt_log.md` first.

**Goal:** Find every piece of code that depends on the draft HTML, and every place generated text enters the draft. Read-only.

**Tasks:**
1. Read `application/rich_text.py` and record exactly what `sanitize_rich_text` keeps and drops (expected: it drops every `class` and `style` and has no `mark` tag) and where it is called (expected: only `api/public.py`; confirm with a repository search). Then, for each consumer of draft HTML (`save_sow_draft`, `sow_docx_export.py`, `sow_pdf_export.py`, `sow_publication.py`, `draft_quality.py`, `quality_guard.py`, `sow_signoffs.py`, `source_coverage.py`, `quote_grounding.py`), record whether it parses the HTML and how it treats `class`, `style`, `mark`, table attributes, `colspan` and `rowspan`. Find real CKEditor output in the tests (for example `test_sow_docx_export.py`) and record the tags, classes and style properties it contains.
2. Read the LLM paths: `sow_narrative_agent.py`, `continue_draft.py`, `suggested_fix_agent.py`, `draft_extraction.py`. Record their output format and where the frontend inserts the result into the editor.
3. List the tests that exercise these paths (for example `test_sow_http_e2e.py`, `test_native_collaboration.py`, `test_sow_rich_text.py`, `test_sow_docx_export.py`) and how they build a fake connection.
4. Append to `audit_backend.md`: the HTML dependency list, the LLM paths table, the test inventory, and a combined open-questions list.

**Constraints:** Read-only. Cite file paths and line ranges.

**Expected outcome:** `audit_backend.md` is complete.

**Acceptance checks:**
- The HTML dependency list covers export, publication, quality and sign-off code.
- Every LLM path has its output format recorded.

**Logging:** Append the CP-04 entry.

**Next prompt will:** write the contracts, roles and decisions.

**Closing line:** Print the open questions, then print exactly: **STOP AND TEST: checkpoint CP-04: confirm both audit files exist and are complete, then commit.** Suggested commit message: `docs(sow_module): frontend and backend audits`.

**END PROMPT**

---

## CP-05: Contracts, roles, and decisions

**BEGIN PROMPT**

**From previous prompt:** both audit files exist; read `audit_frontend.md`, `audit_backend.md`, and `prompt_log.md` first.

**Goal:** Fix the contracts between the React component, the Node collaboration service, and FastAPI, and record the decisions. Types and documents only.

**Tasks:**
1. In `blend_ode/src/sow_module/shared/` create strict, dependency-free TypeScript: `roles.ts` (viewer, commenter, editor, sow_admin; capabilities; `roleToCapabilities` as a stub for CP-06), `protocol.ts` (WebSocket messages including `DURABLE {upToSeq}`, `CHECKPOINTED {version, revision}`, awareness, re-auth; close codes 4401 bad ticket, 4403 access removed, 4410 locked, 4301 owner hint), `errors.ts`, `html-contract.json` plus a typed loader `html-contract.ts` (tags, classes, allowed style properties and attributes, derived from the **real CKEditor output** found in the audits and tests, not from `rich_text.py`), `bridge.ts` (types of the FastAPI internal calls).
2. Write `sow_docs/tracking/contracts.md`: the session endpoint, the internal endpoints (`resolve-access`, `document-state`, `checkpoint`, `access-changed`), the WebSocket lifecycle, the ticket (30 seconds, bound to sow, user, role, a unique id, signed with a secret separate from `JWT_SECRET`), the service key header for internal calls, and the role mapping (propose: guests are commenters with suggest, Blend participants are editors, those allowed to finalize today are SOW admins; confirm with the user).
3. Write `sow_docs/tracking/decisions.md` with ADRs, each with status Proposed: HTML as system of record; checkpoint through `save_sow_draft`; Node service writes only `SOW_COLLAB_*`; **D-B1** relationship to the native editor evaluation (default: reuse its guest, mention and audit tables, supersede its frontend later); **D-B2** Node collaboration service as a separate deployable (alternative considered: Python-only relay, rejected because validation needs Lexical); memory-first batched durable log; lease with fencing in Snowflake; shared code is owned by the frontend repo (CRA `src`) and vendored into the backend repo (**D-B4**); **D-B3** the module owns its HTML allowlist (`html_allowlist.py`, driven by the same `html-contract.json`) and the existing `sanitize_rich_text` is never used on checkpoints; the tracking files and the Node service live in the backend repo; permission changes by 60-second re-check plus explicit hook for guest revoke.
4. Ask the user to confirm: role mapping, D-B1, D-B3, D-B4, and who may finalize.

**Constraints:**
- `shared` must pass `tsc --noEmit` strict once CP-06 adds the config. No runtime logic besides stubs.
- No secrets in any file.

**Expected outcome:** Frozen-enough contracts and a decisions file that later prompts follow.

**Acceptance checks:**
- `contracts.md` lists every endpoint with who may call it.
- `decisions.md` has all listed ADRs with a status.

**Logging:** Append the CP-05 entry. Record the user's answers under decisions.

**Next prompt will:** set up tooling for the frontend module, the Node service and the backend.

**Closing line:** Print the questions you need answered, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-06: Tooling for the frontend module and shared code

**BEGIN PROMPT**

**From previous prompt:** contracts and decisions are written; read `contracts.md`, `decisions.md` and `prompt_log.md` first.

**Goal:** Make the frontend module and `shared/` buildable and testable, and implement the role table.

**Tasks:**
1. Make `blend_ode/src/sow_module/shared/` type-check under strict mode with a small `tsconfig` that extends the app's config. Confirm it contains no import from React, antd, Node or the DOM.
2. Confirm CRA's Jest runs a test placed inside `src/sow_module/`. Add a small helper `src/sow_module/testing/` for later tests.
3. Implement `roleToCapabilities` for real (a role's capabilities, with `features` able only to remove capabilities). Export the table as JSON from a test so the backend can compare against it later.
4. Jest tests: viewer has no edit or comment; commenter can comment but not edit; editor can edit but not finalize or rename; SOW admin can do all; `features` cannot add a capability.
5. Do not add runtime dependencies yet.

**Constraints:** Do not change existing dependencies or existing files.

**Expected outcome:** `shared/` type-checks and the role table is tested.

**Acceptance checks:** Jest role tests pass; the existing frontend suite is unaffected.

**Logging:** Append the CP-06 entry; register the role table.

**Next prompt will:** set up the Node service, the backend package and the combined check script.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-07: Tooling for the Node service, the backend package and the check script

**BEGIN PROMPT**

**From previous prompt:** the frontend module and shared code are tested; read `prompt_log.md` first.

**Goal:** Make the Node service and the backend package buildable and testable, with in-memory fakes.

**Tasks:**
1. In `blend_ode_be/sow_module/collab/` create `package.json`, a strict `tsconfig` whose `src/shared/` folder is the **vendored copy** of the frontend `shared/` folder (created by `sync_shared.sh`; never edited by hand), Vitest, ESLint, and scripts `typecheck`, `test`, `dev`. Add only `express`, `ws`, `yjs`, `y-protocols`, `zod`, `pino` and dev tooling.
2. Define a `DataStore` interface (documents, updates, snapshots, leases, revocations, audit) in `src/data/store.ts` and an `InMemoryDataStore`, with one round-trip test.
3. Backend: create `blend_ode_be/tests/test_sow_module_smoke.py` that imports `src.sow_module` and passes under the repo's pytest setup.
4. Create `.env.example` for the Node service (variable names only: Snowflake account, role and key path, ticket secret, service key, FastAPI base URL, flush interval) and `blend_ode_be/src/sow_module/ENV.md` listing the backend variables. Do not edit `config.py` yet.
5. Create `blend_ode_be/sow_module/sync_shared.sh`: it copies `blend_ode/src/sow_module/shared/` to `blend_ode_be/sow_module/collab/src/shared/`, and the fixtures and `html-contract.json` to `blend_ode_be/tests/fixtures/sow_module_html/`, then writes a `SHARED.sha256` file in both repos. Add a drift test in each repo that fails when its own copy no longer matches its `SHARED.sha256`. Then create `blend_ode_be/sow_module/run_checks.sh`: it runs the sync check, the frontend module tests, the Node typecheck and tests, and the backend smoke test, then prints a summary.

**Constraints:** No changes to existing dependencies. No secrets. No Snowflake driver yet.

**Expected outcome:** `run_checks.sh` passes across all three places.

**Acceptance checks:** Node typecheck and tests pass; backend smoke test passes; the script prints a clear summary.

**Logging:** Append the CP-07 entry; register the DataStore interface.

**Next prompt will:** begin the spikes by connecting to Snowflake from Node.

**Closing line:** Print the check results, then print exactly: **STOP AND TEST: checkpoint CP-07: run `blend_ode_be/sow_module/run_checks.sh`, record in test_log.md, commit in every repo that changed.** Suggested commit message: `chore(sow_module): contracts and tooling for frontend, collab service and backend`.

**END PROMPT**

---

# PHASE B: Spikes (CP-08 to CP-15)

> These decide whether the Snowflake-only collaboration design works. Spike code goes in `blend_ode_be/sow_module/collab/spikes/`. Spikes touch **only** objects prefixed `SOW_SPIKE_` in a scratch schema, and drop them at the end.

---

## CP-08: Snowflake access and capability spike

**BEGIN PROMPT**

**From previous prompt:** tooling works in all three places; read `prompt_log.md` and `contracts.md` first.

**Goal:** Prove the Node service can connect to the provided Snowflake database, learn what it is allowed to do, and learn whether hybrid tables are available.

**Tasks:**
1. Look at how `blend_ode_be/src/common/database.py` and `config.py` connect (key-pair, `SF_*` variables). Do not print any value. Ask the user for: a Snowflake user and role for the Node service (not the FastAPI one), the warehouse, and the name of a **scratch schema** where the spike may create and drop objects. Ask whether `CREATE SCHEMA` is allowed.
2. Add `snowflake-sdk` to `blend_ode_be/sow_module/collab`. Write `spikes/00-connect.ts` (key-pair auth from environment variables or a key file path) that runs `SELECT CURRENT_VERSION(), CURRENT_ROLE(), CURRENT_WAREHOUSE()` and prints them.
3. Write `spikes/01-capabilities.ts` that, in the scratch schema only: tries `CREATE HYBRID TABLE SOW_SPIKE_PROBE (id STRING PRIMARY KEY, n NUMBER)`, inserts and reads a row, drops it; reports the exact error if hybrid tables are unavailable; checks the largest `BINARY` value that can be stored; checks `TIMESTAMP_TZ`.
4. Confirm the Node role **cannot** read the existing application tables (for example `SOWS`, `HUBSPOT_DEALS`). Report the result. (We want it denied.)
5. Write `sow_docs/tracking/snowflake_findings.md`.

**Constraints:**
- Never print or store credentials. Use bind variables. Drop all `SOW_SPIKE_` objects at the end.
- Do not touch any non-spike table.

**Expected outcome:** A connection works and the findings file answers: hybrid tables yes or no, value-size limit, privileges.

**Acceptance checks:**
- The version query prints. The hybrid-table result is a definite yes or no with the exact error.

**Logging:** Append the CP-08 entry. Record the hybrid-table result: it selects the fallback tier.

**Next prompt will:** measure batched write latency.

**Closing line:** If hybrid tables are unavailable, print "Tier 2 fallback likely needed: confirm with the user." Then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-09: Batched durable-log latency spike (the key risk)

**BEGIN PROMPT**

**From previous prompt:** `snowflake_findings.md` states the hybrid-table result; read it and `prompt_log.md` first.

**Goal:** Measure whether memory-first, batched writes to Snowflake meet a durable-acknowledgment gate. This validates the whole architecture.

**Tasks:**
1. Write `spikes/02-durable-log.ts`: a queue that gathers update rows from many simulated documents and flushes every 500 ms as one multi-row insert into `SOW_SPIKE_UPDATE` (hybrid if available, else standard).
2. Simulate 100 documents with an update every 1 to 2 seconds for 5 minutes. Record enqueue-to-commit latency for every update.
3. Also measure: single-row insert, 100-row and 1,000-row inserts, point read, conditional update, and statements per minute. Write a 5 MB state as chunked `BINARY` rows.
4. Write `sow_docs/tracking/spike_durable_log.md` with p50, p95, p99 durable-ack latency, those numbers, an estimate of warehouse cost per active hour, pass or fail against the proposed gate (**p95 durable ack of 1.5 s or less**), and a recommendation: Tier 1 (as planned), Tier 2 (single owner task, standard tables), or escalate to the user.

**Constraints:**
- Scratch objects only; drop at the end. Do not tune silently: report raw numbers.

**Expected outcome:** Measured latencies and a tier recommendation.

**Acceptance checks:**
- p50, p95, p99 reported as numbers. The 5 MB chunked write succeeds or the failure is explained.

**Logging:** Append the CP-09 entry with the tier recommendation.

**Next prompt will:** spike the single-writer lease with fencing.

**Closing line:** Print the p95 and the recommendation, then print exactly: **STOP AND TEST: checkpoint CP-09: review the latency report with the user. This gates the architecture. Record the decision in test_log.md and commit the spike files.** Suggested commit message: `spike(sow_module): snowflake durable-log latency`.

**END PROMPT**

---

## CP-10: Lease and fencing spike

**BEGIN PROMPT**

**From previous prompt:** the latency report recommends a tier; read `spike_durable_log.md` and `prompt_log.md` first.

**Goal:** Prove a single-writer lease with a fencing token held in Snowflake, with no Redis.

**Tasks:**
1. Write `spikes/03-lease.ts`: table `SOW_SPIKE_LEASE`; conditional acquire; one batched renew statement for all rooms a holder owns; fenced write (`INSERT ... SELECT ... WHERE fencing_token = ? AND expires_at > now`).
2. Race two processes for one document, then for 500 documents. Confirm exactly one holder each.
3. Simulate a paused "zombie" holder that loses its lease, then attempts a fenced write: it must insert zero rows.
4. Kill the holder and measure time until the other acquires. Target 30 s or less.
5. Report in `sow_docs/tracking/spike_lease.md`: acquire and renew latency, failover time, zombie result, pass or fail.

**Constraints:**
- Scratch objects only. If hybrid tables are unavailable, use a standard table and say how behavior differs.

**Expected outcome:** One holder per document, zero zombie writes, failover time measured.

**Acceptance checks:**
- Exactly one holder in both races. Zombie write inserts zero rows.

**Logging:** Append the CP-10 entry.

**Next prompt will:** spike crash durability and the client-held-until-durable rule.

**Closing line:** Print failover time and the zombie result, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-11: Crash durability and client-held edits spike

**BEGIN PROMPT**

**From previous prompt:** the lease spike passed; read `spike_lease.md` and `prompt_log.md` first.

**Goal:** Prove the loss rule: every edit that received `DURABLE` survives an owner crash, and every edit without it is still held by the client and merges exactly once.

**Tasks:**
1. Write `spikes/04-durability.ts`: the flusher from CP-09 plus a simulated client that keeps its updates until it receives `DURABLE {upToSeq}`. Use real Yjs updates so merge idempotency is genuinely tested.
2. Kill the owner repeatedly under continuous edits.
3. After each kill verify: every durably acknowledged update is in Snowflake; every unacknowledged update is still in the client's buffer and, after reconnect, appears exactly once.
4. Report in `sow_docs/tracking/spike_durability.md`: kills, durable-acked loss (must be 0), duplicates (must be 0).

**Constraints:** Scratch objects only.

**Expected outcome:** Zero durable-acked loss and zero duplicates.

**Acceptance checks:** Both counts are 0.

**Logging:** Append the CP-11 entry.

**Next prompt will:** spike HTML to Lexical to HTML fidelity using your real draft HTML.

**Closing line:** Print both counts, then print exactly: **STOP AND TEST: checkpoint CP-11: review CP-09, CP-10, CP-11 together with the user and confirm the tier. Record and commit.** Suggested commit message: `spike(sow_module): lease, fencing, crash durability`.

**END PROMPT**

---

## CP-12: Lexical packages, the allowlist and import/export fidelity

**BEGIN PROMPT**

**From previous prompt:** the collaboration spikes are accepted; read `prompt_log.md`, `audit_backend.md` (HTML dependency list) and `audit_frontend.md` first.

**Goal:** Prove real draft HTML survives import into Lexical and export back to HTML. This starts the decision on whether HTML can stay the system of record.

**Tasks:**
1. Ask the user for at least 20 **sanitized** draft HTML samples from `SOW_DRAFT_VERSIONS.CONTENT` (short, long, table-heavy, with highlights, placeholders, merged cells, lists to depth 3). Put them in `blend_ode/src/sow_module/shared/fixtures/html/`. If none are available yet, synthesize representative ones from the CKEditor config and mark them `synthetic-`. No customer data.
2. Add Lexical packages to `blend_ode` (`lexical`, `@lexical/react`, `rich-text`, `list`, `table`, `html`, `selection`, `utils`, `clipboard`, `mark`, `yjs`, `headless`, all pinned to one version) without changing any existing dependency version. Report peer-dependency compatibility with the app's React (18.3.1) and TypeScript (4.9.5) versions; if Lexical's types do not compile under 4.9.5, stop and report instead of upgrading TypeScript.
3. Write `shared/allowlist.ts` that loads `shared/html-contract.json` (tags, classes such as `marker-*`, `pen-*` and `sow-studio-placeholder-marker`, allowed style properties, `colspan`, `rowspan`, font size and family values, text-align). **Derive the contract from what CKEditor actually emitted in the fixtures, not from `rich_text.py`;** if the fixtures show something the contract lacks, update the JSON and record why. Links: the CKEditor config has no Link plugin, but generated drafts may contain links; check the fixtures and ask the user whether links stay (safe `href` only) or are reduced to text.
4. Write a test that runs headless Lexical over every fixture: HTML to Lexical JSON, JSON to HTML, and compare **semantically** (normalized DOM); also assert the JSON is byte-stable on a second pass. Run it under Jest (jsdom) and in Node. Write the allowlist's output for each fixture to `shared/fixtures/expected/` so the backend parity test can compare against it.
5. Write the first half of `sow_docs/tracking/spike_html_fidelity.md`: per-fixture result and the list of differences (approved-loss or defect).

**Constraints:** Do not edit the Python sanitizer or exporters. Fixtures contain no customer data.

**Expected outcome:** Import and export work on every fixture, with a difference list.

**Acceptance checks:** No defects on text, headings, lists, tables or highlights; merged cells survive.

**Logging:** Append the CP-12 entry; register `allowlist.ts` and the fixtures.

**Next prompt will:** test the same output against the Python sanitizer and exporters and give the GO or NO-GO.

**Closing line:** Print the defect count, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-13: Python parity, hostile inputs and the HTML GO or NO-GO

**BEGIN PROMPT**

**From previous prompt:** import and export work; read `spike_html_fidelity.md` (first half) and `prompt_log.md` first.

**Goal:** Prove the backend still accepts the HTML Lexical produces, give the backend its own matching allowlist, and decide whether HTML stays the system of record.

**Tasks:**
1. Run `sync_shared.sh`, then create `blend_ode_be/src/sow_module/html_allowlist.py` with `sanitize_draft_html(html)`: new code that reads the vendored `html-contract.json` and applies the same rules as the TypeScript allowlist (tags, classes, allowed style properties, `colspan`, `rowspan`, safe `href`). Do **not** use or edit `sanitize_rich_text`.
2. Create `blend_ode_be/tests/test_sow_module_html_parity.py`: for every vendored fixture, `sanitize_draft_html` output must equal the TypeScript allowlist's expected output; the Lexical-produced HTML must run through `sow_docx_export.py` and `sow_pdf_export.py` without error; and call `sanitize_rich_text` and record in the report exactly what it removes (expected: classes and styles). Do not assert equality for `sanitize_rich_text`, because it is lossy by design and used only for the public approval page.
3. Add hostile fixtures (script tag, `javascript:` link, `url()` in a style, depth bomb, giant table, HTML inside markdown) and prove each is neutralized by both the TypeScript allowlist and `sanitize_draft_html`.
4. Complete `spike_html_fidelity.md` with the combined results and a clear **GO or NO-GO for HTML as the system of record**, with the reasons.

**Constraints:** Do not edit `rich_text.py` or the exporters; `html_allowlist.py` is new code. Report differences instead.

**Expected outcome:** A finished fidelity report with a verdict.

**Acceptance checks:** Parity test passes; hostile inputs neutralized on both sides; the report has a verdict.

**Logging:** Append the CP-13 entry.

**Next prompt will:** spike paste and LLM-text intake.

**Closing line:** Print the verdict and the defect count, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-14: Paste and LLM-text intake spike

**BEGIN PROMPT**

**From previous prompt:** the fidelity spike gave a GO; read `spike_html_fidelity.md` and `prompt_log.md` first.

**Goal:** Prove one intake pipeline cleans Word paste, HTML from the backend LLM agents, and markdown or plain text, all through the same allowlist.

**Tasks:**
1. Implement `shared/intake.ts`: `intake({format: 'html'|'markdown'|'text', content})` returning cleaned HTML or Lexical nodes plus a removal report. Markdown through `markdown-it` with raw HTML disabled.
2. Ask the user for 10 sample LLM draft outputs (the HTML the backend narrative agent produces, plus a continued draft and a suggested fix) and 5 Word-paste samples. Else synthesize and mark them.
3. Prove: Word paste is reduced to allowed formatting; blank paragraphs collapse; a removal report lists what was dropped; hostile content (script tags, HTML inside markdown, giant tables) is neutralized; fast-check fuzzing on html and markdown paths.
4. Time a 20-page paste in jsdom (note: not the reference workstation).
5. Write `sow_docs/tracking/spike_intake.md`.

**Constraints:** Use the same `allowlist.ts` as CP-12.

**Expected outcome:** Clean, safe conversion with a removal report and a timing.

**Acceptance checks:** Hostile samples add no script, link beyond the allowed set, or unknown node.

**Logging:** Append the CP-14 entry.

**Next prompt will:** run the 100-client room spike and close Phase B.

**Closing line:** Print the timing and hostile-intake result, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-15: 100-client room spike and Phase B report

**BEGIN PROMPT**

**From previous prompt:** the intake spike passed; read all `spike_*.md` and `prompt_log.md` first.

**Goal:** Measure one room with 100 synthetic clients and produce the GO or NO-GO report that closes Phase B.

**Tasks:**
1. Write `spikes/05-scale.ts`: 100 headless Yjs clients to a minimal owner using the CP-09 flusher and CP-10 lease. Measure broadcast p50, p95, p99, memory, CPU, awareness rate in a steady-typing run.
2. Run two or three owner processes locally; force a socket onto a non-holder and proxy it to the holder; measure the added latency.
3. Write `spike_scale.md` (label it local, not production-shaped).
4. Write `sow_docs/tracking/phaseB_report.md`: all spikes in one page; the chosen tier; gate values for later phases; the re-baseline note (team size and schedule to be confirmed by the user); and a GO or NO-GO.

**Constraints:** Local only. Scratch objects only.

**Expected outcome:** Scale numbers and a signed-off decision document.

**Acceptance checks:** 100-client broadcast latency and proxy-hop latency are reported. `phaseB_report.md` has a clear GO or NO-GO.

**Logging:** Append the CP-15 entry. Record the tier and gates.

**Next prompt will:** create the real `SOW_COLLAB_*` tables as a migration for review.

**Closing line:** Print the recommendation, then print exactly: **STOP AND TEST: checkpoint CP-15: review `phaseB_report.md` with the user and get an explicit GO before Phase C. Record the decision and commit.** Suggested commit message: `spike(sow_module): 100-client scale and phase B report`.

**END PROMPT**

---

# PHASE C: Backend Bridge (CP-16 to CP-21)

> Only start after an explicit GO in `phaseB_report.md`.

---

## CP-16: SOW_COLLAB tables migration (written, not applied)

**BEGIN PROMPT**

**From previous prompt:** Phase B is a GO and a tier is chosen; read `phaseB_report.md`, `audit_backend.md` (next migration number) and `prompt_log.md` first.

**Goal:** Write the SQL for the live-collaboration tables as a normal numbered migration. **Do not run it.**

**Tasks:**
1. First state the migration number you will use and **ask the user to confirm it** (the repo stops at 010, the code mentions 013, and 011 to 013 are unaccounted for; choose a number above every number the user reports, never reuse one). Then create `blend_ode_be/migrations/<NNN>_sow_collab_live_workspace.sql` in the style of the existing numbered migrations. Tables (hybrid or standard according to the chosen tier), all prefixed `SOW_COLLAB_`:
   - `SOW_COLLAB_DOCUMENT` (deal id as key, source version, source revision, status mirror, latest snapshot id, updated on)
   - `SOW_COLLAB_UPDATE` (deal id, seq, update bytes, author email, client id, created on; primary key deal id plus seq)
   - `SOW_COLLAB_SNAPSHOT` (snapshot id, deal id, up to seq, chunk number, state bytes, content html on the first chunk, content hash)
   - `SOW_COLLAB_LEASE` (deal id, holder task id, holder address, fencing token, expires on)
   - `SOW_COLLAB_REVOCATION` (user email, deal id, revoked on, reason)
   - `SOW_COLLAB_IDEMPOTENCY` (key, request hash, response, expires on)
   - `SOW_COLLAB_AUDIT` (insert-only, hash-chain columns), unless `decisions.md` (D-B1) says to reuse `SOW_NATIVE_AUDIT_EVENTS`.
   Every table gets a comment. Use `IF NOT EXISTS` so re-runs are safe.
2. Add a pytest that parses the file and checks every created table starts with `SOW_COLLAB_` and that no statement alters or drops an existing object.

**Constraints:**
- **Do not execute any SQL against Snowflake.**
- No `DROP` or `ALTER` of existing objects.

**Expected outcome:** A reviewable migration file with a guard test.

**Acceptance checks:** The pytest passes. The file mentions no application table except in comments.

**Logging:** Append the CP-16 entry; register every table.

**Next prompt will:** write the grants script and the runbook, and ask for your approval.

**Closing line:** Print the table list, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-17: Grants, runbook and migration approval

**BEGIN PROMPT**

**From previous prompt:** the migration file exists; read it and `prompt_log.md` first.

**Goal:** Write the privileges the Node service needs and the steps to apply the migration safely. Still **do not run anything.**

**Tasks:**
1. Create `blend_ode_be/migrations/<NNN+1>_sow_collab_grants.sql`: a role for the Node service with only the privileges it needs on `SOW_COLLAB_*` (insert, select, update where required, delete only where compaction needs it) and no access to application tables.
2. Write `blend_ode_be/migrations/README_SOW_COLLAB.md`: what each table is for, the order to apply the two files, how to apply (following `README_SOW_STUDIO_STAGING.md`), how to roll back, and what to check afterward.
3. Extend the pytest from CP-16 to check the grants file only mentions `SOW_COLLAB_*` objects.
4. Ask the user to review both SQL files and say clearly **approved** or **changes needed**. Record the answer. If approved, ask whether the user will apply them or wants Claude to give exact commands.

**Constraints:** **Do not execute any SQL against Snowflake.**

**Expected outcome:** Reviewed and approved migration and grants files, with a runbook.

**Acceptance checks:** Pytest passes; the user's decision is recorded.

**Logging:** Append the CP-17 entry with the approval decision.

**Next prompt will:** implement the Node Snowflake data layer, tested in a scratch schema.

**Closing line:** Print the approval status, then print exactly: **READY FOR NEXT PROMPT.** (Do not proceed to apply anything.)

**END PROMPT**

---

## CP-18: Node Snowflake data layer

**BEGIN PROMPT**

**From previous prompt:** the migration files exist and are approved; read them and `prompt_log.md` first. If the user has applied the migration, record that.

**Goal:** Implement `SnowflakeDataStore` behind the `DataStore` interface, tested against a scratch schema.

**Tasks:**
1. Implement `src/data/snowflake-store.ts`: pooled connections, key-pair auth from environment variables, bind variables only, statement timeout, retry on transient errors, per-call timing logs without values. A factory chooses `memory` or `snowflake` from `SOW_STORE`.
2. Implement all `DataStore` operations against the `SOW_COLLAB_*` tables: documents, batched update insert, snapshots in chunks, lease acquire, batched renew and fenced write (from CP-10), revocations, idempotency.
3. Integration tests that create a **scratch schema with copies of the tables** (generated from the migration file), run the same contract tests the in-memory store passes, and drop the schema.
4. Add a lint rule or test that fails if any SQL string is built by concatenation.

**Constraints:** Never touch the real `SOW_COLLAB_*` tables from tests. Never print credentials.

**Expected outcome:** The same contract tests pass for the memory and Snowflake stores.

**Acceptance checks:** Integration tests pass in the scratch schema. The SQL-safety test passes.

**Logging:** Append the CP-18 entry; register the store and each method.

**Next prompt will:** add the FastAPI session endpoint and the feature flag.

**Closing line:** Print the test results, then print exactly: **STOP AND TEST: checkpoint CP-18: run Node typecheck and tests including the scratch-schema integration tests; record and commit.** Suggested commit message: `feat(sow_module): snowflake data layer for collab service`.

**END PROMPT**

---

## CP-19: FastAPI session endpoint, ticket and feature flag

**BEGIN PROMPT**

**From previous prompt:** the data layer works; read `contracts.md`, `audit_backend.md` and `prompt_log.md` first.

**Goal:** Let the frontend ask FastAPI for permission to edit a SOW live and receive a short-lived ticket for the WebSocket.

**Tasks:**
1. Add `FE_SOW_LEXICAL_EDITOR = FeaturesList("FE_SOW_LEXICAL_EDITOR")` next to the existing flags in `src/common/config.py`, plus `COLLAB_TICKET_SECRET`, `COLLAB_SERVICE_KEY`, `COLLAB_SERVICE_URL`, `COLLAB_WS_URL` environment variables (read with defaults that disable the feature when unset).
2. Create `src/sow_module/router.py` with `GET /sows/{sow_id}/collab/session`: dependencies `authorize_token`, feature check (`FE_SOW_LEXICAL_EDITOR` or `FE_ADMIN`), `deal_id_from_sow_id`, `assert_deal_access`. Return `{role, capabilities, ticket, wsUrl, sourceVersion, sourceRevision, status}`.
3. Role mapping per `decisions.md` (Blend participants editor; finalize-capable users SOW admin; guests handled in CP-60). Capabilities come from the same table as the TypeScript `roleToCapabilities`; add a pytest that compares them using a JSON export of the TypeScript table to keep the two in step.
4. Ticket: HMAC-signed, 30 seconds, bound to sow id, user email, role and a unique id. Use a secret separate from `JWT_SECRET`.
5. Register the router in `src/main.py` the same way other routers are registered. Add pytest tests in the existing style (for example how `test_sow_http_e2e.py` builds a client): feature off returns 403, no deal access returns 403, valid user gets a ticket, ticket expires.

**Constraints:**
- Do not change behavior of existing routes. Never log the ticket or secrets.

**Expected outcome:** A working, tested session endpoint behind a feature flag.

**Acceptance checks:** All new pytest tests pass and the existing SOW test files still pass.

**Logging:** Append the CP-19 entry; register the endpoint and the flag.

**Next prompt will:** add the internal endpoints the Node service calls.

**Closing line:** Print the pytest results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-20: FastAPI internal endpoints (resolve-access, document-state, checkpoint)

**BEGIN PROMPT**

**From previous prompt:** the session endpoint works; read `contracts.md`, `audit_backend.md` and `prompt_log.md` first.

**Goal:** Give the Node service the three things only FastAPI can do: re-check access, read the latest draft, and save a checkpoint through the existing draft logic.

**Tasks:**
1. Under `/internal/sow-collab/`, protected by the `X-Collab-Service-Key` header (constant-time compare), excluded from the public OpenAPI schema:
   - `POST .../resolve-access` {sow id, user email}: returns role or none, and the SOW status (so the Owner can lock on `Finalized` or pending sign-offs).
   - `GET .../{sow_id}/document-state`: latest draft HTML, version, revision, status, template, `is_provisional`, whether sign-offs are pending.
   - `POST .../{sow_id}/checkpoint` {html, comments, acting user email, expected version and revision}: calls the existing `save_sow_draft` path (use the `SimpleNamespace` user approach seen in `native_collaboration.py`), runs the HTML through `sow_module.html_allowlist.sanitize_draft_html` (the module's own allowlist; **not** `sanitize_rich_text`, which would strip highlights, placeholders and fonts), returns the new version and revision. On a 409 or 428 return a structured error so the Owner can refresh the source revision and retry once.
2. A function `notify_access_changed(sow_id, email)` that posts to the Node service, for use by existing revoke code (for example `revoke_guest_invitation`). Add it without changing existing return values; failures are logged and never block the original action.
3. Pytest tests: wrong or missing service key is rejected; checkpoint on a finalized SOW is rejected (the existing rule); checkpoint saves and returns a new revision; HTML is sanitized by `sanitize_draft_html` while highlight classes and placeholder markers survive; resolve-access matches `assert_deal_access` for owner, collaborator, stranger and `FE_ADMIN`.

**Constraints:**
- Reuse existing functions. Do not copy their logic. Do not change their signatures.
- The Node service must have no way to write anywhere except through these endpoints.

**Expected outcome:** Internal endpoints tested, with the Node service unable to bypass access rules.

**Acceptance checks:** All new tests pass; `test_sow_http_e2e.py`, `test_native_collaboration.py`, `test_sow_revision_integrity.py` still pass.

**Logging:** Append the CP-20 entry.

**Next prompt will:** build the Node Express skeleton that uses these calls.

**Closing line:** Print the results of the existing and new tests, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-21: Node service skeleton, ticket verification, CI-style checks

**BEGIN PROMPT**

**From previous prompt:** the internal endpoints exist; read `contracts.md` and `prompt_log.md` first.

**Goal:** Stand up the Node service shell: health, logging, ticket verification, the FastAPI client, and automatic safety checks.

**Tasks:**
1. `src/server.ts`: Express app with `/healthz`, pino logging with a correlation id, RFC 9457 error handler, graceful startup. `ws` server on `/collab` attached to the same HTTP server.
2. Ticket verification (signature, expiry, single use per process with a short memory set), Origin allowlist from configuration, close code 4401 on failure.
3. `src/bridge/fastapi-client.ts`: calls to the three internal endpoints with the service key, timeouts and a small circuit breaker. Types come from `shared/bridge.ts`.
4. Safety checks as scripts that can run in whatever CI the repo has (check `.github/` first; if there is no workflow, add the scripts and document how to run them): typecheck, tests, a module-boundary check (the Node service imports shared code only from its own vendored `src/shared`; the frontend `sow_module` never imports collab code; the `SHARED.sha256` drift check passes), and a sentinel-secret test (a fake secret in the environment must appear in no log line or response).
5. Tests for ticket expiry, tampering, wrong origin, FastAPI client timeout.

**Constraints:** No collaboration logic yet. No secret in logs.

**Expected outcome:** A running service that authenticates a ticket and talks to FastAPI, with boundary and secret checks.

**Acceptance checks:** Tests pass. A tampered or expired ticket closes the socket with 4401. The boundary check fails when a forbidden import is added in a throwaway edit, then is reverted.

**Logging:** Append the CP-21 entry.

**Next prompt will:** begin Phase D with the Document Owner (rooms, lease, persist queue).

**Closing line:** Print the results, then print exactly: **STOP AND TEST: checkpoint CP-21: run `blend_ode_be/sow_module/run_checks.sh` and the FastAPI tests; record and commit in each changed repo. This closes Phase C.** Suggested commit message: `feat(sow_module): fastapi bridge, collab service skeleton, safety checks`.

**END PROMPT**

---

# PHASE D: Editor Core and Checkpoint Integration (CP-22 to CP-39)

---

## CP-22: Document Owner: rooms, lease and persist queue

**BEGIN PROMPT**

**From previous prompt:** Phase C is complete; read `phaseB_report.md`, `spike_durable_log.md`, `spike_lease.md` and `prompt_log.md` first.

**Goal:** Turn the spikes into production code for the first half of the Document Owner.

**Tasks:**
1. `src/owner/`: a room manager (one in-memory Yjs doc per open SOW; track clients; close when empty).
2. `LeaseManager`: acquire, one batched renew for all rooms held, fenced writes, release on shutdown (from CP-10).
3. `PersistQueue`: gather updates across rooms, flush every 500 ms (configurable) as one multi-row insert, assign per-document `seq`, then send `DURABLE {upToSeq}` to the originating clients (from CP-09 and CP-11).
4. Tests re-proving, against production code: durable-acknowledged edits survive a crash, and a paused "zombie" owner cannot persist.

**Constraints:**
- The Owner is the only writer of live content. All writes are fenced by the lease token.
- The tier from Phase B decides table types and whether more than one owner task runs.

**Expected outcome:** An Owner that holds rooms, persists in batches and acknowledges durability.

**Acceptance checks:** Durable edits survive a crash; a zombie cannot persist.

**Logging:** Append the CP-22 entry; register the classes.

**Next prompt will:** add snapshots, reopening and seeding from the latest draft.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-23: Document Owner: snapshots, reopening and seeding

**BEGIN PROMPT**

**From previous prompt:** rooms, lease and the persist queue work; read `prompt_log.md` first.

**Goal:** Make the Owner able to snapshot, reopen a document, and seed a new live document from the latest draft HTML.

**Tasks:**
1. Snapshots: write chunked Yjs state plus the current HTML together with `up_to_seq`. Open path: load the latest snapshot plus the tail of updates.
2. **Seeding:** if a SOW has no live state, call FastAPI `document-state` and seed the Yjs document from the latest draft HTML through intake. Use a conditional write so two simultaneous opens seed once. Record the source version and revision.
3. Tests: a reopened document reconstructs from snapshot plus tail; two concurrent first opens seed exactly once; seeding does not modify any existing data.

**Constraints:** Seeding is read-only toward the existing draft tables.

**Expected outcome:** Reliable reopen and seed-once.

**Acceptance checks:** All three tests pass.

**Logging:** Append the CP-23 entry.

**Next prompt will:** add the WebSocket protocol with role enforcement and the validation boundary.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-24: WebSocket protocol, roles and validation boundary

**BEGIN PROMPT**

**From previous prompt:** the Owner works; read `contracts.md`, `shared/protocol.ts` and `prompt_log.md` first.

**Goal:** Connect clients to the Owner with role-correct behavior and server-side validation of every update.

**Tasks:**
1. Implement the protocol: join, initial sync, update, awareness, re-auth, `DURABLE`, `CHECKPOINTED`, close codes.
2. On connect: verify ticket, check Origin, check `SOW_COLLAB_REVOCATION`, call `resolve-access` (cached briefly) to confirm the role. Map role to mode: viewer view, commenter comment, editor and SOW admin edit. A view-mode client that sends a content update has its socket closed.
3. Validation boundary: apply the shared `allowlist.ts` to every incoming update before it is applied or broadcast. Reject unregistered nodes, unsafe links, bad classes, depth bombs, oversized payloads.
4. Re-auth on a live socket when a new ticket arrives.
5. Integration tests with real Yjs clients: malicious update rejected before broadcast; view-mode update closes the socket; re-auth works.

**Constraints:** One allowlist module only. Set `maxPayload`. Reject malformed frames.

**Expected outcome:** Authenticated, role-correct sockets and safe updates.

**Acceptance checks:** Malicious updates never reach other clients. Viewer cannot edit even with a modified client.

**Logging:** Append the CP-24 entry.

**Next prompt will:** build the React component shell and collaboration provider.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-25: LexicalDraftEditor shell with the same props as DraftEditor

**BEGIN PROMPT**

**From previous prompt:** the Owner accepts authenticated sockets; read `audit_frontend.md` (props table and swap point) and `prompt_log.md` first.

**Goal:** Create `LexicalDraftEditor` with the **same props as `DraftEditor`** and a working live connection, not yet wired into any screen.

**Tasks:**
1. In `blend_ode/src/sow_module/` create `LexicalDraftEditor.tsx` exporting a component with the `DraftEditorProps` contract from the audit. Keep antd for buttons and alerts so it looks like the rest of the app.
2. A `useCollabSession` hook: calls `GET /sows/{sow_id}/collab/session` through the existing axios setup (look at `config/ApiService.ts`), holds the ticket in memory only, exposes role, capabilities, and status.
3. A collaboration provider using `y-protocols` over the WebSocket, `y-indexeddb` for local persistence, and durable-ack tracking (keep local updates until `DURABLE`). Use the Yjs UndoManager scoped to the local user; no `HistoryPlugin`.
4. Lexical composer with paragraph and heading nodes only, an error boundary, a loading state with Retry (reuse the props `isLoading`, `error`, `onRetry`).
5. Lazy-load the component with `React.lazy` so Lexical stays out of the main bundle. Record bundle size before and after.
6. Jest tests with a mocked socket; a manual two-browser test note in the log.

**Constraints:**
- No existing file changes. CSS classes prefixed `sowx-`. No secrets in the browser besides the ticket.

**Expected outcome:** Two browsers can type paragraphs together in a test harness page.

**Acceptance checks:** Jest tests pass. Local edits persist across refresh. The existing `DraftEditor` tests still pass.

**Logging:** Append the CP-25 entry; register the component, hook and provider.

**Next prompt will:** add the toolbar and formatting that match CKEditor.

**Closing line:** Print the results and the bundle sizes, then print exactly: **STOP AND TEST: checkpoint CP-25: run frontend tests and open two browsers on the harness; record and commit.** Suggested commit message: `feat(sow_module): LexicalDraftEditor shell with live collaboration`.

**END PROMPT**

---

## CP-26: Toolbar framework and basic formatting

**BEGIN PROMPT**

**From previous prompt:** the shell syncs; read the parity table in `audit_frontend.md` and `prompt_log.md` first.

**Goal:** Build the toolbar in CKEditor's order, driven by capabilities, with the basic formatting controls.

**Tasks:**
1. Toolbar items in the CKEditor order: bold, italic, underline, strikethrough, highlight, find and replace, heading, font size, font family, alignment, bulleted list, numbered list, undo, redo. Highlight, find and replace, font controls and lists are placeholders until their prompts; they must be visible but disabled with a tooltip.
2. Implement bold, italic, underline, strikethrough, headings H1 to H3 and alignment (left, center, right, justify), with pressed states following the selection.
3. Undo and redo with the Yjs UndoManager; shortcuts Ctrl or Cmd+Z, Ctrl or Cmd+Shift+Z and Ctrl+Y.
4. Capability gating: edit-only controls are hidden for viewers and commenters.
5. Accessibility: keyboard-operable toolbar, accessible names, focus order.
6. Jest and component tests for each control and for keyboard-only use.

**Constraints:** Do not change the existing SCSS; add styles under `sowx-`. Every format must pass the allowlist.

**Expected outcome:** A working basic toolbar.

**Acceptance checks:** Viewer sees no edit controls. Axe shows no serious toolbar issues.

**Logging:** Append the CP-26 entry.

**Next prompt will:** add highlight, font size and font family with CKEditor-identical classes.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-27: Highlight, font size and font family parity

**BEGIN PROMPT**

**From previous prompt:** the toolbar framework and basic formatting work; read `audit_frontend.md` (format options) and `prompt_log.md` first.

**Goal:** Reproduce the CKEditor highlight and font options so the saved HTML matches what the existing CSS and exports expect.

**Tasks:**
1. Font size options 10, 12, 14, default, 18, 24, 32 and the 9 font families, as the exact allowlist values.
2. Highlight options: yellow, green, pink and blue markers; red and green pens; the `sowPlaceholder` marker. Each must serialize to the **same HTML classes** CKEditor produces (`marker-yellow`, `pen-red`, `sow-studio-placeholder-marker`, and so on) so existing styles and exports keep working.
3. Each control shows the current value from the selection and is hidden for read-only roles.
4. Tests: every option serializes to the expected class or value; round trip through the fixtures; values outside the allowlist are impossible to enter, including by paste.

**Constraints:** Do not rename classes. Do not change the existing SCSS.

**Expected outcome:** Highlight and font controls that match CKEditor's output.

**Acceptance checks:** Class names match the audit table exactly. Fixture round trips are unchanged.

**Logging:** Append the CP-27 entry.

**Next prompt will:** add lists to three levels.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-28: Lists to three levels

**BEGIN PROMPT**

**From previous prompt:** formatting parity works; read `prompt_log.md` first.

**Goal:** Add bulleted and numbered lists limited to three levels with keyboard level control.

**Tasks:**
1. Lists with Tab and Shift+Tab level changes; deeper items flatten to level 3 on input and on paste.
2. The depth cap is enforced in the editor and in `allowlist.ts` the same way.
3. Round trip lists through HTML using the fixtures with nested lists.
4. Tests: depth cap by keyboard and by paste; list HTML matches what CKEditor-saved fixtures contain.

**Constraints:** Same depth rule on client and server.

**Expected outcome:** Three-level lists that round-trip.

**Acceptance checks:** Cannot exceed depth 3. Fixture lists survive import and export.

**Logging:** Append the CP-28 entry.

**Next prompt will:** implement tables with merge cells and column resize.

**Closing line:** Print the results, then print exactly: **STOP AND TEST: checkpoint CP-28: run the frontend tests for formatting and lists; record and commit.** Suggested commit message: `feat(sow_module): formatting and list parity`.

**END PROMPT**

---

## CP-29: Tables: structure, operations and column resize

**BEGIN PROMPT**

**From previous prompt:** lists work; read the table notes in `audit_frontend.md`, `spike_html_fidelity.md` and `prompt_log.md` first.

**Goal:** Build the table structure and the everyday table operations.

**Tasks:**
1. Table, row and cell nodes; an insert picker; row and column insert and delete (with a delete confirmation); Tab navigation; header row; column resize.
2. Column widths stored in the node so they persist through Yjs and HTML export.
3. No nested tables. Cell content is paragraphs and lists.
4. Accessible table header semantics.
5. Tests: widths survive reload and a second synced client; keyboard navigation; delete confirmation.

**Constraints:** Allowed attributes and styles come only from `allowlist.ts`.

**Expected outcome:** Everyday table editing that survives collaboration.

**Acceptance checks:** Column widths persist across reload and between clients.

**Logging:** Append the CP-29 entry.

**Next prompt will:** add merge and split cells, table and cell properties, and export parity.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-30: Tables: merged cells, properties and export parity

**BEGIN PROMPT**

**From previous prompt:** the table basics work; read `audit_frontend.md`, `spike_html_fidelity.md` and `prompt_log.md` first.

**Goal:** Match CKEditor's advanced table features and prove the exporters accept the result.

**Tasks:**
1. **Merge and split cells** (`colspan`, `rowspan`).
2. The table and cell properties CKEditor offered: look at the HTML attributes and styles CKEditor produced in the fixtures and support exactly those.
3. Merged cells and properties survive reload, a second synced client and HTML round trip.
4. Extend the parity test in `blend_ode_be/tests/test_sow_module_html_parity.py` with table-heavy fixtures so `python-docx` and `reportlab` exporters handle the Lexical output.
5. Tests for merges, properties and the export parity.

**Constraints:** Allowed attributes come only from `allowlist.ts`.

**Expected outcome:** Tables with merges and properties that survive collaboration and export.

**Acceptance checks:** Merges and properties persist; export parity passes.

**Logging:** Append the CP-30 entry.

**Next prompt will:** build the paste and drop pipeline into the editor.

**Closing line:** Print the results, then print exactly: **STOP AND TEST: checkpoint CP-30: run frontend tests and the backend parity test; record and commit.** Suggested commit message: `feat(sow_module): formatting parity, lists, tables with merge and resize`.

**END PROMPT**

---

## CP-31: Paste and drop pipeline in the editor

**BEGIN PROMPT**

**From previous prompt:** tables work; read `shared/intake.ts`, `spike_intake.md` and `prompt_log.md` first.

**Goal:** Wire the intake pipeline into paste and drop, with a removal message and server re-validation.

**Tasks:**
1. Paste and drop go through `intake`. Plain-text paste (Ctrl or Cmd+Shift+V) bypasses formatting. Blank paragraphs collapse.
2. Show a removal message (using antd `message`) when content was dropped, with details for SOW admins.
3. One paste is one undo step.
4. The Owner re-validates pasted content server-side with the same allowlist.
5. Tests: Word paste cleaned; plain-text paste; one undo; server blocks a crafted bypass; 20-page timing recorded.

**Constraints:** Same allowlist on client and server.

**Expected outcome:** Safe, fast paste with a clear message.

**Acceptance checks:** Word paste yields only allowed formatting; the server rejects what the client missed.

**Logging:** Append the CP-31 entry.

**Next prompt will:** add placeholders, find and replace, and block IDs.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-32: Placeholders and find and replace

**BEGIN PROMPT**

**From previous prompt:** paste works; read the placeholder rules in `audit_frontend.md` and `prompt_log.md` first.

**Goal:** Port the SOW placeholder behavior and build find and replace.

**Tasks:**
1. **Placeholders:** `sowPlaceholder` highlighted text such as `[Client name]` stays marked while either bracket remains; when the author removes both brackets the marker is unwrapped automatically (the rule in `clearCompletedPlaceholderHighlights`). The HTML class stays `sow-studio-placeholder-marker`.
2. **Find and replace:** a plugin and small panel (find, next, previous, replace, replace all, match case) that works on the local document and produces normal undoable edits. Read-only users can find but not replace. Enable the toolbar button from CP-26.
3. Tests for each rule, including a placeholder inside a table cell, matching the existing placeholder tests in `DraftEditor.test.tsx`.

**Constraints:** Do not change the placeholder class name or the bracket rule.

**Expected outcome:** Placeholders and find and replace behave as in CKEditor.

**Acceptance checks:** Placeholder unwrap rule matches the existing tests; replace all is one undo step.

**Logging:** Append the CP-32 entry.

**Next prompt will:** add stable block IDs.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-33: Stable block IDs

**BEGIN PROMPT**

**From previous prompt:** placeholders and find and replace work; read `prompt_log.md` first.

**Goal:** Give every block a stable identity that later features (comments, add-ins, scroll to block) rely on.

**Tasks:**
1. Every block node carries an explicit `id` in its data (not a Lexical runtime key).
2. IDs are stable through edit, reload, sync, paste and checkpoint-then-reopen.
3. Decide with the user whether IDs appear in exported HTML (for example as a `data-` attribute). If they do, confirm `html-contract.json` and `sanitize_draft_html` allow the attribute and that nothing in export or quality code breaks. Record the decision.
4. Property-based tests (fast-check): IDs unique within a document; copy and paste produces new IDs; splitting and merging blocks keeps one of them.

**Constraints:** The allowlist must be updated in the one shared module if an attribute is added.

**Expected outcome:** Reliable block identity.

**Acceptance checks:** All tests pass; the decision is recorded.

**Logging:** Append the CP-33 entry.

**Next prompt will:** port page chrome and read-only behavior.

**Closing line:** Print the results, then print exactly: **STOP AND TEST: checkpoint CP-33: run frontend tests and the backend suite; record and commit.** Suggested commit message: `feat(sow_module): paste pipeline, placeholders, find and replace, block ids`.

**END PROMPT**

---

## CP-34: Page chrome and read-only behavior

**BEGIN PROMPT**

**From previous prompt:** block IDs work; read `audit_frontend.md` and `prompt_log.md` first.

**Goal:** Match the surrounding visuals and read-only behavior of `DraftEditor`.

**Tasks:**
1. Watermark when `hasPendingSignOffs`; the Blend logo (`utils/Blue.png`); the provisional-template banner when `isProvisionalTemplate`.
2. `readOnly` and `readOnlyLabel` for historical versions: no typing, paste, drop or direct updates. A historical version does not open a live room; it shows a static render of the version HTML. The server also refuses updates for read-only sessions.
3. Jest tests for each state, in the style of the existing `DraftEditor.test.tsx`.

**Constraints:** Keep text, labels and layout consistent with the existing editor. The watermark is presentation only and is never stored in content.

**Expected outcome:** The Lexical editor looks like the current one around the text area.

**Acceptance checks:** The existing label strings and states are reproduced; the watermark never appears in saved HTML.

**Logging:** Append the CP-34 entry.

**Next prompt will:** add the action bar, saving state and unload warning.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-35: Action bar, saving state and unsaved-changes warning

**BEGIN PROMPT**

**From previous prompt:** page chrome and read-only work; read `audit_frontend.md` and `prompt_log.md` first.

**Goal:** Match the Save, Download and Copy link controls and make the save state honest.

**Tasks:**
1. Action bar: Save (becomes a **checkpoint request** wired in CP-37), the Download dropdown (doc, pdf) calling `onDownload`, Copy link calling `onCopyLink`, and the unsaved-changes indicator.
2. Saving state: "Saving…" until the server confirms `DURABLE`, then "Saved".
3. Before-unload warning when edits are not yet durable.
4. Tests for each state; compare the labels with the existing editor.

**Constraints:** Keep button text and layout consistent with the existing editor.

**Expected outcome:** The action bar behaves like today's, with honest save status.

**Acceptance checks:** Each state is covered; the warning appears only with non-durable edits.

**Logging:** Append the CP-35 entry.

**Next prompt will:** port add-in insertion and navigation and expose the editor API.

**Closing line:** Print the results, then print exactly: **STOP AND TEST: checkpoint CP-35: run frontend tests; open the harness and compare it side by side with the CKEditor editor; record and commit.** Suggested commit message: `feat(sow_module): page chrome, read-only, action bar and saving state`.

**END PROMPT**

---

## CP-36: Add-in insertion and navigation, editor API

**BEGIN PROMPT**

**From previous prompt:** page chrome works; read `audit_frontend.md` (add-in findings) and `prompt_log.md` first.

**Goal:** Make add-in features work against Lexical and expose the public API of the component.

**Tasks:**
1. Port the operations in `sowStudioAddInInsertion.ts`, `sowStudioAddInPresence.ts` and `addInNavigation` to a Lexical adapter, keeping the existing functions untouched. Insertion at the right section, duplicate detection, scroll to the inserted add-in (`addInNavigation` prop) and clearing feedback.
2. Expose via a ref or `onReady`: `getHtml()`, `setHtml()`, `insertContent({format, content, target})` (cursor, end, after block, replace block), `focus()`, `scrollToBlock()`. `onContentChange(html)` fires only for author edits, never for programmatic ones.
3. Add-in insertion creates one undo step and goes through the allowlist.
4. Jest tests using the existing add-in test fixtures where possible.

**Constraints:** Do not modify the existing add-in services. Add an adapter layer.

**Expected outcome:** Add-ins can be inserted and navigated in Lexical.

**Acceptance checks:** Existing add-in service tests still pass. New adapter tests pass.

**Logging:** Append the CP-36 entry.

**Next prompt will:** wire checkpoint saving and the screen integration behind the flag.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-37: Owner checkpoint to the existing draft system

**BEGIN PROMPT**

**From previous prompt:** the component is feature-complete for editing; read `audit_frontend.md` (autosave and revision flow), `contracts.md` and `prompt_log.md` first.

**Goal:** Make the Owner save live content into the existing draft system, as the single writer.

**Tasks:**
1. Export the live document to HTML with headless Lexical and pass it through the allowlist.
2. Call FastAPI `checkpoint` with the acting user, the source version and the source revision. On 409 or 428 refresh from `document-state` and retry once.
3. After success broadcast `CHECKPOINTED {version, revision}` and update the stored source revision.
4. Checkpoints run on explicit Save and on a quiet-period timer (configurable) from the Owner, **not from each client**. Skip the save when the normalized HTML equals the current draft.
5. Tests with a fake FastAPI: success, conflict then retry, repeated conflict, no-change skip, FastAPI down.

**Constraints:** Never write to `SOW_DRAFT_VERSIONS` except through the `checkpoint` endpoint.

**Expected outcome:** One writer saving live content into the existing system.

**Acceptance checks:** All tests pass; a no-op session creates no revision.

**Logging:** Append the CP-37 entry.

**Next prompt will:** wire the editor into the real screen behind the flag.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-38: Screen integration behind the flag and other writers

**BEGIN PROMPT**

**From previous prompt:** the Owner can checkpoint; read `audit_frontend.md` (swap point and autosave flow) and `prompt_log.md` first.

**Goal:** Show the Lexical editor in the real screen only for flagged users, without breaking saving.

**Tasks:**
1. At the swap point from the audit, which is **inside `DraftEditorScreen.tsx` where `<DraftEditor` is rendered** (`SowStudioPage.tsx` always renders `DraftEditorScreen`, so it needs no change), render `LexicalDraftEditor` instead of `DraftEditor` only when the user has `FE_SOW_LEXICAL_EDITOR` (or `FE_ADMIN` with the flag on), reading the user's features the same way the rest of the frontend does (as found in CP-02). Everything else in `DraftEditorScreen` is unchanged.
2. In Lexical mode the screen's autosave must defer to the Owner (the screen must not also write), and the screen must update its revision state from `CHECKPOINTED`. Describe the exact lines changed in the log.
3. **Other writers:** when intake sync, add-in save or open-items update change the draft through the existing path while a room is open, the Owner must notice the changed source revision at its next checkpoint and reload from `document-state` with a visible notice. Cover with a test.
4. Run the whole existing frontend suite for the draft screen and the backend SOW tests.

**Constraints:** With the flag off, behavior must be identical to today.

**Expected outcome:** A flagged user edits live and Save creates a normal revision the rest of the app understands.

**Acceptance checks:** Existing `DraftEditorScreen` tests pass unchanged; a checkpoint appears in the version history.

**Logging:** Append the CP-38 entry.

**Next prompt will:** freeze the Core API and run the Phase D regression.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-39: Core API freeze and Phase D regression

**BEGIN PROMPT**

**From previous prompt:** the screen integration works behind the flag; read `contracts.md` and `prompt_log.md` first.

**Goal:** Freeze the public surface and prove everything still works.

**Tasks:**
1. Write `sow_docs/tracking/core_api_freeze.md`: the frozen routes, WebSocket messages, component props and close codes, with a version stamp. Later changes need a note in the log.
2. Full regression: the existing frontend suite, the existing backend suite, the module's frontend tests, the Node tests and the backend module tests. List any failure that existed before this project separately.
3. A two-user manual test script with the flag on, written into `phaseD_report.md`.

**Constraints:** Fix regressions; do not edit the tests to make them pass.

**Expected outcome:** A frozen API and a clean regression.

**Acceptance checks:** All suites pass; the freeze file exists.

**Logging:** Append the CP-39 entry.

**Next prompt will:** begin Phase E with presence and chunked sync.

**Closing line:** Print the results, then print exactly: **STOP AND TEST: checkpoint CP-39: run all suites and a two-user manual test with the flag on; record and commit in every changed repo. This closes Phase D and freezes the Core API.** Suggested commit message: `feat(sow_module): checkpoint saving, flagged screen integration, API freeze (phase D complete)`.

**END PROMPT**

---

# PHASE E: Collaboration Hardening and Scale (CP-40 to CP-47)

---

## CP-40: Presence, cursors and chunked initial sync

**BEGIN PROMPT**

**From previous prompt:** Phase D is complete and the Core API is frozen; read `core_api_freeze.md` and `prompt_log.md` first.

**Goal:** Add live presence and large-document loading.

**Tasks:**
1. Awareness: user name (from the session) and colour, live cursors and selections. Awareness stays in memory on the Owner and is never written to Snowflake. Cap at 100 entries per room.
2. Chunked initial sync in 256 KB pieces (configurable).
3. Presence UI (initials or avatars using antd) with an overflow list for many people.
4. Tests: two clients see each other's cursors; a 5 MB fixture syncs.

**Constraints:** Awareness is throttled; no personal data beyond name and colour.

**Expected outcome:** Visible collaborators and reliable loading of large documents.

**Acceptance checks:** Cursors appear across clients; the 5 MB fixture loads.

**Logging:** Append the CP-40 entry.

**Next prompt will:** make the client resilient to disconnects and offline work.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-41: Reconnect, offline editing and saving state

**BEGIN PROMPT**

**From previous prompt:** presence works; read `prompt_log.md` first.

**Goal:** Edits survive disconnects, and the user always knows the save state.

**Tasks:**
1. Reconnect with exponential backoff and jitter; get a fresh ticket from the session endpoint on every reconnect.
2. Offline edits buffer in y-indexeddb and replay exactly once on return (re-verify CP-11 end to end in the browser).
3. Status bar: connection state, "Saving…" until `DURABLE`, last checkpoint time, connected people.
4. Tests: dropped and restored connection loses no edits; offline edit merges once; the unload warning appears with unconfirmed edits.

**Constraints:** The saving state reflects durability, not just delivery.

**Expected outcome:** Reliable offline and reconnect behavior with honest status.

**Acceptance checks:** No loss and no duplicates across disconnects.

**Logging:** Append the CP-41 entry.

**Next prompt will:** implement the Snowflake-outage rule and graceful shutdown.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-42: Snowflake-unreachable rule and graceful shutdown

**BEGIN PROMPT**

**From previous prompt:** reconnect works; read `prompt_log.md` and `phaseB_report.md` first.

**Goal:** Safe behavior when Snowflake is down, and shutdowns that lose nothing.

**Tasks:**
1. Outage rule: keep rooms open and keep broadcasting; take no new leases and open no new rooms; if the lease cannot be renewed before it expires, put that room in read-only rather than risk a second writer; show "Saving paused" in the client.
2. SIGTERM: stop new connections, flush the persist queue, run a final checkpoint for rooms with unsaved changes if FastAPI is reachable, release leases, tell clients to reconnect, exit within a bounded time.
3. Tests: 60-second outage with active editing loses nothing after recovery; read-only only if the lease expired; SIGTERM drains cleanly.

**Constraints:** Never allow two writers; prefer read-only over risk.

**Expected outcome:** A service that degrades safely.

**Acceptance checks:** Outage and drain tests pass.

**Logging:** Append the CP-42 entry.

**Next prompt will:** snapshot compaction and pruning.

**Closing line:** Print the results, then print exactly: **STOP AND TEST: checkpoint CP-42: run the outage and drain tests; record and commit.** Suggested commit message: `feat(sow_module): presence, offline, outage rule, graceful shutdown`.

**END PROMPT**

---

## CP-43: Compaction and pruning

**BEGIN PROMPT**

**From previous prompt:** outage and drain work; read `prompt_log.md` first.

**Goal:** Keep the update log bounded.

**Tasks:**
1. Snapshot after N updates or after idle time (configurable).
2. Compaction deletes updates at or below the latest snapshot's `up_to_seq`, keeping a 24-hour safety tail, as one batched statement that does not block editing.
3. Prove a document reconstructs identically after compaction.

**Constraints:** Never prune beyond the safety tail.

**Expected outcome:** Bounded log with safe reconstruction.

**Acceptance checks:** Post-compaction reconstruction equals pre-compaction content.

**Logging:** Append the CP-43 entry.

**Next prompt will:** route sockets to the lease holder and re-check access.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-44: Socket routing and connection limits

**BEGIN PROMPT**

**From previous prompt:** compaction works; read `spike_scale.md`, `contracts.md` and `prompt_log.md` first.

**Goal:** Serve a socket correctly from any task.

**Tasks:**
1. A task that receives a socket for a document it does not own proxies the WebSocket to the lease holder over the private address; if the holder is unreachable it waits for expiry then acquires. Cache the lease for 5 seconds.
2. If proxying exceeds its latency budget from the spike, close with code 4301 and an owner hint so the client can reconnect.
3. Connection limits per room and per task, taken from the scale spike with headroom, configurable by environment.
4. Tests: a misrouted socket is served through the holder; an over-limit connection is rejected with a clear message.

**Constraints:** In the normal case clients must not notice proxying.

**Expected outcome:** Correct single-owner serving whichever task a socket hits.

**Acceptance checks:** Both tests pass; added proxy latency is reported.

**Logging:** Append the CP-44 entry.

**Next prompt will:** enforce access changes and the finalize lock on live sockets.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-45: Access re-checks, explicit hook and finalize lock

**BEGIN PROMPT**

**From previous prompt:** routing works; read `contracts.md`, `audit_backend.md` (finalize and sign-off findings) and `prompt_log.md` first.

**Goal:** Make permission and status changes take effect on live sockets.

**Tasks:**
1. **Re-check:** every 60 seconds (configurable) each connected user is re-resolved through FastAPI `resolve-access`. Lost access closes the socket with 4403; a changed role updates capabilities live.
2. **Explicit hook:** an authenticated internal `access-changed` endpoint on the Node service (called by FastAPI `notify_access_changed`, for example when a guest invitation is revoked) closes or downgrades sockets within a second and writes a `SOW_COLLAB_REVOCATION` row that other tasks read every 5 seconds as a backstop.
3. **Finalize lock:** when `resolve-access` or `document-state` reports status `Finalized` (or the pending sign-off rule found in the audit), the Owner makes the room read-only, sends close code 4410 to editors and refuses further updates.
4. Offline edits from a user whose role dropped are rejected on reconnect with a clear message.
5. Tests for each rule and the times: hook path under 2 s, re-check within its interval, backstop within 5 s.

**Constraints:** The server is authoritative; the UI follows.

**Expected outcome:** Permission and status changes apply to live sockets.

**Acceptance checks:** All timings meet the stated limits.

**Logging:** Append the CP-45 entry.

**Next prompt will:** build the scale harness.

**Closing line:** Print the timings, then print exactly: **STOP AND TEST: checkpoint CP-45: run the access and lock tests; record and commit.** Suggested commit message: `feat(sow_module): socket routing, access re-checks, finalize lock`.

**END PROMPT**

---

## CP-46: Scale harness and scenario scripts

**BEGIN PROMPT**

**From previous prompt:** access enforcement works; read `phaseB_report.md`, `spike_scale.md` and `prompt_log.md` first.

**Goal:** Build a repeatable harness. Do not run the full measurements yet.

**Tasks:**
1. A harness with headless Yjs clients (typing model, optional paste, round-trip timing) and scripts for these scenarios: steady typing, burst join, reconnect storm, mixed roles, 5 MB document, lease failover, paste storm, awareness load, revoke under load, many rooms, 2-hour soak at 50 editors, "Snowflake slow" (inject latency), "Snowflake down 60 s".
2. Metrics output as JSON: p50, p95, p99 for broadcast, durable acknowledgment, session open and revoke-to-close; memory and CPU; Snowflake statements per minute.
3. A smoke run of each scenario at 10 editors to prove the scripts work.
4. Document how to run each scenario in `sow_docs/tracking/scale_howto.md`.

**Constraints:** Local only; test data only; no customer content.

**Expected outcome:** Every scenario runs at small scale.

**Acceptance checks:** All scenarios complete at 10 editors and write JSON.

**Logging:** Append the CP-46 entry.

**Next prompt will:** run the scenarios at 10, 25, 50 and 100 editors and record the gates.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-47: Scale measurements and approved gates

**BEGIN PROMPT**

**From previous prompt:** the harness works; read `scale_howto.md` and `prompt_log.md` first.

**Goal:** Measure at 10, 25, 50 and 100 editors and record the results against the gates.

**Tasks:**
1. Run every scenario at each level (the soak at 50 only).
2. Write `sow_docs/tracking/scale_results.md`: measured values for each gate (delivered broadcast, durable acknowledgment, session open, revoke-to-close, error rate), a table by level, and Snowflake statement counts and an estimated warehouse cost.
3. For any gate not met, record the measured value and ask the user to approve it. **Never relax a gate silently.**

**Constraints:** Label every figure as a local, not production-shaped, measurement.

**Expected outcome:** A scale report against every gate.

**Acceptance checks:** All scenarios ran at 100 editors, or the failure is explained.

**Logging:** Append the CP-47 entry.

**Next prompt will:** begin Phase F with the production intake pipeline.

**Closing line:** Print the headline results, then print exactly: **STOP AND TEST: checkpoint CP-47: review `scale_results.md` with the user and approve any measured gates; record and commit. This closes Phase E.** Suggested commit message: `test(sow_module): scale harness and measured gates (phase E complete)`.

**END PROMPT**

---

# PHASE F: Content Intake: LLM, Application Data, Legacy Documents (CP-48 to CP-53)

---

## CP-48: Production intake pipeline

**BEGIN PROMPT**

**From previous prompt:** Phase E is complete; read `spike_intake.md`, `shared/intake.ts` and `prompt_log.md` first.

**Goal:** Promote the intake spike to the one pipeline every content source uses.

**Tasks:**
1. Finalize `shared/intake.ts`: formats `html` (primary), `markdown`, `text`; allowlist; removal report; stable block IDs.
2. The Owner applies intake output as **one Yjs transaction** attributed to the acting user with `meta.source` of `author`, `code` or `llm`.
3. fast-check fuzzing over all formats; hostile corpus from CP-12 and CP-14.
4. A pytest in `blend_ode_be/tests/` that feeds the vendored fixture corpus through `sanitize_draft_html` and checks its output equals the TypeScript allowlist's expected output. Also confirm `sanitize_rich_text` and the exporters are unchanged.

**Constraints:** Exactly one allowlist and one intake module.

**Expected outcome:** A single safe pipeline with parity to the Python `sanitize_draft_html`.

**Acceptance checks:** All formats convert; hostile corpus neutralized; parity pytest passes.

**Logging:** Append the CP-48 entry.

**Next prompt will:** insert LLM and application-generated content into the canvas.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-49: LLM and generated content: client and server paths

**BEGIN PROMPT**

**From previous prompt:** intake is final; read `audit_backend.md` (LLM paths), `audit_frontend.md` and `prompt_log.md` first.

**Goal:** Let generated text (narrative drafts, continued drafts, suggested fixes, add-ins, pasted LLM answers) enter the live document safely.

**Tasks:**
1. Client API `insertContent({format, content, target})` with targets cursor, end, after block, replace block. Editors and SOW admins only.
2. **(Deferred 2026-11-12: needs a durable pending-insert table; see pending.txt.)** Server path: a FastAPI endpoint `POST /sows/{sow_id}/collab/insert` (same auth as the session endpoint) that forwards `{format, content, target, idempotencyKey, meta: {model, requestId}}` to the Owner over the internal channel, so backend-generated content can be placed even when called by a service. Idempotent on the key.
3. Record each insertion in the audit store (who, model, request id, content hash). Do not store prompts or raw responses. **Decided 2026-11-12 (Option A):** store only the final generated HTML hash, model and actor; no prompts or raw LLM responses.
4. Find in `audit_frontend.md` how suggested fixes and generated drafts currently reach CKEditor, and route those same calls to `insertContent` when the Lexical editor is active, without editing the CKEditor path.
5. Tests: each target; role denial; idempotent replay inserts once; the CKEditor path still works with the flag off.

**Constraints:** LLM text is untrusted and goes through intake like any paste.

**Expected outcome:** Generated content can be placed safely, once, with an audit record.

**Acceptance checks:** All listed tests pass.

**Logging:** Append the CP-49 entry.

**Next prompt will:** add the streaming preview and smart paste.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-50: Streaming preview and smart paste

**BEGIN PROMPT**

**From previous prompt:** insertion paths work; read `prompt_log.md` first.

**Goal:** Make LLM insertion pleasant and safe.

**Tasks:**
1. **Streaming preview:** while tokens arrive, show them in a local ghost block that is not in the shared document; commit once at the end through intake (one undo step; collaborators never see partial text). Handle cancel and error.
2. **Smart paste:** plain text that looks like markdown (headings, lists, tables) converts to formatted content with a notice and one undo to return to plain.
3. Tests: streaming commits exactly once; collaborators never receive partial text; cancel leaves the document unchanged; smart paste converts and reverts cleanly.

**Constraints:** Only the final committed result is shared and persisted.

**Expected outcome:** Smooth LLM preview-and-commit and helpful markdown paste.

**Acceptance checks:** All tests pass.

**Logging:** Append the CP-50 entry.

**Next prompt will:** make existing documents open in Lexical.

**Closing line:** Print the results, then print exactly: **STOP AND TEST: checkpoint CP-50: run the insert tests and try pasting a real LLM answer; record and commit.** Suggested commit message: `feat(sow_module): intake pipeline, LLM insert, streaming preview, smart paste`.

**END PROMPT**

---

## CP-51: Existing drafts open in Lexical (legacy documents)

**BEGIN PROMPT**

**From previous prompt:** LLM insert works; read `spike_html_fidelity.md` and `prompt_log.md` first.

**Goal:** Documents written in CKEditor open in Lexical with no data loss and with a report of anything changed. The stored HTML is never altered by opening.

**Tasks:**
1. Opening a SOW with no live state seeds from the latest draft HTML (CP-23). Add a **conversion report** (what was removed or normalized) kept with the live state, shown to SOW admins.
2. The original HTML stays untouched in `SOW_DRAFT_VERSIONS`. A checkpoint with unchanged content must not create a spurious change: compare normalized HTML and skip the save when equal.
3. A batch fidelity script `blend_ode_be/scripts/` or `blend_ode_be/sow_module/collab/scripts/` that reads a user-supplied set of draft HTML files, runs them through import and export, and writes a combined report. It never writes to Snowflake.
4. Tests: seeding from fixtures; a no-op open and checkpoint creates no new revision; report lists approved losses.
5. **Open Items scroll (CKEditor parity gap):** `DraftEditorScreen.tsx` currently finds placeholders via `.ck-editor__editable` and `sow-studio-manual-placeholder` DOM selectors. Those selectors return nothing in the Lexical editor. Wire the open-items click handler to call `lexicalApiRef.current.scrollToBlock(blockId)` using the block ID injected by `ensureSectionAnchors`. The right-rail open-items list display is unaffected; only the click-to-scroll focus breaks. Test: clicking an open item in Lexical mode scrolls the editor to the correct block.

**Constraints:** Opening a document must never modify existing data.

**Expected outcome:** Safe, reportable legacy opening. Open-items scroll works in Lexical mode.

**Acceptance checks:** No defects on the fixture set. A no-op session leaves revisions unchanged. Clicking an open item scrolls to the correct placeholder in the Lexical editor.

**Logging:** Append the CP-51 entry.

**Next prompt will:** handle regeneration and replace-all safely.

**Closing line:** Print the fidelity summary, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-52: Regeneration, intake sync and reset to latest

**BEGIN PROMPT**

**From previous prompt:** legacy opening works; read `audit_frontend.md` (intake sync) and `prompt_log.md` first.

**Goal:** Handle the actions that replace the whole draft while a live room exists.

**Tasks:**
1. Identify every action that replaces draft content (regenerate, intake synchronization, continue a historical version, restore). Describe each in the log.
2. For each, define the live-room behavior: show a confirmation; take a checkpoint first so the current content is a restorable version; replace through intake as one transaction; tell connected clients with a visible notice.
3. A "reload latest" action (like `reset_native_editor_to_latest`) that discards the live state and reseeds from the latest draft, with an expected-revision check and an audit entry.
4. Tests: replacing content with two connected clients; the checkpoint-before-replace guarantee; stale revision refused.

**Constraints:** Never discard live content without a checkpoint.

**Expected outcome:** Whole-draft replacements are safe in live rooms.

**Acceptance checks:** The restorable-version guarantee holds in tests.

**Logging:** Append the CP-52 entry.

**Next prompt will:** polish intake messages and close Phase F.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-53: Intake messages, fuzzing and Phase F close

**BEGIN PROMPT**

**From previous prompt:** replacement flows work; read `prompt_log.md` first.

**Goal:** Finish the intake experience and harden it.

**Tasks:**
1. User-facing "Some formatting was removed" messages for paste, LLM insert, add-in insert and legacy open, with details for SOW admins.
2. Large-scale fast-check fuzzing on every format and fix anything that slips through.
3. Write `phaseF_report.md`: sources wired, fidelity summary, fuzz coverage, open items.
4. Full regression of frontend module, Node service and the backend SOW tests.

**Constraints:** No content source may bypass the allowlist.

**Expected outcome:** A hardened intake path and a phase report.

**Acceptance checks:** Fuzzing finds no bypass; regression green.

**Logging:** Append the CP-53 entry.

**Next prompt will:** begin Phase G with the decision on the existing native editor.

**Closing line:** Print the summary, then print exactly: **STOP AND TEST: checkpoint CP-53: run the full suites; record and commit. This closes Phase F.** Suggested commit message: `feat(sow_module): legacy opening, regeneration safety, intake hardening (phase F complete)`.

**END PROMPT**

---

# PHASE G: Mentions, Comments, Suggestions, Guests (CP-54 to CP-61)

---

## CP-54: Decision gate: relationship to the native editor, and the participants list

**BEGIN PROMPT**

**From previous prompt:** Phase F is complete; read `decisions.md` (D-B1), `audit_backend.md` and `prompt_log.md` first.

**Goal:** Settle how comments, suggestions, guests and mentions are stored and reused before building them.

**Tasks:**
1. Re-read `native_collaboration.py`, the `SOW_NATIVE_*` tables, `NativeDraftEditorScreen.tsx`, `SowNativeGuestReview.tsx` and `nativeEditorModel.ts`. Summarize in `sow_docs/tracking/design_collab_features.md`: what the native editor stores (threads, suggestions, guests, mention deliveries, audit), who can do what, and what it lacks (real-time, block-stable anchors).
2. Propose and record the plan, then **ask the user to confirm D-B1**. Default proposal: comment threads and suggestions live in the Yjs document so they sync live, and are persisted into the existing `COMMENT_THREADS` and `SUGGESTIONS` columns (same JSON shape) at checkpoint through a FastAPI internal endpoint that reuses the existing audit and mention-notification code; guest invitations, mention deliveries and audit events reuse the existing tables and endpoints unchanged.
3. Define the participants list: reuse `_participants` logic (deal owner, leads, collaborators, pricing contributors, active guests). Implement a FastAPI endpoint `GET /sows/{sow_id}/collab/participants` that calls it, returning name, email and role.
4. Pytest for the participants endpoint (access rules, guests included, expired guests excluded).

**Constraints:** Do not change existing native-editor behavior. Do not migrate its data.

**Expected outcome:** A confirmed design and a participants endpoint.

**Acceptance checks:** The design file exists; the user answered D-B1; tests pass.

**Logging:** Append the CP-54 entry with the decision.

**Next prompt will:** build mentions.

**Closing line:** Print the question for the user, then print exactly: **READY FOR NEXT PROMPT.** (Do not continue until D-B1 is answered.)

**END PROMPT**

---

## CP-55: Mentions

**BEGIN PROMPT**

**From previous prompt:** D-B1 is decided and participants exist; read `design_collab_features.md` and `prompt_log.md` first.

**Goal:** `@` mentions in comments and the document, stored as people references.

**Tasks:**
1. `MentionNode` storing the person's email (as the existing comment model does) and typeahead on `@` using the participants list; viewers cannot mention.
2. Mentions inside comments keep the existing `mentions` array shape.
3. New mentions trigger the **existing** delivery path (`SOW_NATIVE_MENTION_DELIVERIES`, idempotency key, email only when `ENABLE_SOW_EMAIL_SENDING`, guest tokens for guests) through a FastAPI internal function that reuses `_send_new_mention_notifications` logic by import, not copy.
4. Tests: viewer blocked; mention stored; delivery is idempotent; suppressed when email sending is off.

**Constraints:** Reuse existing delivery code; never email from the Node service.

**Expected outcome:** Mentions that notify through the existing system.

**Acceptance checks:** Tests pass; existing `test_native_collaboration.py` and `test_send_mail.py` still pass.

**Logging:** Append the CP-55 entry.

**Next prompt will:** build comment threads anchored in the document.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-56: Comment threads and anchors

**BEGIN PROMPT**

**From previous prompt:** mentions work; read `design_collab_features.md` and `prompt_log.md` first.

**Goal:** Inline comment threads that follow the text through edits and sync live.

**Tasks:**
1. Threads stored in a shared Yjs map using the existing thread and comment JSON shape; the document carries a mark (`@lexical/mark`) holding only the thread id.
2. Anchors use the block id plus Yjs relative positions so they survive edits; keep the old `nodeId`, `start`, `end`, `contextBefore`, `contextAfter` fields filled for compatibility with the existing review screens.
3. Create thread, reply, resolve and reopen. Commenters, editors and SOW admins may comment; viewers may not.
4. Orphan handling: if the anchored text is deleted the thread stays visible and is marked orphaned; test full and partial deletes.
5. Resolve restyles the mark rather than removing it.
6. Tests for each rule.

**Constraints:** Create the thread data before applying the mark; if the mark fails, remove the thread.

**Expected outcome:** Anchored, live-synced comment threads.

**Acceptance checks:** Anchors follow edits; orphans are shown correctly.

**Logging:** Append the CP-56 entry.

**Next prompt will:** build the comments panel and persist threads.

**Closing line:** Print the results, then print exactly: **STOP AND TEST: checkpoint CP-56: run frontend and Node tests; try two users commenting; record and commit.** Suggested commit message: `feat(sow_module): mentions and anchored comment threads`.

**END PROMPT**

---

## CP-57: Comments panel

**BEGIN PROMPT**

**From previous prompt:** threads work; read `design_collab_features.md` and `prompt_log.md` first.

**Goal:** A usable comments panel in the app's style.

**Tasks:**
1. Panel using antd: list threads, jump to the anchor, reply, resolve and reopen, filter open or resolved, orphan badge.
2. Authors can edit and delete their own comments; SOW admins can redact one (the data work is in CP-58; here build the controls and states).
3. Viewers see the panel read-only; commenters and above can reply and resolve.
4. Keyboard operation and accessible names; empty, loading and error states.
5. Component tests for each state and role.

**Constraints:** Follow the look of the existing review screens. Styles under `sowx-`.

**Expected outcome:** A comments panel that works for every role.

**Acceptance checks:** Role states are correct; axe shows no serious issues.

**Logging:** Append the CP-57 entry.

**Next prompt will:** persist threads, audit them and handle redaction.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-58: Comment persistence, audit and redaction

**BEGIN PROMPT**

**From previous prompt:** the panel works; read `design_collab_features.md` and `prompt_log.md` first.

**Goal:** Persist comments through the existing backend, with audit and complete redaction.

**Tasks:**
1. Persist threads into `COMMENT_THREADS` and the `comments` payload of the draft checkpoint through a FastAPI internal endpoint that reuses the audit helpers (`_audit_review_changes`), so audit events (`comment_created`, `comment_replied`, `comment_resolved` and the rest) remain identical to the native editor's.
2. Edit and delete by the author; redaction by a SOW admin removes the text everywhere it is stored, including later checkpoints.
3. Comments remain available after finalize for roles that can comment (content stays locked).
4. Tests, including a redaction test that searches stored data for the removed text.

**Constraints:** Keep the stored JSON shape compatible with `SerializedDraftComments` and the native threads.

**Expected outcome:** Comments persist, audit and redact correctly.

**Acceptance checks:** Redaction leaves no text; audit event names match the native ones; existing native tests still pass.

**Logging:** Append the CP-58 entry.

**Next prompt will:** build suggestions.

**Closing line:** Print the results, then print exactly: **STOP AND TEST: checkpoint CP-58: run comment tests and the existing native-editor tests; record and commit.** Suggested commit message: `feat(sow_module): comments panel, persistence, audit, redaction`.

**END PROMPT**

---

## CP-59: Suggestions

**BEGIN PROMPT**

**From previous prompt:** comments persist; read `design_collab_features.md`, `nativeEditorModel.ts` and `prompt_log.md` first.

**Goal:** Suggested edits using the existing shape, live in the editor.

**Tasks:**
1. Suggestions in the shared document using the existing shape (`originalText`, `replacementText`, `status` of pending, accepted or rejected, `anchor`).
2. Create (commenters and above), accept and reject (editors and SOW admins only). Accepting applies a normal undoable edit.
3. Persist like comments, with `suggestion_created`, `suggestion_accepted`, `suggestion_rejected` audit events as the native editor writes them.
4. Pending suggestions must be decided before a checkpoint is used for sign-off, matching the existing rule in `checkpoint_native_editor`.
5. Tests for create, accept, reject, role limits and the sign-off rule.

**Constraints:** Keep the stored shape compatible with the native editor.

**Expected outcome:** Working suggestions.

**Acceptance checks:** Tests pass; `test_native_collaboration.py` still passes.

**Logging:** Append the CP-59 entry.

**Next prompt will:** add guest reviewers.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-60: Guest reviewers

**BEGIN PROMPT**

**From previous prompt:** suggestions work; read `design_collab_features.md`, the guest code in `native_collaboration.py` and `prompt_log.md` first.

**Goal:** Let external guests review in the live editor with the same limits they have today (comment and suggest, no direct edits).

**Tasks:**
1. Reuse the existing guest invitation and verification flow (`verify_guest_invitation`, the bearer token the guest page uses). Add a guest variant of the session endpoint that issues a ticket with role `commenter` plus the suggest capability, only for an active, unexpired, unrevoked invitation.
2. A guest sees only what the guest page shows today; test every capability, on the component, REST and socket layers.
3. A guest page variant (or flag) that mounts the Lexical editor in comment-and-suggest mode, following `SowNativeGuestReview.tsx`.
4. Tests: valid, expired and revoked invitations; a guest cannot edit content on the client or the server; a guest cannot call any editor-only endpoint.

**Constraints:** Do not change how invitations are created, verified or emailed.

**Expected outcome:** Guests review live within existing limits.

**Acceptance checks:** Tests pass; existing native guest tests still pass.

**Logging:** Append the CP-60 entry.

**Next prompt will:** wire guest revocation and run the full role matrix.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-61: Guest revocation, role regression and Phase G close

**BEGIN PROMPT**

**From previous prompt:** suggestions and guests work; read `prompt_log.md` first.

**Goal:** Close the loop on access changes for guests and verify every role.

**Tasks:**
1. Call `notify_access_changed` from the existing revoke-invitation code path (without altering its response) and prove a revoked guest is disconnected within the stated time.
2. A role regression: every role (viewer, commenter, editor, SOW admin, guest) against every action (view, comment, reply, mention, suggest, edit, insert content, checkpoint, rename, finalize lock) at three layers: component, REST and socket.
3. Write `phaseG_report.md`.

**Constraints:** The server is authoritative.

**Expected outcome:** Verified roles and revocation.

**Acceptance checks:** Full matrix passes.

**Logging:** Append the CP-61 entry.

**Next prompt will:** begin Phase H with version history and the finalize lock.

**Closing line:** Print the matrix summary, then print exactly: **STOP AND TEST: checkpoint CP-61: run the role matrix; record and commit in every changed repo. This closes Phase G.** Suggested commit message: `feat(sow_module): suggestions, guest reviewers, revocation (phase G complete)`.

**END PROMPT**

---

# PHASE H: Versions, Finalize Lock, Export Parity, Audit (CP-62 to CP-66)

> Versions, finalize, sign-off, publication and DOCX/PDF export **already exist** in your backend. This phase proves the Lexical editor works with them. It does not rebuild them.

---

## CP-62: Version history and historical view

**BEGIN PROMPT**

**From previous prompt:** Phase G is complete; read `audit_frontend.md`, `audit_backend.md` and `prompt_log.md` first.

**Goal:** Make version history, historical viewing and "continue from this version" work with the Lexical editor.

**Tasks:**
1. Historical versions (`get_sow_draft_version`) render read-only from the version's HTML without opening a live room, using the same `readOnly` and `readOnlyLabel` props as today.
2. "Continue from a version" (`continue_draft_version`) creates a new draft; the live room for that SOW must reseed from the new current draft (reuse the CP-52 flow) and tell connected users.
3. A checkpoint updates the current version **in place** as `save_sow_draft` does; confirm the version list in the existing version management screen (`features/versionManagement.tsx`) shows the right entries and that no extra versions appear from live editing.
4. Tests (Jest and pytest) for each flow.

**Constraints:** Do not change the version-management code. Add adapters only.

**Expected outcome:** Version flows work for flagged users.

**Acceptance checks:** Existing version tests pass. New tests pass.

**Logging:** Append the CP-62 entry.

**Next prompt will:** prove finalize and sign-off work with Lexical content.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-63: Flush before finalize

**BEGIN PROMPT**

**From previous prompt:** version flows work; read `audit_backend.md` (finalize findings) and `prompt_log.md` first.

**Goal:** Make sure finalize captures the latest live content.

**Tasks:**
1. The existing finalize action must first request a checkpoint from the Owner (a FastAPI function that calls the Node service, with a timeout), so the finalized version contains the latest live content. If the Owner is unreachable, finalize must fail with a clear message rather than finalize stale content.
2. Add this as a small wrapper at the router layer without changing the signatures of existing finalize code. Describe the exact lines in the log. It does nothing when no live room exists or the flag is off.
3. After finalize, the room locks (CP-45).
4. Tests with a fake Owner: flush succeeds then finalize; Owner down then finalize refused; no live room then finalize unchanged.

**Constraints:** Existing finalize behavior must be unchanged when the flag is off or no live room exists.

**Expected outcome:** Finalize never loses live edits.

**Acceptance checks:** `test_sow_http_e2e.py` and finalize tests still pass; new tests pass.

**Logging:** Append the CP-63 entry.

**Next prompt will:** prove sign-off and publication work with Lexical content.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-64: Sign-off and publication integration

**BEGIN PROMPT**

**From previous prompt:** flush before finalize works; read `audit_backend.md` and `prompt_log.md` first.

**Goal:** A SOW edited in Lexical can be signed off and published exactly like a CKEditor SOW.

**Tasks:**
1. Run the existing sign-off, publication and finalize tests; list any failures that existed before.
2. Integration test: edit live in two clients, checkpoint, finalize, confirm the final HTML equals the last live content, and that further edits are refused.
3. Confirm `sow_publication.py` and the public approval pages render Lexical-produced HTML (use fixtures with tables, highlights and merged cells).
4. Confirm the `hasPendingSignOffs` watermark follows live status.

**Constraints:** Do not change sign-off or publication code.

**Expected outcome:** The full SOW lifecycle works with Lexical content.

**Acceptance checks:** `test_sow_signoffs.py`, `test_sow_publication.py`, `test_sow_http_e2e.py` pass; the new integration test passes.

**Logging:** Append the CP-64 entry.

**Next prompt will:** prove DOCX and PDF export and quality checks accept Lexical output.

**Closing line:** Print the results, then print exactly: **STOP AND TEST: checkpoint CP-64: run the finalize, sign-off, publication and e2e tests; record and commit in every changed repo.** Suggested commit message: `feat(sow_module): version history, finalize flush, sign-off and publication integration`.

**END PROMPT**

---

## CP-65: Export, quality and open-items parity

**BEGIN PROMPT**

**From previous prompt:** finalize integration works; read `audit_backend.md` (HTML dependency list) and `prompt_log.md` first.

**Goal:** Prove every downstream consumer of draft HTML accepts Lexical output.

**Tasks:**
1. Golden-file pytest: a fixture set of Lexical-produced HTML (tables with merges, highlights, placeholders, nested lists, alignment, fonts) through `sow_docx_export`, `sow_pdf_export`, `draft_quality`, `quality_guard`, open-items reconciliation, `source_coverage` and `quote_grounding`. Compare against the same content produced by CKEditor fixtures; differences must be listed and either approved or fixed **in the Lexical serializer**.
2. Download buttons: `onDownload('doc' | 'pdf')` in the Lexical editor must call the same existing export path as the CKEditor one.
3. Check that placeholder markers and highlight classes render the same in exports as before.
4. Write the differences list in `sow_docs/tracking/export_parity.md`.

**Constraints:** Do not modify exporters or quality code. Fix differences in our serializer.

**Expected outcome:** Exports and checks give equivalent results.

**Acceptance checks:** Golden tests pass; the differences list has no unapproved items.

**Logging:** Append the CP-65 entry.

**Next prompt will:** add audit, metrics and the local runbook.

**Closing line:** Print the results, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-66: Audit, metrics and local runbook

**BEGIN PROMPT**

**From previous prompt:** export parity holds; read `prompt_log.md` first.

**Goal:** Make the module observable and its operations written down.

**Tasks:**
1. Audit events for live sessions: joined, left, checkpoint created, content inserted (LLM or add-in), access changed, room locked. Use the audit store decided in D-B1, with the same hash-chain or event shape as existing audit events.
2. Metrics logs (structured, no content): durable-ack latency, queue depth, flush time, Snowflake statements, lease renewals and losses, rooms and sockets, intake rejections, checkpoint time.
3. Runbook `sow_docs/tracking/runbook_local.md`: start and stop each service, environment variables, Snowflake outage, stuck lease, stuck flush, rollback to CKEditor, key rotation.
4. Retention note for the new tables (live updates are pruned by compaction; audit follows existing policy); ask the user for the retention decision.

**Constraints:** No document text in logs or metrics.

**Expected outcome:** Audit trail, metrics and a runbook.

**Acceptance checks:** A sentinel test shows no document text reaches logs.

**Logging:** Append the CP-66 entry.

**Next prompt will:** begin Phase I with full test matrices.

**Closing line:** Print the results, then print exactly: **STOP AND TEST: checkpoint CP-66: run the checks; record and commit. This closes Phase H.** Suggested commit message: `feat(sow_module): version, finalize, export integration and audit (phase H complete)`.

**END PROMPT**

---

# PHASE I: Hardening (CP-67 to CP-71)

> Local hardening only. The independent penetration test and the cloud recovery drill belong to the Deployment block.

---

## CP-67: Full test matrices

**BEGIN PROMPT**

**From previous prompt:** Phase H is complete; read `prompt_log.md` first.

**Goal:** Run complete matrices and close gaps.

**Tasks:**
1. Matrices: every role by every action; the document lifecycle (draft, live edit, checkpoint, finalize, sign-off, new draft); intake sources (paste, LLM, add-in, legacy open); failure cases (Snowflake slow and down, lease loss, FastAPI down, drain).
2. Record coverage in `sow_docs/tracking/test_matrix.md`. Fix failures.
3. Run the **entire** existing frontend and backend test suites and confirm nothing outside `sow_module` regressed; list any pre-existing failures separately.

**Constraints:** Use production code paths, not spikes.

**Expected outcome:** A passing, documented matrix.

**Acceptance checks:** All cells pass or have an accepted exception.

**Logging:** Append the CP-67 entry.

**Next prompt will:** re-run load tests against the approved gates.

**Closing line:** Print the summary, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-68: Load tests against the gates

**BEGIN PROMPT**

**From previous prompt:** matrices pass; read `scale_results.md`, `phaseB_report.md` and `prompt_log.md` first.

**Goal:** Re-run load at 10, 25, 50 and 100 editors and a 5 MB document against the approved gates.

**Tasks:**
1. Re-run the harness (k6 or a script for HTTP session, insert and participants endpoints; the Node client harness for sockets).
2. Record broadcast, durable acknowledgment, session open, mention search, checkpoint time, revoke-to-close and error rate.
3. Update `scale_results.md`. For any gate not met, ask the user to approve the measured value.

**Constraints:** Local and labelled as such.

**Expected outcome:** Measured results against every gate.

**Acceptance checks:** Every gate is measured; failures are flagged.

**Logging:** Append the CP-68 entry.

**Next prompt will:** run the security hardening pass.

**Closing line:** Print the headline results, then print exactly: **STOP AND TEST: checkpoint CP-68: review the gate results with the user; record and commit.** Suggested commit message: `test(sow_module): full matrices and load results`.

**END PROMPT**

---

## CP-69: Security: tickets, internal endpoints, SQL and secrets

**BEGIN PROMPT**

**From previous prompt:** load results are recorded; read `contracts.md`, `decisions.md` and `prompt_log.md` first.

**Goal:** Harden the authentication and data-access surface.

**Tasks:**
1. Ticket checks: expiry, replay, tampering, wrong SOW, wrong user, wrong origin. Internal endpoints: missing or wrong service key.
2. Confirm every SQL statement in the Node service and the new Python code uses bind variables.
3. Run the sentinel-secret test over logs, responses and the built frontend bundle.
4. Confirm the Node service's Snowflake role cannot read any application table.
5. Report (do not change) that `blend_ode_be/src/common/database.py` connects with `insecure_mode: True`, and make sure the Node service does not copy that setting. Record it for the user as a finding.
6. Start `sow_docs/tracking/security_pass.md` with findings and fixes.

**Constraints:** Any high finding blocks progress until fixed.

**Expected outcome:** A clean first half of the security pass.

**Acceptance checks:** No leaks; the Node role is denied application tables.

**Logging:** Append the CP-69 entry.

**Next prompt will:** fuzz content paths, abuse the WebSocket and run the role regression.

**Closing line:** Print the summary, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-70: Security: content fuzzing, WebSocket abuse and role regression

**BEGIN PROMPT**

**From previous prompt:** the first half of the security pass is clean; read `security_pass.md` and `prompt_log.md` first.

**Goal:** Finish the security pass.

**Tasks:**
1. Fuzz every content entry (paste, LLM, add-in, legacy open, checkpoint HTML) and confirm `sanitize_draft_html` and the TypeScript allowlist agree.
2. WebSocket abuse: oversized payload, malformed frames, message flood, cross-site origin, a viewer sending updates.
3. A role regression on every endpoint and socket message.
4. Complete `security_pass.md` and record any accepted risks for the user.

**Constraints:** Any high finding blocks progress until fixed.

**Expected outcome:** A recorded, clean security pass.

**Acceptance checks:** No bypass found; parity holds; every abuse case is rejected.

**Logging:** Append the CP-70 entry.

**Next prompt will:** accessibility and cross-browser checks.

**Closing line:** Print the summary, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-71: Accessibility and cross-browser

**BEGIN PROMPT**

**From previous prompt:** the security pass is clean; read `prompt_log.md` first.

**Goal:** Confirm the editor is accessible and works in the required browsers.

**Tasks:**
1. Axe plus manual keyboard-only checks of the toolbar, tables, find and replace, comments panel and mention list; accessible names; focus order; screen-reader notes for table headers.
2. Browser matrix: Chrome and Edge required; Firefox and Safari non-blocking. Check the antd look and the SCSS do not conflict (the editor styles stay under `sowx-`).
3. Fix serious issues; record the rest in `sow_docs/tracking/a11y_report.md`. Write `phaseI_report.md`.

**Constraints:** Serious accessibility issues block.

**Expected outcome:** An accessible editor and a browser matrix.

**Acceptance checks:** No serious axe findings; Chrome and Edge pass.

**Logging:** Append the CP-71 entry.

**Next prompt will:** begin Phase J with rollout controls.

**Closing line:** Print the summary, then print exactly: **STOP AND TEST: checkpoint CP-71: run the accessibility and browser checks; record and commit. This closes Phase I.** Suggested commit message: `test(sow_module): security, accessibility, cross-browser (phase I complete)`.

**END PROMPT**

---

# PHASE J: Cutover Tooling (CP-72 to CP-74)

---

## CP-72: Rollout controls and migration dry run

**BEGIN PROMPT**

**From previous prompt:** Phase I is complete; read `prompt_log.md` and `core_api_freeze.md` first.

**Goal:** Control who uses Lexical and prove every existing draft converts safely.

**Tasks:**
1. Rollout: `FE_SOW_LEXICAL_EDITOR` grants Lexical per user, as other flags do. Add an admin-visible view (follow the pattern in `components/admin/UsersAdminTab.tsx`) showing who has the flag, without changing other admin behavior.
2. A **read-only** dry-run script that takes exported draft HTML (from a file the user provides, never reading Snowflake directly) and writes a combined fidelity report: converted cleanly, approved loss, defect.
3. List documents that need manual attention.

**Constraints:** The dry run never writes to any database.

**Expected outcome:** Per-user rollout and a reviewed migration report.

**Acceptance checks:** Report covers every supplied document; flag toggles per user.

**Logging:** Append the CP-72 entry.

**Next prompt will:** prove rollback to CKEditor.

**Closing line:** Print the dry-run summary, then print exactly: **READY FOR NEXT PROMPT.**

**END PROMPT**

---

## CP-73: Rollback to CKEditor

**BEGIN PROMPT**

**From previous prompt:** rollout controls exist; read `prompt_log.md` first.

**Goal:** Prove a user can be returned to CKEditor with nothing lost.

**Tasks:**
1. Because HTML is the system of record, rollback means: request a final checkpoint from the Owner, then turn off the flag. Implement a safe "switch off for this SOW" step that checkpoints first and closes the room.
2. Test: edit in Lexical, checkpoint, flag off, open in CKEditor, and confirm the content matches (normalized comparison); then repeat with flag on again.
3. Confirm comment and suggestion data are visible in the native review screens as before (D-B1 shape compatibility).

**Constraints:** No data conversion on rollback beyond the checkpoint.

**Expected outcome:** Reversible rollout.

**Acceptance checks:** Round trip between editors loses no content.

**Logging:** Append the CP-73 entry.

**Next prompt will:** write the cutover runbook and the CKEditor removal plan.

**Closing line:** Print the results, then print exactly: **STOP AND TEST: checkpoint CP-73: run the rollback tests; record and commit.** Suggested commit message: `feat(sow_module): rollout controls, dry run and rollback`.

**END PROMPT**

---

## CP-74: Cutover runbook, CKEditor removal plan, build-complete report

**BEGIN PROMPT**

**From previous prompt:** rollback is proven; read the whole `sow_docs/tracking/` folder first.

**Goal:** Finish local development with the documents needed to deploy, roll out and remove CKEditor.

**Tasks:**
1. `cutover_runbook.md`: the staged rollout (named pilot users first, then a team, then percentages), what to watch (durable-ack latency, error rate, Snowflake cost and warehouse hours, checkpoint failures), the rollback trigger and steps, and a pilot checklist with blanks for the user (pilot users, support contact, on-call).
2. `ckeditor_removal_plan.md`: after a stable period, remove `DraftEditor.tsx`'s CKEditor use, the `ckeditor5` and `@ckeditor/ckeditor5-react` dependencies, `scripts/cleanup-stale-ckeditor-builds.js` and its test, and the CKEditor mocks and CSS import; decide what happens to `NativeDraftEditorScreen`, `nativeEditorModel.ts` and `FE_SOW_NATIVE_EDITOR` (D-B1). **Do not remove anything yet.**
3. `build_complete_report.md`: every CKEditor parity row from `audit_frontend.md` marked built, approved loss or open; outstanding decisions; and the handoff list for the Deployment block.
4. Full regression across the frontend, the Node service and the backend.

**Constraints:** Do not delete CKEditor or native-editor code in this prompt.

**Expected outcome:** Local development is complete and documented.

**Acceptance checks:** All three documents exist; the parity table has no unexplained rows; full regression is green.

**Logging:** Append the CP-74 entry with the final feature state.

**Next prompt will:** be in the separate Deployment block (to be written).

**Closing line:** Print the parity summary and gaps, then print exactly: **STOP AND TEST: checkpoint CP-74: run the full regression; record and commit. Local development is complete. Ask whether to write the Deployment block next.** Suggested commit message: `docs(sow_module): cutover runbook, removal plan, build complete (local development complete)`.

**END PROMPT**

---

---

# Appendix A: Deployment block (to be written after local development)

About five prompts, not written yet:

1. **CP-D1** Dockerfiles for the Node collab service and local container run.
2. **CP-D2** ECS service, load balancer with WebSocket support, private service discovery, secrets, Snowflake network policy. **First verify whether your current backend host (the config comments mention App Runner) supports WebSockets; the collaboration service needs ECS or another WebSocket-capable host.**
3. **CP-D3** Staging environment tests, cloud checkpoints, warehouse cost measurement.
4. **CP-D4** Independent penetration test coordination, recovery drill from Snowflake Time Travel, key rotation drill.
5. **CP-D5** Staged production rollout, named pilot, rollback rehearsal, go or no-go.

# Appendix B: The test and commit points (28)

CP-00, CP-04, CP-07, CP-09, CP-11, CP-15, CP-18, CP-21, CP-25, CP-28, CP-30, CP-33, CP-35, CP-39, CP-42, CP-45, CP-47, CP-50, CP-53, CP-56, CP-58, CP-61, CP-64, CP-66, CP-68, CP-71, CP-73, CP-74.

Decision checkpoints (review with the user, not only run tests): **CP-09, CP-11, CP-15** (architecture), **CP-17** (approve the SQL migration), **CP-47 and CP-68** (scale gates), **CP-54** (native editor decision, D-B1).

# Appendix C: Questions Claude Code will ask you along the way

| When | Question |
|---|---|
| CP-00 | Which Node version do you run? Confirm tracking and the Node service go in the backend repo (`blend_ode_be/sow_module/`) |
| CP-03 | Where are migrations 011 to 013? Please run `GET_DDL` on the `SOW_NATIVE_*` tables and paste the result |
| CP-05 | Role mapping, D-B1, D-B3 (module-owned HTML allowlist), D-B4 (vendored shared code), who may finalize |
| CP-08 | Snowflake user and role for the Node service, scratch schema, whether `CREATE SCHEMA` is allowed |
| CP-12 | Sanitized draft HTML samples; whether links stay allowed |
| CP-14 | Sample LLM outputs and Word pastes |
| CP-16 | Confirm the migration number before the file is written |
| CP-17 | Approval to apply the migration |
| CP-33 | Whether block IDs appear in exported HTML |
| CP-49 | Whether to store prompts and raw LLM responses |
| CP-54 | D-B1 answer |
| CP-66 | Retention for new tables |
| CP-72 | Exported draft HTML for the dry run |