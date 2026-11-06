# Backend (`blend_ode_be`) — SOW Editor Module

This document covers everything done on the backend side of the SOW Editor Module: the audit of the
existing SOW Studio backend, the contracts and architecture decisions written on top of that audit, and
the Node.js collaboration service and backend tooling built so far. Append-only — new checkpoints add
new sections, old ones stay as a historical record.

Repository: `blend_ode_be` (FastAPI, Python 3.11, Snowflake via SQLAlchemy with key-pair auth).

---

## Part 1 — Audit of the existing backend (CP-03, CP-04)

Read-only. No application file was changed in either of these checkpoints. Full line-numbered detail
is in `sow_docs/tracking/audit_backend.md`; this is the readable narrative.

### Access control

Every route into SOW Studio passes through a chain of checks: a JWT is decoded to get the user's roles
and feature flags (`authorize_token`); a feature-flag dependency confirms the user has `FE_SOW_STUDIO`,
`FE_SOW_STUDIO_PILOT`, or the admin override `FE_ADMIN` (`authorize_sow_access`); and a per-deal check
(`assert_deal_access`) confirms this specific user is actually associated with this specific deal (or
is an admin). A separate, stricter flag (`FE_SOW_NATIVE_EDITOR`) gates the existing (unrelated)
evaluation-only native editor feature. Guests (external reviewers in that same native editor feature)
authenticate differently — by a hashed session token tied to an invitation, not a JWT. There is also a
fully public, token-scoped path used for the SOW approval-preview and share-link pages, which has no
login at all.

### `save_sow_draft` — the single door into the database

This is the most important function in the entire backend audit, because the new module's entire
design rests on *not* bypassing it. Its job, in order:

1. Confirm deal access.
2. **Lock the row.** Snowflake does not enforce primary-key or unique constraints the way most
   databases do — they exist only as metadata. So the codebase protects against two simultaneous writes
   the hard way: a `MERGE INTO` statement that acts as a real serialization point.
3. Fetch the current version of the draft and check: has it been finalized? (If so, reject — a
   finalized SOW cannot be silently edited.)
4. **Check the revision.** Every save must include an `expected_revision` value, which the frontend got
   from the last time it fetched or saved the draft. This value is a SHA-256 hash of the draft's
   version number, content, comments, open items, add-ins, creation timestamp, and name — i.e., a
   fingerprint of the entire row. If the hash the frontend sent doesn't match the hash of what's
   actually in the database right now, the save is rejected (HTTP 409) — someone else saved first. This
   is the system's only current protection against lost updates.
5. Merge in any resolved "open items" (missing-field warnings that get automatically cleared once the
   corresponding field is filled in correctly — see "Open-item detection" below).
6. Invalidate any in-progress review/evaluation for this deal, since the content just changed.
7. **Write.** For a normal save or autosave, this *updates the existing row in place* — it does not
   create a new version. A new version row is only created for specific flows (applying a field from
   intake, or continuing from a historical version) — not for ordinary editing.

**Why this matters for the new module:** the Node collaboration service's eventual "checkpoint" (saving
accumulated live edits back to the permanent record) must go through this exact function, with the
exact same revision-checking discipline, rather than writing to the database directly. This means the
new module inherits all of the above guarantees for free, and never has to reimplement — or risk
reimplementing incorrectly — the finalized-draft guard, the row lock, or the audit trail.

### The rest of the save-related system

- **Draft retrieval** (`get_latest_sow_draft`, `get_sow_draft_version`) shapes a database row into the
  response the frontend expects, computing the same revision hash so the frontend always has a current
  fingerprint to send back on its next save.
- **AI draft generation** creates an entirely new version from scratch using an AI model; it is only
  reachable from the initial "submit intake" step, not from ordinary editing, so the new module doesn't
  touch it.
- **"Continue from a historical version"** copies an old version's content into a new current version,
  with its own revision checks on both the source version and the current one.
- **Suggested versions** — an "accept/reject this AI-suggested fix" flow — creates an immutable
  proposed version alongside the real one; accepting it promotes it to current.
- **The finalize pipeline** (`publish_final_sow`): locks the SOW's row, checks it is actually eligible
  to finalize (approved status, no unresolved required approvals, a current quality review exists),
  generates a PDF from the HTML, uploads it to S3 and attaches it to the deal in HubSpot, and only then
  flips the SOW's status to "Finalized." Finalizing is gated by exactly the same permission check as
  editing, plus the status check — there is no existing separate "who can finalize" concept in the
  code today (this fact drives one of the open questions below).
- **The existing native editor feature** — a *separate*, evaluation-only collaboration feature with its
  own set of database tables (`SOW_NATIVE_*`) for document state, guest invitations, guest sessions, and
  an audit trail — reaches the real `SOW_DRAFT_VERSIONS` table only through one narrow function
  (`checkpoint_native_editor`), which itself just calls `save_sow_draft`. This is architecturally the
  closest existing precedent for what the new module is building, which is why reusing some of its
  guest/invitation machinery is one of the live open questions below.
- **Database connection**: Snowflake, via SQLAlchemy, using key-pair authentication, with one notable
  existing fact that was flagged (not changed, per standing rule): the connection sets
  `insecure_mode: True`, which disables TLS certificate verification.
- **Ten existing SQL migrations** were inventoried; migrations 011–013 were found to have been
  deliberately removed from the repository (per an existing staging README) even though the database
  tables for one of them (the native editor's tables) still exist live in Snowflake — meaning there is
  live database schema with no corresponding file in version control. **The next migration number for
  this module is 011.**

### Every piece of code that reads or writes the draft's HTML (CP-04)

This is the audit that makes the HTML contract (Part 2, below) possible. 19 separate backend functions
were traced, each with its own unwritten assumptions about the shape of the HTML it's given. A few
examples, because they illustrate exactly the kind of silent breakage this module has to avoid:

- **Signature detection** (`_signatory_fields_are_complete`) looks for a `<table>` row containing
  exactly two `<td>` cells, each with "Signature:", "Name:", and "Title:" text lines inside it, and
  only considers a value "filled in" if it contains no literal `[` or `]` characters and no `__`
  underscores. If the new editor wrapped signature tables in an extra layer of `<div>` or produced three
  cells instead of two, this check would silently stop recognizing completed signatures — not crash,
  just quietly never clear that open item.
- **Billing-schedule detection** (`completed_billing_schedule`) checks a table's header cells against
  exact, case-insensitive text: `"Month"`/`"Monthly fee"` for a retainer engagement, or
  `"Payment milestone"`/`"Amount"`/`"Invoice trigger"` for a fixed-fee one. Any different header
  wording breaks this silently.
- **Placeholder detection** (`_bracketed_placeholder_tokens`) scans the *raw* HTML string character by
  character looking for literal `[` and `]` characters — if the new editor ever HTML-encoded a bracket
  as an entity (`&#91;`), this detection would simply never see it.
- **The DOCX/PDF export parser** (`_HtmlToBlocksParser`) understands a specific, limited set of tags
  (`p`, `h1`–`h6`, lists, basic tables, a handful of inline formatting tags) and silently drops colspan
  and rowspan, class, and style attributes — meaning any table with merged cells will export as a flat,
  un-merged grid today, regardless of which editor produced the HTML.
- **The approval-preview sanitizer** (`sanitize_rich_text`) is far stricter still: it has no `<mark>`
  tag at all and strips every single `class` and `style` attribute from every element. This means, as
  things stand today, highlight colors and placeholders are already invisible on the public
  approval-preview page — a pre-existing fact, not something this module causes, but one that matters
  because of a decision explained in Part 2.
- Of five AI agent code paths examined, only one — `suggested_fix_agent` — ever writes HTML back into
  the saved draft (as a brand-new proposed version); the other four either produce structured JSON (not
  HTML) or only *read* the HTML, stripped down to plain text, to feed a model.

The full 19-row table, the LLM-path table, and the test-file inventory are in `audit_backend.md`
sections 14–17.

---

## Part 2 — Contracts and decisions (CP-05)

With both audits complete, this checkpoint turned the findings into two things: a prose contract
document (`contracts.md`) and a set of formally numbered Architecture Decision Records
(`decisions.md`), both in `sow_docs/tracking/`. The actual type definitions these
documents describe live in the frontend repo's `shared/` folder (see `frontend.md`) and are vendored
into this repo — this file explains the backend-facing *reasoning*, not the TypeScript itself.

### The endpoints

Seven endpoints make up the whole bridge between the three parties. All seven are currently
**proposed, not implemented** — CP-05 is contracts only.

| # | Endpoint | Who calls it | What it does |
|---|---|---|---|
| 1 | `POST /sows/{sow_id}/collab/session` | Browser (logged-in Blend user) | Checks every existing access gate, then issues a short-lived connection ticket |
| 2 | `POST /sow-native/guest/{sow_id}/collab/session` | Browser (guest) | Same, for an invited external reviewer |
| 3 | WebSocket `wss://.../ws/sow/{sow_id}` | Browser | The actual live-editing connection, hosted by the Node service |
| 4 | `POST /internal/sow-collab/resolve-access` | Node service → FastAPI | "Is this person still allowed in, and with what role?" |
| 5 | `POST /internal/sow-collab/document-state` | Node service → FastAPI | "Give me the current saved HTML, version, and revision" |
| 6 | `POST /internal/sow-collab/checkpoint` | Node service → FastAPI | "Here is accumulated live-edit HTML; please save it" — this is the only path that calls `save_sow_draft` |
| 7 | `POST /internal/access-changed` | FastAPI → Node service | "Someone's access just changed — re-check them immediately" (e.g. a guest was revoked) |

Endpoints 4–7 are **internal only** — they must never be reachable from the public internet, only from
one service to the other. Whether the existing deployment setup can actually enforce that is still an
open question (below).

### The ticket — how a WebSocket connection proves who it is

A normal login (JWT) is not used directly to authenticate the WebSocket connection, deliberately.
Instead, FastAPI issues a separate, very short-lived, single-use "ticket" when a session is first
requested:

- Lasts **30 seconds** — long enough to open the WebSocket and send it, too short to be useful if
  intercepted later.
- Is bound to one specific SOW, one specific person, and one specific role — it cannot be replayed
  against a different document or reused after a role change.
- Carries a unique, one-time-use ID that the Node service remembers for at least a minute, so even a
  ticket used twice within its 30-second window would be rejected the second time.
- **Is signed with a secret that is deliberately different from the main login secret (`JWT_SECRET`).**
  This is the single most important security property of the design: a leaked ticket is useless as a
  login credential, and a leaked login token is useless as a connection ticket, because they are
  different kinds of object, cryptographically, not just by convention.

### The WebSocket lifecycle

In short: connect → send the ticket within 5 seconds or get disconnected → the Node service re-checks
access with FastAPI → if everything's fine, the service confirms readiness and starts syncing the
live document → every 60 seconds access is silently re-checked again in the background → if access is
revoked, the server closes the connection with a specific code (`4403`); if the document gets finalized
while someone's connected, it closes with a different code (`4410`) and the client falls back to the
existing read-only HTML view; if a *different* server instance actually owns this document right now
(see the lease/fencing explanation below), the connection is refused with a hint about which instance
does. The full numbered sequence, with exact message names, is in `contracts.md` section 5.

### Role mapping (confirmed by the user, with a planned extension)

| Role | Who | Status |
|---|---|---|
| `commenter` | External/guest reviewers — read, comment, and suggest, but cannot resolve a comment thread or edit | Confirmed |
| `editor` | Blend staff on the deal | Confirmed |
| `sow_admin` | Whoever can finalize today | **Still open** — see below |

**Planned, not yet designed:** external (client-side, non-guest) access, authenticated by a
secret-key-style credential rather than a normal login, with access only granted after an explicit
approval request. This is intended to reuse the existing native editor's guest-invitation pattern
rather than invent a new one, but the actual mechanism has not been built or even fully specified yet.

**Why `sow_admin` is unresolved rather than assumed:** the audit found that finalizing a SOW today uses
*exactly* the same permission gate as editing one, plus a status check — there is no existing "some
editors can finalize, others can't" distinction anywhere in the code to read off. Making up an answer
here would be inventing a product rule, not reporting a fact, so it was left as an explicit open
question rather than guessed.

### The HTML contract

A single JSON file (`html-contract.json`, vendored from the frontend repo) is the one place that lists
exactly which tags, attributes, CSS classes, and inline styles the new editor is allowed to output —
built directly from the CP-01/CP-02 audit of real CKEditor output and the CP-03/CP-04 audit of what the
backend's parsers actually require. A handful of details (certain table inline styles, column widths,
cell text-alignment) are explicitly marked "unverified" in the file, because no real, saved SOW draft
was available to check them against — only hand-written test fixtures. This should be revisited against
genuine production data before the contract is treated as final.

**Decision: the module gets its own HTML allowlist, and never uses `sanitize_rich_text`.** The existing
sanitizer was built for a different job (locking down the public approval-preview page) and is far too
strict for this one — it has no `<mark>` tag at all and strips every `class` and `style` attribute,
which would silently delete every highlight color, every pending-placeholder marker, and all table
styling. One consequence worth knowing: the approval-preview page, as it exists today, *already*
doesn't show highlights or placeholders, for the same reason — this module doesn't change that, and
fixing it is explicitly out of this module's scope.

### The 12 Architecture Decision Records (ADRs)

Each one is written down in full, with its status, in `decisions.md`. In brief:

1. **HTML is the system of record** — the Yjs/Lexical document is disposable working state; the HTML
   is permanent.
2. **Checkpoints always go through `save_sow_draft`** — never a direct database write.
3. **The Node service only ever writes to its own `SOW_COLLAB_*` tables** — never the existing SOW
   tables, never the native editor's tables, directly.
4. **(D-B1) Reuse the native editor's guest/mention/audit machinery** — the user's preferred default,
   but this directly conflicts with an existing standing rule (never touch `SOW_NATIVE_*` tables) and
   with the fact that those tables' schema isn't even in version control. Flagged as needing the rule
   itself to be explicitly amended if this path is taken.
5. **(D-B2) The Node service is a genuinely separate deployable**, not a Python-based relay — rejected
   alternative, because validating a Lexical-shaped Yjs document correctly requires Lexical's own
   JavaScript packages, which don't exist for Python.
6. **Live edits are held in memory and flushed to storage in batches**, with a "your edits are now
   durably saved" acknowledgment sent back only after a batch actually commits — because Snowflake
   performs poorly with many tiny individual inserts.
7. **One document has exactly one "owner" server instance, held via a database lease with an
   incrementing fencing token** — using the same `MERGE`-based locking trick the existing codebase
   already relies on, since Snowflake doesn't enforce real uniqueness constraints on its own.
8. **(D-B4) Shared TypeScript code is owned by the frontend repo and copied into this one** by a
   script, with a hash check in both places — see `frontend.md` for the full reasoning.
9. **(D-B3) The module owns its own HTML allowlist**, never the existing `sanitize_rich_text`.
10. **Tracking docs and the Node service live in this (backend) repository** — because the Node
    service will deploy and run alongside FastAPI and share its CI pipeline.
11. **Permission changes propagate within 60 seconds by default**, with an immediate push (the
    `access-changed` hook) for urgent cases like a guest being revoked.
12. **(Added after CP-05) All of this module's own database tables live in
    `BLEND_INTERNAL_IT_APPS.NEXDRAFT`, using only the Snowflake user `PRICING_APP_DEV_USER`** — a
    restriction stated directly by the user, not inferred. See "Storage and environment" below.

### Storage and environment (added after CP-05)

The user has specified a hard constraint: this module's own tables (leases, live-edit logs, snapshots,
audit records — everything in the `DataStore` interface described in Part 3) may live **only** in
Snowflake schema `BLEND_INTERNAL_IT_APPS.NEXDRAFT`, connected to **only** as the user
`PRICING_APP_DEV_USER`. Nothing else is permitted. This is recorded as ADR-012, and a standalone
reminder file was created at the repository root, `ENV_REMINDER.md`, specifically because no real
`.env` files exist yet — the user intends to add them later, and that file lists exactly what to check
at that point: that the chosen Snowflake user actually has the necessary grants in both the new schema
and wherever the *existing* SOW tables live (since a checkpoint still needs to read and write those,
through `save_sow_draft`), and that the existing SOW tables are reachable from whatever schema ends up
being the active session schema. **No migration will be run against the real Snowflake database until
the user explicitly approves it, per the project's standing rule.**

---

## Part 3 — The Node.js collaboration service and backend tooling (CP-06, CP-07)

### Why a separate Node.js service, specifically

This was debated and written down as ADR-005: the alternative — keeping everything in Python, with the
existing FastAPI backend acting as the WebSocket relay too — was rejected, because correctly validating
and serializing a Lexical-shaped Yjs document requires the actual Lexical JavaScript packages. There is
no equivalent Python library that understands Lexical's internal document model well enough to do that
validation correctly. The practical cost of this decision is a second thing to build, test, and deploy;
that cost was accepted deliberately rather than discovered as a surprise later.

### What exists so far (`blend_ode_be/sow_module/collab/`)

- **A small, deliberate dependency list**: Express (a minimal HTTP surface, currently just a health
  check), `ws` (WebSockets), `yjs` and `y-protocols` (the CRDT sync protocol itself), Zod (input
  validation), and Pino (structured logging) — plus TypeScript, Vitest, and ESLint as dev tooling.
  Notably **no Snowflake driver yet** — that's intentionally deferred to the next checkpoint, so the
  service's core logic can be built and fully tested against a fake first.
- **A strict TypeScript configuration**, matching the same "no accidental `any`" discipline as the
  shared contract code.
- **The `DataStore` interface** (`src/data/store.ts`) — a single, explicit description of every way
  this service will ever need to read or write persistent data, deliberately kept separate from any
  particular database technology:
  - `DocumentRecord` — one row per SOW the service knows about, tracking which saved version/revision
    it was last seeded from.
  - `UpdateRow` — one individual live edit, as raw Yjs update bytes plus who made it and when.
  - `SnapshotChunk` — a periodic compacted snapshot of a document's live state, so the service doesn't
    have to replay every single edit ever made from the beginning every time it restarts.
  - `Lease` — who currently "owns" a given document (see the fencing-token explanation above), with an
    expiry time and an incrementing `fencingToken`.
  - `Revocation` — a record that a specific person's access to a specific document was pulled.
  - `AuditEvent` — a generic append-only audit log entry.
  - A dedicated `StaleFencingError`, thrown whenever something tries to write using a fencing token
    that is no longer the current one for that document — this is the actual mechanism (not just a
    design description) that stops an old, demoted server instance from corrupting data after a newer
    instance has taken over.
- **`InMemoryDataStore`** — a complete, working implementation of the entire `DataStore` interface
  backed by plain in-memory JavaScript objects. This is not a mock with stubbed-out methods; it is
  fully functional for a single-process run, including correctly enforcing "only one lease holder at a
  time" and correctly rejecting stale fencing tokens. This lets every piece of real collaboration logic
  that gets built next be written and tested completely before any actual Snowflake connection exists —
  the real Snowflake-backed implementation, when it's built, only has to satisfy the same interface and
  pass the same tests.
- **A minimal `server.ts`** — currently just a health-check HTTP endpoint — exists purely so the `npm
  run dev` script has something to actually run; it is not yet the real collaboration server.
- **Environment variable names only** (`collab/.env.example` and `src/sow_module/ENV.md` in the Python
  side) — Snowflake account/role/key-path, the ticket-signing secret, two separate service-to-service
  keys (one per direction, matching `contracts.md`), the FastAPI base URL, and the batch-flush interval
  (default 250ms, matching ADR-006). **No actual secret values exist anywhere in the repository** — by
  design, and by standing project rule.

### Backend (Python) side of the tooling

- `blend_ode_be/tests/test_sow_module_smoke.py` — confirms the new `src/sow_module` Python package can
  actually be imported under the existing `pytest` setup, with nothing more required.
- `blend_ode_be/tests/test_sow_module_shared_drift.py` — the backend-side half of the "did someone
  hand-edit the vendored shared copy" check described in `frontend.md`.
- `blend_ode_be/sow_module/sync_shared.sh` — the vendoring script itself: copies the frontend's
  `shared/` folder into the backend's `collab/src/shared/`, also copies the HTML contract and the role
  table into `blend_ode_be/tests/fixtures/sow_module_html/` so a future Python test can compare the
  backend's own HTML validator against the same source of truth, then writes a SHA-256 hash file in
  both locations.
- `blend_ode_be/sow_module/run_checks.sh` — runs every check across both repositories and the Node
  service in one command, and prints a single pass/fail summary table. As of CP-07 this is seven steps:
  the shared-copy sync check, the frontend's Jest tests for the module, a strict TypeScript check on the
  shared code, a TypeScript check on the Node service, an ESLint pass on the Node service, the Node
  service's Vitest test suite, and the backend's Python smoke/drift tests. All seven currently pass.

### What is still open on the backend side

- **Who is `sow_admin`** — unresolved, as explained above; this blocks nothing built so far, but will
  need an answer before any actual permission enforcement is written.
- **The native-editor table reuse question (D-B1)** — needs an explicit decision, because the default
  the user prefers currently conflicts with an existing standing rule and depends on database schema
  that isn't in version control.
- **Whether `/internal/*` endpoints can actually be locked down at the network/ingress level** — not
  yet confirmed against the real deployment setup.
- **Confirming `PRICING_APP_DEV_USER`'s actual Snowflake grants**, and confirming the existing SOW
  tables are reachable from a session scoped to `BLEND_INTERNAL_IT_APPS.NEXDRAFT` — neither has been
  checked yet, because no real environment file exists to check against.
- **No Snowflake-backed `DataStore` implementation exists yet** — only the in-memory fake. That is the
  explicitly planned next step ("the spikes begin by connecting to Snowflake from Node").
- **No actual collaboration protocol logic exists yet** — no WebSocket server, no ticket
  issuing/verification code, no lease acquisition loop, no checkpoint logic. CP-05 through CP-07 built
  the contracts, the role table, and the persistence *interface* that all of that will be built on top
  of — the collaboration logic itself is still ahead.
