# Backend audit — Part 1 (CP-03)

## 1. Access control

| Gate | Where | How |
|---|---|---|
| `assert_deal_access` | `sow_studio/crud.py:78` | Checks user-deal relationship in `PRICING_DEALS` OR `FE_ADMIN` in `current_user.features`. Uses bind variables with `DictCursor`. |
| `authorize_sow_access` | `sow_studio/dependencies.py:8` | FastAPI dependency; requires `FE_SOW_STUDIO`, `FE_SOW_STUDIO_PILOT`, or `FE_ADMIN` in token features. Returns `current_user`. |
| `authorize_sow_native_editor` | `sow_studio/dependencies.py:14` | Requires `FE_SOW_NATIVE_EDITOR` or `FE_ADMIN`. Isolates native editor from production editor. |
| `authorize_token` | `pricing_tool/dependencies.py:36` | Decodes JWT (HS256 via `jose`), fetches role/features from DB, returns `User` Pydantic model with `roles`, `features`, `deal`. |
| Guest auth | `native_collaboration.py:644` | `authorize_guest_session`: validates session token hash against `SOW_NATIVE_GUEST_SESSIONS` joined to invitations; checks revocation and expiry. |
| Public (unauthenticated) | `sow_studio/api/public.py` | `/sow-approval/{token}` and `/sow-share/{token}` — token-scoped, no JWT. Approval returns sanitized draft via `sanitize_rich_text`. Share returns PDF. |

### SOW id convention

`_deal_id_from_sow_id(sow_id)` at `crud.py:52`: `re.fullmatch(r"sow-(\d+)", ...)` extracts `deal_id`.

---

## 2. Draft saving — `save_sow_draft`

**File:** `sow_studio/crud.py:1875–1988`

### Signature

```python
save_sow_draft(conn, deal_id, current_user, content, comments,
               expected_version=None, sow_name=None, added_add_ins=None,
               open_items=None, expected_revision=None, commit=True,
               create_version=False)
```

### Step-by-step flow

1. **`assert_deal_access`** (line 1882).
2. **`_lock_sow_row_for_draft_write`** (line 1884) — `MERGE INTO SOWS` serializes concurrent writes at `crud.py:1537`.
3. **`_normalize_added_add_ins`** / **`_normalize_open_items`** — validate and canonicalize.
4. **SELECT current version** (lines 1892–1897) — JOINs `SOW_DRAFT_VERSIONS` on `SOWS.CURRENT_DRAFT_VERSION_ID`.
5. **Finalized guard** (line 1901–1902): rejects edits to a finalized draft (HTTP 409).
6. **Revision gate** (lines 1903–1906):
   - `expected_revision` is required (HTTP 428 if `None`).
   - Compared against `_draft_revision(prior)` — SHA-256 of `{version, content, comments, open_items, added_add_ins, created_on, sow_name}` (line 1443).
   - Mismatch → HTTP 409.
7. **Version gate** (lines 1907–1909): if `expected_version` provided, must match.
8. **Add-in validation** (line 1916–1917): S22 is only for Fixed Fee.
9. **Open items merge** (lines 1918–1923):
   - Blocking items (`severity == 'blocking'`) cannot be removed by the browser.
   - `_reconcile_open_items_for_saved_content` (line 1412) removes resolved items.
   - `merge_quality_items` imports from `draft_quality`.
10. **`_invalidate_draft_review`** (line 1928): clears evaluation and sign-offs for this deal.
11. **Persist** (lines 1929–1967):
    - If `create_version=True`: INSERT new row in `SOW_DRAFT_VERSIONS` with `next_version`, update `SOWS.CURRENT_DRAFT_VERSION_ID`, log to `SOW_CHANGE_LOG`.
    - If `create_version=False` (default, normal save/autosave): UPDATE existing row in place — sets `CONTENT`, `OPEN_ITEMS`, `ADDED_ADD_INS`, `COMMENTS`, `CREATED_ON=CURRENT_TIMESTAMP()`, `CREATED_BY_EMAIL`, `CREATED_BY_NAME`.
12. **Update SOWS row** (lines 1968–1973): `STATUS='Draft'`, update `SOW_NAME` if provided.
13. **Return** `get_latest_sow_draft` and optionally commit.

### Key rules for the new module

- **Normal save and autosave update in place** (no new version row). `CREATED_ON` is reset to `CURRENT_TIMESTAMP()` on every save.
- **`create_version=True`** is used by intake-field-apply flows, not regular editing.
- The revision hash changes after every save (because `created_on` and `content` change), so the frontend must reload it from the response.
- The `_lock_sow_row_for_draft_write` MERGE is the only real serialization — Snowflake does not enforce PK/UNIQUE.

---

## 3. Draft retrieval

### `get_latest_sow_draft` — `crud.py:1991–2011`

- `assert_deal_access`, then SELECT via `SOWS.CURRENT_DRAFT_VERSION_ID` with a fallback to highest version if no SOWS row.
- Returns `_sow_draft_response(deal_id, row)` which shapes the row and computes `revision` via `_draft_revision`.

### `get_sow_draft_version` — `crud.py:2252–2268`

- Fetches a specific version by `(deal_id, version)`.

### `_sow_draft_response` — `crud.py:1470–1502`

- Shapes DB row to frontend `DraftResponse`.
- Computes and attaches `revision` = `_draft_revision(row)`.
- Parses JSON fields: `open_items`, `added_add_ins`, `comments`, `source_references`, `proposal_scenarios`.

### `_SOW_DRAFT_VERSION_SELECT_COLUMNS` — `crud.py:1504–1511`

```
VERSION, CONTENT, TEMPLATE, IS_PROVISIONAL_TEMPLATE, SOURCE_REFERENCES,
OPEN_ITEMS, ADDED_ADD_INS, COMMENTS, CREATED_ON,
PROPOSAL_STATUS, BASE_VERSION, PROPOSAL_SCENARIOS, PROPOSAL_SUMMARY
```

---

## 4. Draft generation

### `generate_sow_draft` — `crud.py:1593–1873`

AI-powered draft generation. Creates a new immutable version (always `create_version` semantics). Uses Bedrock/Strands via `BEDROCK_MODEL` (default `us.anthropic.claude-sonnet-4-6`). Not in the editing path — called from an intake submit. The new module does not touch this.

---

## 5. Continue draft

### `continue_draft_version` — `sow_studio/continue_draft.py:13–97`

Copies a historical snapshot as a new current version. Locks the row, validates `expected_version` and `expected_revision` against the current draft, validates `source_revision` against the source version. Creates a new version row, updates `SOWS.CURRENT_DRAFT_VERSION_ID`, invalidates review, logs to `SOW_CHANGE_LOG` and `SOW_STATUS_HISTORY`.

---

## 6. Suggested versions

### `create_sow_suggested_version` — `crud.py:2014–2086`

Creates an immutable "proposed" version. Validates base revision. INSERT into `SOW_DRAFT_VERSIONS` with `PROPOSAL_STATUS='proposed'`.

### `decide_sow_suggested_version` — `crud.py:2127–2249`

Accept: archives current, promotes proposed to "current", updates `SOWS.CURRENT_DRAFT_VERSION_ID`, re-runs Deal Desk evaluation. Reject: sets `PROPOSAL_STATUS='rejected'`.

---

## 7. Finalize path

### `publish_final_sow` — `sow_studio/sow_publication.py:405–512`

1. Parse sow_id, assert publication config.
2. Lock SOWS row (`UPDATE SOWS SET UPDATED_ON = UPDATED_ON`).
3. If already finalized: idempotent return (ensure share token exists).
4. `assert_sow_finalizable` (`sow_deal_desk.py:1177`):
   - Requires current quality review, no unresolved process gates or approvals, SOW status must be "Approved".
5. Generate PDF, upload to S3 and HubSpot, attach Note to deal.
6. Persist file references, share token, set status to "Finalized" via `set_sow_status`.

### `set_sow_status` — `sow_deal_desk.py:1062`

State machine transition with `can_transition`. MERGE INTO SOWS. Logs to `SOW_STATUS_HISTORY` and `SOW_CHANGE_LOG`.

### Who can finalize

Only users with `FE_SOW_STUDIO`, `FE_SOW_STUDIO_PILOT`, or `FE_ADMIN` (via `authorize_sow_access` dependency on the route at `routers.py:560`), AND `assert_deal_access` passes, AND the SOW is "Approved".

---

## 8. Native editor (existing evaluation feature)

### Tables touched

| Table | Purpose |
|---|---|
| `SOW_NATIVE_DOCUMENTS` | One row per deal; stores `DOCUMENT_JSON`, `HTML_CONTENT`, `COMMENT_THREADS`, `SUGGESTIONS`, `SOURCE_VERSION`, `REVISION`. |
| `SOW_NATIVE_GUEST_INVITATIONS` | Token-hash-based guest invitations with OTP, 7-day expiry, revocation. |
| `SOW_NATIVE_GUEST_SESSIONS` | Session-token-hash for authenticated guest access, 8-hour expiry. |
| `SOW_NATIVE_AUDIT_EVENTS` | Immutable audit trail (`DEAL_ID`, `EVENT_TYPE`, `PAYLOAD_JSON`, `ACTOR_EMAIL`). |
| `SOW_NATIVE_MENTION_DELIVERIES` | Idempotent mention email tracking. |

### Key functions — `native_collaboration.py`

| Function | Line | Description |
|---|---|---|
| `bootstrap_native_editor` | 277 | Returns latest draft + workspace state. If workspace exists, returns its `html_content`/`document_json`; otherwise returns the draft HTML directly. |
| `save_native_editor_state` | 303 | Saves workspace (document JSON, HTML, threads, suggestions). Validates `expected_revision` and `source_revision`/`source_version` against latest draft. Increments workspace `REVISION`. Does NOT call `save_sow_draft`. |
| `checkpoint_native_editor` | 363 | Calls `save_native_editor_state` then `save_sow_draft` in one transaction. This is the only path from the native editor into `SOW_DRAFT_VERSIONS`. Rejects pending suggestions. |
| `reset_native_editor_to_latest` | 400 | Discards stale workspace, replaces with latest draft content. |
| `_participants` | 167 | Builds participant list from deal roles + guest invitations. |
| Guest functions | 442–797 | Invitation creation/verification/revocation, guest session auth, guest review saving with strict validation (cannot remove/edit existing threads, cannot resolve threads). |

### API routes — `api/native.py`

All authenticated routes use `authorize_sow_native_editor` (`FE_SOW_NATIVE_EDITOR` or `FE_ADMIN`). Guest routes (`/sow-native/guest/*`) use bearer session token.

| Method | Path | Handler |
|---|---|---|
| GET | `/sows/{sow_id}/native-editor/bootstrap` | `bootstrap_native_editor` |
| PUT | `/sows/{sow_id}/native-editor/state` | `save_native_editor_state` |
| POST | `/sows/{sow_id}/native-editor/checkpoint` | `checkpoint_native_editor` |
| POST | `/sows/{sow_id}/native-editor/reload-latest` | `reset_native_editor_to_latest` |
| POST | `/sows/{sow_id}/native-editor/invitations` | `create_guest_invitation` |
| GET | `/sows/{sow_id}/native-editor/invitations` | `list_guest_invitations` |
| DELETE | `/sows/{sow_id}/native-editor/invitations/{id}` | `revoke_guest_invitation` |
| POST | `/sow-native/guest/verify` | `verify_guest_invitation` |
| POST | `/sow-native/guest/mention-session` | `verify_guest_mention_link` |
| GET | `/sow-native/guest/{sow_id}/bootstrap` | `bootstrap_guest_editor` |
| PUT | `/sow-native/guest/{sow_id}/review` | `save_guest_review` |

### Relationship to the new module

The native editor writes to `SOW_NATIVE_*` tables only. It reaches `SOW_DRAFT_VERSIONS` only through `checkpoint_native_editor` → `save_sow_draft`. The new Lexical collab module must NOT write to any `SOW_NATIVE_*` table. Its checkpoint path should also call `save_sow_draft` (the same way the native editor does), preserving the single write path into `SOW_DRAFT_VERSIONS`.

---

## 9. Database and middleware

### `common/database.py`

- SQLAlchemy engine with `snowflake.sqlalchemy.URL`.
- Key-pair auth: `SF_PRIVATE_KEY` PEM → DER.
- **`insecure_mode: True`** in `connect_args` (line 111). Reports it, does not change it.
- Pool: `pool_size=10`, `max_overflow=20`, `pool_timeout=30`, `pool_recycle=1800`.
- `get_db()` yields `engine.raw_connection()`, closes in `finally`.

### `common/config.py`

Feature flags defined as `FeaturesList(str)` constants:

| Flag | Value | Purpose |
|---|---|---|
| `FE_SOW_STUDIO` | `"FE_SOW_STUDIO"` | Production SOW Studio access |
| `FE_SOW_STUDIO_PILOT` | `"FE_SOW_STUDIO_PILOT"` | Pilot group access |
| `FE_SOW_NATIVE_EDITOR` | `"FE_SOW_NATIVE_EDITOR"` | Native editor evaluation |
| `FE_ADMIN` | `"FE_ADMIN"` | Admin override for all SOW features |

New flag `FE_SOW_LEXICAL_EDITOR` will be added at CP-19.

AI model config: `BEDROCK_MODEL` default `us.anthropic.claude-sonnet-4-6`. Anthropic client mode: `strands` (default) or `direct`. Escalation ladder: Haiku → Sonnet → Opus.

### `common/middlewares.py`

Three middleware classes:
1. `ProcessTimeMiddleware`: extracts `deal_id` from query params or JSON body, adds `X-Process-Time` header.
2. `ErrorNotificationMiddleware`: fires Teams notification on 5xx via `AGENT_API_URL`, throttled by 5-minute TTL cache per `method:path`.
3. `TimeoutMiddleware`: wraps `call_next` in `asyncio.wait_for(timeout)`, returns 504 on timeout.

No CORS middleware is defined here; likely configured in the main app setup.

---

## 10. Migrations

### Files in repo

| Number | File | Purpose |
|---|---|---|
| 001 | `001_sow_studio_foundation.sql` | Base SOW tables |
| 002 | `002_sow_studio_hardening.sql` | Publication columns on SOWS |
| 003 | `003_sow_studio_names.sql` | SOW_NAME, naming fields |
| 004 | `004_sow_studio_add_ins.sql` | ADDED_ADD_INS column |
| 005 | `005_sow_studio_review_versions.sql` | Proposal/review version columns |
| 006 | `006_sow_studio_finalize_share.sql` | Share token + expiry on SOWS |
| 007 | `007_sow_signoff_workflow.sql` | SOW_SIGNOFFS table |
| 008 | `008_sow_notification_delivery_metadata.sql` | Delivery metadata |
| 009 | `009_sow_signoff_decision_timestamp.sql` | DECIDED_ON column |
| 010 | `010_sow_signoff_evidence_metadata.sql` | Evidence verification fields |

Also in repo: `db/migrations/002_pricing_template_deletion.sql`, `db/migrations/003_fractional_duration.sql` (pricing-tool, unrelated).

### Missing migrations (011–013)

Per `README_SOW_STUDIO_STAGING.md` line 7: _"the unused manual-overrides migration 012 and unimplemented Native Editor migration 013 were removed."_ There is no 011 in the repo either.

**Next migration number: 011.**

The `SOW_NATIVE_*` tables (`SOW_NATIVE_DOCUMENTS`, `SOW_NATIVE_GUEST_INVITATIONS`, `SOW_NATIVE_GUEST_SESSIONS`, `SOW_NATIVE_AUDIT_EVENTS`, `SOW_NATIVE_MENTION_DELIVERIES`) are referenced in code but their DDL (migration 013) was removed from the repo. They exist in the live database (the code runs), but there is no checked-in DDL.

---

## 11. Reuse table

| Function / artefact | File : lines | Lexical module reuses? | Notes |
|---|---|---|---|
| `assert_deal_access` | `crud.py:78` | **Yes** — call, never rewrite | Core ACL gate |
| `authorize_sow_access` | `dependencies.py:8` | **Yes** — FastAPI dep for routes | Add `FE_SOW_LEXICAL_EDITOR` to the set at CP-19 |
| `authorize_token` | `pricing_tool/dependencies.py:36` | **Yes** — inherited via deps | JWT + role/features resolution |
| `save_sow_draft` | `crud.py:1875` | **Yes** — call from checkpoint endpoint | Single write path into `SOW_DRAFT_VERSIONS` |
| `get_latest_sow_draft` | `crud.py:1991` | **Yes** — call to load initial HTML | Returns content + revision |
| `get_sow_draft_version` | `crud.py:2252` | **Yes** — historical version read | |
| `_draft_revision` | `crud.py:1443` | **Yes** — compute revision for concurrency | SHA-256 hash |
| `_lock_sow_row_for_draft_write` | `crud.py:1537` | **Yes** — call from checkpoint | Row-level serialization via MERGE |
| `_invalidate_draft_review` | `crud.py:1454` | Indirectly (via `save_sow_draft`) | Clears evaluation + sign-offs |
| `_sow_draft_response` | `crud.py:1470` | Indirectly (via `get_latest_sow_draft`) | Response shaping |
| `get_db` | `common/database.py:119` | **Yes** — FastAPI dep | Connection lifecycle |
| Feature flags | `common/config.py:121–124` | **Yes** — import and extend | |
| `publish_final_sow` | `sow_publication.py:405` | **Must not break** | Finalize reads from `SOW_DRAFT_VERSIONS.CONTENT` |
| `assert_sow_finalizable` | `sow_deal_desk.py:1177` | **Must not break** | Checks approvals, quality, status |
| `set_sow_status` | `sow_deal_desk.py:1062` | **Must not break** | State machine: Draft → In Review → Approved → Finalized |
| `continue_draft_version` | `continue_draft.py:13` | **Must not break** | Copies historical version as new current |
| `sanitize_rich_text` | `application/rich_text.py` (via `api/public.py`) | **Must not break, never edit** | Used for public approval preview; drops `class`/`style`, no `mark` tag |
| Native editor functions | `native_collaboration.py` | **Must not touch** | Separate evaluation feature with own `SOW_NATIVE_*` tables |

---

## 12. Do-not-break list

1. **`save_sow_draft` contract** — the only write path into `SOW_DRAFT_VERSIONS`. The Lexical module's checkpoint calls this; it must never write to `SOW_DRAFT_VERSIONS` directly.
2. **Revision-based optimistic locking** — `expected_revision` must be sent on every save; `_draft_revision` computes SHA-256 of the full row. Changing any hashed field breaks all concurrent sessions.
3. **`_lock_sow_row_for_draft_write`** — MERGE-based serialization. Any code that writes to SOWS or SOW_DRAFT_VERSIONS must call this first.
4. **Finalize pipeline** — reads `CONTENT` from `SOW_DRAFT_VERSIONS` via the approved version. The HTML the Lexical editor checkpoints must be valid for PDF generation.
5. **`sanitize_rich_text`** — used in public approval preview. The Lexical HTML contract must produce output that survives this sanitizer (it drops all `class`/`style` attributes and allows no `mark` tag).
6. **Status state machine** — Draft → In Review → Approved → Finalized. The module must not bypass `set_sow_status`.
7. **`SOW_NATIVE_*` tables** — the new module must never read or write these; they belong to the native editor evaluation.
8. **`SOW_CHANGE_LOG` and `SOW_STATUS_HISTORY`** — append-only audit tables. The module should not write to these directly; they are populated by `save_sow_draft`, `continue_draft_version`, and `set_sow_status`.

---

## 13. Open questions

1. **Migration 013 DDL**: The `SOW_NATIVE_*` tables exist in the live database but their migration (013) was removed from the repo. Should we keep the new module's migration numbering at 011, or does an applied-but-removed 011 exist in the live DB? **Ask user.**
2. **`insecure_mode: True`**: `common/database.py:111` connects to Snowflake with TLS certificate verification disabled. Noted; not changed per standing rules.
3. **`sanitize_rich_text` vs Lexical HTML**: The public approval endpoint sanitizes content with a function that strips all `class` and `style` attributes and has no `mark` tag allowance. The Lexical HTML contract (CP-05) must produce output that is meaningful after this sanitization. Does the product team accept that placeholders (`<mark class="sow-studio-placeholder-marker">`) will not appear in the approval preview?
4. **CORS configuration**: No CORS middleware was found in `common/middlewares.py`. Where is it configured? Needed to confirm the collab WebSocket origin policy.

---

# Backend audit — Part 2 (CP-04): HTML dependencies and LLM paths

## 14. Every piece of code that depends on draft HTML

### 14a. Backend HTML consumers

| # | Consumer | File : lines | How it reads HTML | Tags / attrs it cares about | What breaks if HTML structure changes |
|---|---|---|---|---|---|
| 1 | `sanitize_rich_text` | `application/rich_text.py:76` | `_RichTextSanitizer` (HTMLParser) — allowlist filter | Allowed: a, b, blockquote, br, code, div, em, h1-h6, hr, i, li, ol, p, pre, s, span, strong, sub, sup, table, tbody, td, th, thead, tr, u, ul. Attrs: title (any), href+target (a), colspan+rowspan (td/th). **Strips ALL class/style. No mark, figure, img, font, section.** | Any tag outside the allowlist is silently dropped. Any class/style attr is stripped. Called only from `api/public.py:36` for approval preview. |
| 2 | `_HtmlToBlocksParser` | `sow_docx_export.py:80` | HTMLParser → ParagraphBlock / TableBlock | Block: p, h1-h6, li, ul, ol. Inline: strong, b, em, i, u. Table: table, tr, td, th. Also: br. **No mark, figure, span, font-size, font-family, class, style. Tables become flat text cells — no colspan/rowspan.** | Unknown tags become inline text (data handler fires). `<figure class="table">` wrapper is ignored — table still parses. But colspan/rowspan are lost; tables flatten to single-cell grids. |
| 3 | `_append_html_story` | `sow_pdf_export.py:133` | Reuses `_HtmlToBlocksParser` from docx export → ReportLab Paragraph/Table | Same as #2 | Same as #2 |
| 4 | `plain_text()` | `draft_quality.py:38` | Regex: `re.sub(r'</(?:p\|li\|h[1-6]\|tr\|td\|th)>\|<br\s*/?>', '\n', ...)` then `re.sub(r'<[^>]+>', ' ', text)` | Block-boundary tags for newlines; all other tags stripped | Structure-agnostic — works on any well-formed HTML. Adding new block tags won't break it (they become spaces). |
| 5 | `quality_issues()` | `draft_quality.py:62` | `_HtmlToBlocksParser` for table checks + `plain_text()` for text checks | Same as #2 for tables; plain text for: termination options, delivery model, signatory placeholders, commitment dates | Table structure must remain parseable (standard `<table><tr><td>`). Text checks work on plain text — agnostic. |
| 6 | `_bracketed_placeholder_tokens()` | `crud.py:974` | Character-by-character balanced bracket scanner on raw HTML string | `[` and `]` characters | Works on raw HTML — if HTML entities encode brackets (`&#91;`), they are invisible to this scanner. Lexical must emit literal brackets. |
| 7 | `_signatory_fields_are_complete()` | `crud.py:1334` | Two-pass regex: (1) `<tr>` → `<td>` extraction, strip tags, `html.unescape`; (2) fallback: find "IN WITNESS WHEREOF", strip all HTML, regex for NAME/TITLE fields | `<tr>`, `<td>`, `<br>`, `</p>`, `</div>`, `</li>`, inline tags stripped. Looks for "NAME:", "TITLE:", "SIGNATURE:" text patterns. | Table path: needs standard `<tr><td>` structure (no `<figure>` wrappers between `<table>` and `<tr>`). Fallback path: structure-agnostic (strips all HTML). Both paths: values must not contain `[…]` or `__` patterns. |
| 8 | `_team_composition_has_completed_row()` | `crud.py:1371` | Regex: find heading matching "Team Composition" or "Staffing" → extract section → find `<table>` → iterate `<tr>` → extract `<td>` cells | `<h1>`-`<h6>` headings, `<table>`, `<tr>`, `<td>`, `<th>` (skipped as headers). Strips inline tags from cell text. | Heading must be recognizable (numbered prefix OK). Table must use standard `<table><tr><td>` without extra wrappers inside. Cell text must not contain `[…]` or `__`. |
| 9 | `_fixed_fee_is_complete()` | `crud.py:1402` | Regex: `<p>` extraction → strip tags → `html.unescape` → pattern match for "Client will pay Blend a total fixed fee" with dollar amount | `<p>` tags only. Inline tags stripped. | Fee statement must be in a `<p>` tag (not a `<div>` or `<li>`). Dollar pattern must match `[$€£]?\d[\d,.]*`. No `[…]` or `__`. |
| 10 | `completed_billing_schedule()` | `billing_schedule.py:35` | `_Tables` HTMLParser class: extracts all `<table>` → rows → cells as text. Also regex on `plain_text` for T&M invoicing sentence. | `<table>`, `<tr>`, `<td>`, `<th>`. Cell text is joined whitespace-normalized. Header matching is case-insensitive (`month`, `monthly fee`, `payment milestone`, `amount`, `invoice trigger`). | Tables must use standard structure. `<figure>` wrapper is ignored (HTMLParser treats it as unknown, data still flows). Header text must match expected patterns exactly. T&M path uses plain text regex — agnostic. |
| 11 | `_reconcile_open_items_for_saved_content()` | `crud.py:1412` | Orchestrator: calls #6, #7, #8, #9, #10 above | Delegates to sub-functions | Breaks if any sub-function breaks |
| 12 | `_draft_revision()` | `crud.py:1443` | SHA-256 of row including `CONTENT` | Treats content as opaque string | Never breaks — just produces a different hash when content changes |
| 13 | `sow_signoffs._lookup()` | `sow_signoffs.py:6` | SQL fetch of `dv.CONTENT` → pass-through to response | None — opaque pass-through | Never breaks |
| 14 | `sow_publication._approved_artifact()` | `sow_publication.py:~334` | Reads `draft["content"]`, passes to `build_sow_pdf_bytes()` | Delegates to #3 | Breaks if #3 breaks |
| 15 | `publish_final_sow()` | `sow_publication.py:405` | Reads HTML from `SOW_DRAFT_VERSIONS.CONTENT` via #14 | Delegates to #3 | Breaks if #3 breaks |
| 16 | `continue_draft_version()` | `continue_draft.py:13` | SQL INSERT…SELECT copies `CONTENT` column | None — opaque copy | Never breaks |
| 17 | `source_coverage.review_coverage()` | `source_coverage.py:190` | `plain_text(draft)` → `QuoteIndex(visible_draft)` | Delegates to #4 | Structure-agnostic |
| 18 | `quote_grounding.QuoteIndex` | `quote_grounding.py:1` | Typography-normalized case-folded substring matching on plain text | None — works on pre-stripped text | Never breaks |
| 19 | `draft_extraction` | `draft_extraction.py:1` | `plain_text(sow_draft_text)` → model | Delegates to #4 | Structure-agnostic |

### 14b. Frontend HTML consumers (summary; details in `audit_frontend.md`)

| # | Consumer | File | How it reads HTML | CKEditor-specific? |
|---|---|---|---|---|
| F1 | `sowStudioAddInInsertion.ts` | services/ | `DOMParser` on HTML string | No — DOM-generic |
| F2 | `sowStudioAddInPresence.ts` | services/ | `DOMParser` + `document.querySelector(".ck-editor__editable")` | **Yes** — `.ck-editor__editable` selector |
| F3 | `sowStudioIntakeDraftSync.ts` | services/ | `DOMParser` on HTML string | No — DOM-generic |
| F4 | `sowStudioOpenItems.ts` | services/ | `DOMParser` on HTML string | No — DOM-generic |
| F5 | `addInNavigation.ts` | services/ | `document.querySelector(".ck-editor__editable")` | **Yes** — `.ck-editor__editable` selector |
| F6 | `openItemNavigation.ts` | services/ | `document.querySelector(".ck-editor__editable")` | **Yes** — `.ck-editor__editable` selector |

CKEditor-specific selectors (F2, F5, F6) must be branched or replaced at CP-19/CP-20 when the feature flag routes to the Lexical editor.

---

## 15. LLM paths — every place generated text enters the draft

| # | Agent | File : lines | Input to model | Output format | How output reaches the draft | Lexical module impact |
|---|---|---|---|---|---|---|
| 1 | `sow_narrative_agent` | `sow_narrative_agent.py:1–282` | Sources + deal evidence (structured prompt) | JSON object: `{scope_sufficient, project_description, scope, deliverables, assumptions, exclusions, …}` — **NOT HTML** | Frontend template renderer assembles HTML from JSON fields, then user saves via `save_sow_draft` | None — output is JSON, insertion is via the editor's save path |
| 2 | `suggested_fix_agent` | `suggested_fix_agent.py:1–106` | `draft_text` (complete HTML) | `SuggestedFixResult.proposed_content` = **complete HTML draft** + `summary` + `scenario_keys` | New `SOW_DRAFT_VERSIONS` row via `create_sow_suggested_version` (crud.py:2014) | **High risk**: model returns CKEditor-era HTML. Must be re-prompted or post-processed if Lexical HTML differs (e.g., no `<figure class="table">`, different class names). Also has deterministic regex path `_time_and_materials_monthly_in_arrears_fix`. |
| 3 | `draft_extraction` | `draft_extraction.py:1–232` | `sow_draft_text` (HTML) → `plain_text()` strips it | `DraftExtractionResult`: billing_schedule (text excerpt) + field conflict lists | **Does not enter the draft** — feeds Deal Desk evaluation flags | None — reads HTML, strips it, never writes back |
| 4 | `source_coverage` | `source_coverage.py:1–244` | `draft` (HTML) → `plain_text()` strips it | Coverage classification against source commitments | **Does not enter the draft** — feeds Deal Desk evaluation | None — reads HTML, strips it, never writes back |
| 5 | `draft_quality` | `draft_quality.py:1–179` | `content` (HTML) — both `_HtmlToBlocksParser` and `plain_text()` | Quality issue list (missing fields, placeholder warnings) | Merged into `open_items` JSON via `merge_quality_items` (not into HTML content) | None — writes to `open_items`, not `CONTENT` |

### AI model configuration

- `BEDROCK_MODEL`: default `us.anthropic.claude-sonnet-4-6` (config.py).
- Anthropic client mode: `strands` (default) or `direct`.
- Escalation ladder: Haiku → Sonnet → Opus (used by `draft_extraction`).
- `suggested_fix_agent` system prompt: _"Return the entire revised HTML draft, not markdown and not a diff."_ — SUPPORTED_SCENARIOS: `billing_schedule_missing`, `engagement_type_wording_mismatch`, `legal_risk_review`.

---

## 16. Test inventory — backend tests that touch draft HTML

| Test file | Lines | DB pattern | What it covers | CKEditor-specific markup? |
|---|---|---|---|---|
| `test_sow_docx_export.py` | ~50 | Direct function call, no DB | `build_sow_docx_bytes` with simple `<p>` HTML | No |
| `test_sow_rich_text.py` | ~38 | Direct function + `TestClient` | `sanitize_rich_text` stripping; public approval endpoint CSP headers | `colspan` in test HTML |
| `test_sow_http_e2e.py` | ~100+ | `class _Connection: pass`, `app.dependency_overrides[get_db]`, `app.dependency_overrides[authorize_sow_access]`, `patch()`, `TestClient(app)` | HTTP endpoints, save flow, version management | No |
| `test_native_collaboration.py` | ~80+ | `conn = Mock()`, `monkeypatch.setattr` | Guest review validation, thread immutability | No |
| `test_sow_draft_add_ins.py` | ~160+ | Direct function call | `_reconcile_open_items_for_saved_content` with real HTML | **Yes**: `<figure class="table"><table><tbody>` (line 121), signature tables |
| `test_signature_layout.py` | ~30+ | Direct function call | Signature table detection | **Yes**: `<figure class="table">` wrapper (line 14) |

### Fake connection patterns (for reuse in new module tests)

1. **`test_sow_http_e2e.py`**: `class _Connection: pass` → `app.dependency_overrides[get_db] = lambda: _Connection()` and `app.dependency_overrides[authorize_sow_access] = lambda: _user`. Uses `patch()` for individual sow functions. `TestClient(app)` for HTTP.
2. **`test_native_collaboration.py`**: `conn = Mock()` (from `unittest.mock`), `monkeypatch.setattr` to replace functions.

### CKEditor markup found in backend tests

- `<figure class="table">` — `test_sow_draft_add_ins.py:121`, `test_signature_layout.py:14`
- `colspan` — `test_sow_rich_text.py`
- No `<mark>`, no `style=`, no `data-add-in` attributes in any backend test

---

## 17. Combined open questions (CP-03 + CP-04)

### From CP-03

1. **Migration numbering**: Does an applied-but-removed 011 exist in the live DB, or is 011 safe as the next number?
2. **`sanitize_rich_text` vs placeholders**: The public approval endpoint strips all `class`/`style` and has no `<mark>` tag. Placeholders (`<mark class="sow-studio-placeholder-marker">`) will not appear in the approval preview. Is this acceptable?
3. **CORS configuration**: Where is CORS configured? Not found in `common/middlewares.py`. Needed for collab WebSocket origin policy.

### From CP-04

4. **`<figure class="table">` wrappers**: CKEditor wraps tables in `<figure class="table">`. The DOCX/PDF export parser (`_HtmlToBlocksParser`) ignores `<figure>` — it does not break, but the tag is noise. `sanitize_rich_text` also drops it. The Lexical HTML contract (CP-05) should NOT emit `<figure>` wrappers. **Confirm this approach.**
5. **`suggested_fix_agent` HTML output**: This agent returns a complete HTML draft using CKEditor-era markup. If the Lexical HTML contract differs from CKEditor's output (e.g., no `<figure>`, different class names), the agent's system prompt and/or a post-processing step must be updated. **When should this happen — CP-05 (contract) or a later checkpoint?**
6. **`completed_billing_schedule` header matching**: The `_Tables` parser in `billing_schedule.py` matches table headers case-insensitively (`month`, `monthly fee`, `payment milestone`, `amount`, `invoice trigger`). The Lexical editor must preserve these exact header texts in billing schedule tables. **Note for CP-05 contract.**
7. **`_signatory_fields_are_complete` regex parsing**: Uses regex to find `<tr>/<td>` and text patterns like `NAME:`, `TITLE:`, `SIGNATURE:`. Lexical's HTML output for tables must use standard `<table><tr><td>` structure (not wrapped in extra elements). **Note for CP-05 contract.**
8. **`_bracketed_placeholder_tokens` operates on raw HTML**: If Lexical HTML-encodes bracket characters (`&#91;`, `&#93;`) instead of emitting literal `[` and `]`, placeholder detection will silently fail. **The HTML contract must require literal brackets.**
9. **Test fixtures with CKEditor markup**: `test_sow_draft_add_ins.py` and `test_signature_layout.py` use `<figure class="table">` in test HTML. These tests exercise backend functions that must also work with Lexical HTML. **Should we add parallel Lexical-format fixtures, or update the existing ones?**
