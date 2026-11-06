# SOW Editor Module — Documentation

This folder is the running record of the SOW Editor Module build: a Lexical + Yjs live-collaboration
editor that replaces CKEditor inside SOW Studio. It is meant to be read by anyone joining the project
later, including a future version of the assistant, so it explains not just *what* exists but *why*.

It is appended to as work continues — never rewritten from scratch. If something here goes stale,
correct it in place and note the date, the same way the checkpoint tracking files are kept.

## Files in this folder

| File | Covers |
|---|---|
| `README.md` (this file) | The whole story so far: why this project exists, how it is organized, what has been built, what decisions were made and why, what is still open. |
| `frontend.md` | Everything on the `blend_ode` (React/CRA) side: the audit of the existing CKEditor component, the shared contract code, the role table, the test setup. |
| `backend.md` | Everything on the `blend_ode_be` (FastAPI) side: the audit of existing draft-handling code, the Node collaboration service, the contracts and ADRs, the tooling. |

## Why this project exists

SOW Studio's current rich-text editor is CKEditor 5, wired directly into `DraftEditor.tsx`. It works,
but it has no true multi-user collaboration: two people editing the same SOW at once can silently
overwrite each other, protected only by an optimistic revision check that rejects the second save
with an error. The goal of this module is to replace CKEditor with **Lexical** (Meta's editor
framework) wired to **Yjs** (a CRDT library) through a **Node.js collaboration service**, so multiple
people — including external reviewers — can edit or comment on a SOW live, the way Google Docs works.

The constraint that shapes every decision below: **the saved SOW document must stay HTML**, stored
exactly where it is stored today (`SOW_DRAFT_VERSIONS.CONTENT` in Snowflake), because a large amount
of existing code reads that HTML — DOCX/PDF export, the approval-preview sanitizer, quality checks,
sign-off lookups, and several AI agents. None of that code can be asked to understand Yjs or Lexical's
internal document model. So Yjs/Lexical is *live editing state*; HTML remains *the system of record*.
That single sentence is the organizing principle of the whole design.

## How the work is organized

The build follows a fixed, numbered sequence of prompts ("checkpoints", CP-00 through CP-74), defined
in `SOW_EDITOR_BUILD_PROMPTS_v1.md` and `SOW_EDITOR_RUN_GUIDE_v1.md` at the repository root. Each
checkpoint is a small, scoped piece of work — an audit, a contract, a piece of tooling, eventually a
piece of implementation — with its own acceptance checks and a mandatory stop-and-test point before
the next one begins. This is deliberate: it means at every point so far the state of the module is
either "audit only, nothing can break" or "typed/tested tooling that nothing else depends on yet."
No application code has been changed. No migration has touched Snowflake. Nothing has been deployed.

Every checkpoint's outcome is recorded in `sow_docs/tracking/prompt_log.md` (one entry
per checkpoint: summary, decisions/assumptions, a registry of new or changed files, open questions,
and a handoff line to the next checkpoint). This documentation folder is a *reader-friendly narrative*
built from that log and from the two audit and two decision files it points to — read this first, and
go to `sow_docs/tracking/` only when you need the line-by-line detail (exact file paths and line numbers).

### Two repositories, one plan

The codebase is split across two separate git repositories that cannot see each other at build time:

- **`blend_ode`** — the React frontend (Create React App 5), where SOW Studio and its CKEditor-based
  draft editor live today.
- **`blend_ode_be`** — the FastAPI backend, where the SOW Studio API, the database access layer, and
  (going forward) the Node.js collaboration service live.

Because Create React App can only import files inside its own `src/`, and the two repos are not built
together, any TypeScript code that both the frontend and the future Node service need to agree on
(role definitions, the WebSocket protocol, the HTML contract) is **written once, in the frontend repo,
and copied ("vendored") into the backend repo by a script with a hash check**. This is explained in
full in `frontend.md` and `backend.md`; the short version is: `blend_ode/src/sow_module/shared/` is
the one true copy, and nobody is allowed to hand-edit the copy that lands in `blend_ode_be`.

## What has been done, checkpoint by checkpoint

### CP-00 — Scaffolding
Created the folder layout and the tracking files in both repos, and recorded the facts about each
repository (versions, how CKEditor is wired in, where migrations live, etc.) in `CLAUDE.md` so later
work has a settled factual foundation instead of re-discovering it each time.

### CP-01 and CP-02 — Frontend audit (read-only)
Before touching anything, a full audit was done of the *existing* CKEditor-based editor:

- `DraftEditor.tsx` — every one of its 19 props, its 18 CKEditor plugins, its 7 highlight colors
  (used for things like pending placeholders — e.g. `[Client Name]`), its 7 font sizes and 9 font
  families, and every SOW-specific behavior layered on top of CKEditor (a "DRAFT" watermark, a
  provisional-pricing banner, placeholder auto-removal when a bracket is filled in, add-in navigation,
  download/copy-link buttons). This produced a 41-row **parity table**: for every CKEditor feature,
  what the Lexical equivalent will be, and whether that equivalent already exists in Lexical's
  ecosystem or needs to be built ("TBD").
- `DraftEditorScreen.tsx` — the save and autosave flow (a 15-second autosave timer, an optimistic
  revision lock, a "generation counter" so a slow save doesn't clobber newer edits), the **exact swap
  point** where the new editor will eventually replace the old one (`DraftEditorScreen.tsx:1097`), and
  six service files that read or write the draft's HTML directly with `DOMParser` (add-in insertion,
  add-in presence detection, intake-field sync, open-item detection, and two navigation helpers).

Why this mattered: without this audit, nobody could say with confidence what the new editor has to
reproduce, or where exactly it plugs in. The full detail is in `audit_frontend.md`; the narrative
version is in `frontend.md`.

### CP-03 and CP-04 — Backend audit (read-only)
The same discipline, applied to the backend:

- CP-03 covered access control, the `save_sow_draft` function (the **single** function that is allowed
  to write a draft into the database, with its revision-locking and row-locking logic), draft
  retrieval, AI draft generation, the "continue from a historical version" flow, the finalize pipeline
  (which turns an approved SOW into a PDF and sends it to HubSpot), the existing (separate,
  evaluation-only) native editor feature, the database connection setup, and all ten existing SQL
  migrations.
- CP-04 covered every piece of code, anywhere in the backend or frontend, that reads or writes the
  draft's HTML: the DOCX/PDF export parser, the approval-preview sanitizer, the quality-check engine,
  sign-off lookups, and five different AI agent code paths (only one of which, `suggested_fix_agent`,
  actually writes HTML back into the draft).

Why this mattered: this is what makes the HTML contract (below) possible. Every one of these backend
functions has quiet assumptions about the HTML it is given — e.g. a signature block must be a
`<table>` with exactly two `<td>` cells, a billing-schedule table's header cells must say exactly
"Month" and "Monthly fee", a placeholder's brackets must be literal `[` and `]` characters, not HTML
entities. If the new Lexical editor emitted HTML that violated any of these unwritten rules, SOW
Studio's finalize, export, and quality-check features would quietly break — not with an error, but
with wrong data in a legal document. The full detail is in `audit_backend.md`; the narrative is in
`backend.md`.

### CP-05 — Contracts, roles and decisions
With both audits in hand, this checkpoint wrote down, as actual TypeScript types (not just prose),
everything the three parties (the browser, the Node collaboration service, and FastAPI) need to agree
on, and a prose contract document explaining it:

- **Roles**: `viewer`, `commenter`, `editor`, `sow_admin`.
- **The WebSocket protocol**: what messages travel between the browser and the Node service, and what
  each closing code means (e.g. "your access was removed," "this document is locked/finalized,"
  "another server instance owns this document").
- **The HTML contract**: a machine-readable list of exactly which HTML tags, classes, and inline
  styles the new editor is allowed to produce, derived from the real CKEditor output found in the
  audit (not from an existing unrelated sanitizer that is far too strict for this purpose).
- **The internal API between the Node service and FastAPI**: how the Node service asks "is this user
  allowed in, and with what role," how it fetches the current draft HTML to seed a session, and how it
  hands a finished edit back to FastAPI to be saved.
- **A security design for the short-lived WebSocket "ticket"**: a 30-second, single-use, signed token
  — deliberately signed with a *different* secret than the normal login token, so a leaked ticket can
  never be used to log in, and a leaked login token can never be used to open a collaboration session.
- **11 Architecture Decision Records (ADRs)** — each a short, numbered, written-down decision with its
  reasoning and status, covering things like: HTML stays the system of record; the only way to save a
  draft is still the existing `save_sow_draft` function; the Node service is a genuinely separate
  deployable (not folded into Python, because validating a Lexical/Yjs document needs the actual
  Lexical JavaScript packages); how a "lease" with a fencing token prevents two server instances from
  fighting over the same document; and who owns the shared TypeScript code.

Why ADRs specifically: a project like this makes a lot of decisions that are each individually
reasonable but collectively easy to forget the reasoning behind, six months later, when someone asks
"why does the Node service never touch the native editor's tables?" Writing each one down with a
status (`Proposed` until the user signs off) means decisions can be revisited deliberately instead of
being silently reinterpreted by whoever touches the code next.

### CP-06 — Frontend module made real and tested
CP-05's `roleToCapabilities` function was a deliberate stub (it returned "no capabilities" for every
role, so that nothing could silently default to too much access). CP-06 implemented it for real: a
table from role to a list of capabilities (`read`, `comment`, `suggest`, `edit`, `resolve_thread`,
`checkpoint`, `rename`, `manage_access`, `finalize`), where each role is a strict superset of the
one before it, and where a "feature flag" style override can only ever *remove* a capability from a
role, never *add* one. Nine Jest tests pin this behavior down, including the specific scenario
requested: an editor can edit but cannot finalize or rename the SOW; only `sow_admin` can do either.

This checkpoint also confirmed two basic facts that everything after it depends on: the shared
TypeScript code type-checks cleanly under TypeScript's strictest settings, and it contains no import
from React, antd, Node.js, or the browser DOM — i.e., it really is portable, pure logic that both a
browser and a Node.js server can run unmodified.

### CP-07 — Node service and backend package made real and tested
This finished the plumbing needed before any real collaboration logic can be written: the Node service
now has its own `package.json` (with a short, deliberately small dependency list — Express for a
minimal HTTP surface, `ws` for WebSockets, Yjs and `y-protocols` for the CRDT sync protocol, Zod for
validation, Pino for logging — no database driver yet, on purpose), a strict TypeScript config, a test
runner (Vitest), a linter, and a `DataStore` interface describing every way the service will eventually
read or write persistent data (documents, live edit updates, periodic snapshots, "who currently owns
this document" leases, access-revocation records, and an audit log) — backed for now by a
fully-working **in-memory fake**, so the actual collaboration logic can be built and tested completely
before a single line of Snowflake-specific code is written. The backend repo got a matching smoke test
and a "did someone edit the vendored copy by hand" drift test. A single script, `run_checks.sh`, now
runs all seven checks (both repos, the Node service, and the sync check) and prints one summary.

## Key decisions and the reasoning behind each

These are expanded versions of the ADRs in `backend.md`; read that file for the full list. The ones
most worth understanding up front:

1. **HTML is, and remains, the system of record.** The Yjs/Lexical document only exists while people
   are actively editing. At any point it can be thrown away and rebuilt from the saved HTML with no
   data loss (other than unsaved live edits), because the HTML is what every other system — export,
   finalize, quality checks, sign-off — actually reads.
2. **There is still only one function that writes a draft to the database.** The Node service never
   talks to Snowflake's `SOW_DRAFT_VERSIONS` table directly. It calls FastAPI, which calls the existing
   `save_sow_draft`, which does the revision-locking that already protects against two people saving
   over each other. This means none of SOW Studio's existing save-related guarantees (the
   finalized-draft guard, the open-items reconciliation, the audit trail) need to be reimplemented or
   risk being reimplemented wrong.
3. **The existing HTML sanitizer (`sanitize_rich_text`) is never used for this module.** It was built
   for a different purpose (stripping an approval-preview page down to the bare minimum) and would
   delete the highlight colors, placeholders, and table styling that the new editor needs to produce.
   Instead, the module gets its own allowlist, driven by a single JSON file (`html-contract.json`) that
   both the frontend and backend can check their output against.
4. **The Node service is a real, separate server, not a Python module.** Validating that a document
   produced by Lexical's collaborative editing is well-formed requires Lexical's own JavaScript
   packages; there is no way to do that validation correctly from Python. This does mean the project
   has two things to deploy instead of one, which is an accepted cost.
5. **A document has exactly one "owner" server instance at a time, enforced by a database lease with a
   fencing token.** This is the same pattern (`MERGE INTO`) the existing codebase already uses to avoid
   race conditions in Snowflake, which does not enforce uniqueness constraints on its own. Without this,
   if two Node server instances both thought they owned the same document, they could each save
   different versions and the later one would silently win.
6. **Shared TypeScript code is owned by the frontend repository and copied into the backend repository
   by a script, with a SHA-256 hash recorded in both places.** This exists purely because of a
   structural fact: Create React App cannot import files from outside its own `src/` folder, and the
   two repositories cannot see each other during a build. Writing the same types twice would guarantee
   they eventually drift out of sync; a human is not allowed to hand-edit the copy, only the script is.

## What is still open — and what you need to know before touching this

- **Who exactly is a `sow_admin`?** Today, finalizing a SOW uses exactly the same permission gate as
  editing one — there is no existing "only some editors can finalize" concept in the code. So the
  mapping of real people to the `sow_admin` role cannot be read off the existing system; this needs a
  product decision, not a technical one. Currently Blend users are editors by default, and the
  `sow_admin` boundary is still undefined.
- **External (client-side) access is planned, not built.** The direction agreed so far: an external
  person is treated like a "commenter," is let in by a secret-key-style credential (not a normal login)
  and their access is explicitly approved via a request flow, reusing the pattern the existing (separate)
  native-editor feature already uses for guest reviewers. The actual mechanism has not been designed.
- **Storage location is fixed, but not yet connected.** The user has specified that this module's
  database tables must live in schema `BLEND_INTERNAL_IT_APPS.NEXDRAFT`, using the Snowflake user
  `PRICING_APP_DEV_USER` — and *only* those two things, nothing else. This is recorded as ADR-012 and
  in `ENV_REMINDER.md` at the repository root. No `.env` files exist yet; the user will add them later.
  Before connecting anything for real, someone needs to confirm that `PRICING_APP_DEV_USER` actually
  has the right Snowflake grants, and that the *existing* SOW tables (which this module's checkpoint
  path still needs to read and write) are reachable from that same connection.
  **No SQL migration will be applied to the shared Snowflake database without the user's explicit
  sign-off at CP-17** — this is a standing rule, not a suggestion.
  - A small number of HTML details (exact inline styles on tables, column widths, text alignment on
  table cells) are marked "unverified" in the HTML contract because no real, saved SOW draft was
  available to check them against. The next real test should use actual saved drafts, not just
  hand-written test fixtures.
- **Two pre-existing frontend tests are currently failing** (`DraftEditorScreen.integration.test.tsx`
  and `SowStudioPage.test.tsx`), apparently from test timeouts rather than anything this module
  changed — no file outside the module imports anything from it. This has not yet been confirmed
  against a clean checkout of the branch; do that before assuming it is unrelated.

## How to run the checks yourself

```bash
bash blend_ode_be/sow_module/run_checks.sh
```

This runs, in order: a check that the shared TypeScript hasn't drifted between the two repos, the
frontend's Jest tests for the module, a strict TypeScript check on the shared code, a TypeScript check
and ESLint pass on the Node collaboration service, the Node service's Vitest suite, and the backend's
Python smoke tests — then prints a pass/fail summary line for each. Every checkpoint above ends with
this script passing before moving on.

## Where to look for more detail

- `sow_docs/tracking/prompt_log.md` — one dated entry per checkpoint, the ground truth
  for exactly what changed and when.
- `sow_docs/tracking/audit_frontend.md` / `audit_backend.md` — the full audits, with
  exact file paths and line numbers, underlying `frontend.md` and `backend.md`.
- `sow_docs/tracking/contracts.md` — the full prose contract (endpoints, ticket format,
  WebSocket lifecycle, role mapping).
- `sow_docs/tracking/decisions.md` — the full list of ADRs and the user's answers to
  open questions, as they come in.
- `sow_docs/tracking/test_log.md` — a dated table of every test run and its result.
- `blend_ode_be/sow_module/CLAUDE.md` — the standing rules this entire build follows (no secrets in
  code, SQL only with bind variables, never edit the vendored shared copy by hand, etc.) and the
  verified facts about each repository.
- `SOW_EDITOR_BUILD_PROMPTS_v1.md` / `SOW_EDITOR_RUN_GUIDE_v1.md` (repository root) — the full 75-step
  checkpoint plan this build is following.
- `ENV_REMINDER.md` (repository root) — the note-to-self about the Snowflake schema and user
  restriction, to be checked when real `.env` files are finally added.
