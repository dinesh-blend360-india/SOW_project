# Frontend (`blend_ode`) — SOW Editor Module

This document covers everything done on the frontend side of the SOW Editor Module: the audit of the
existing CKEditor-based editor, and the new shared contract code and its tests built on top of that
audit. It is append-only — new checkpoints add new sections below, they don't rewrite old ones.

Repository: `blend_ode` (Create React App 5, React 18, TypeScript 4.9, antd 5, `ckeditor5` 48.5.1).

---

## Part 1 — Audit of the existing editor (CP-01, CP-02)

Before writing any new code, the entire existing CKEditor-based draft editor was read and documented,
read-only, so later work replaces it deliberately rather than guessing at what it does. Nothing in this
section was changed; it is a record of what exists today. The full line-numbered detail is in
`sow_docs/tracking/audit_frontend.md`; this is the readable summary.

### `DraftEditor.tsx` — the editor component itself (754 lines)

**What it is:** a React component wrapping CKEditor 5, with 19 props covering content, save/autosave
hooks, watermark/banner display flags, read-only mode, download and copy-link callbacks, and a dirty
(unsaved-changes) indicator.

**The CKEditor configuration** — this is the part the new Lexical editor has to reproduce:

- **18 plugins**: the basics (undo/redo, paragraphs, headings h1–h6, bold, italic, underline,
  strikethrough), a highlight tool, font size and font family pickers, text alignment, bulleted and
  numbered lists, a full table toolkit (insert, column/row operations, merge cells, table and
  cell-level styling dialogs, column drag-resize), and a find-and-replace tool.
- **7 highlight options**, each applying a `<mark class="...">` around text: four colored "marker"
  styles, two colored "pen" (text color) styles, and one special one — `sowPlaceholder`
  (`class="sow-studio-placeholder-marker"`) — used specifically to flag a field still waiting for a
  value, like `[Client Name]`.
- **7 font sizes** (10, 12, 14, "default", 18, 24, 32) and **9 font families** (Arial, Courier New,
  Georgia, Lucida Sans, Tahoma, Times New Roman, Trebuchet MS, Verdana, plus "default"). Font family
  additionally allows *any* pasted value through unchanged — not just the 9 listed ones.
- Explicitly **not** present: comments, a trial-only "collaboration" plugin, cloud services, a table of
  contents, a balloon toolbar — confirmed by an existing test that checks these are absent.

**SOW-specific behavior layered on top of CKEditor**, none of which is a CKEditor feature itself:

- **Placeholder auto-removal.** When someone finishes filling in a bracketed placeholder (so no `[` or
  `]` remains in it), the highlight marking it as "pending" is automatically stripped away, both at the
  level of the saved HTML and at the level of CKEditor's internal document model (so the cursor doesn't
  jump).
- **A "DRAFT" watermark** — a 3×3 grid of large, faint, rotated "DRAFT" text — shown only while the SOW
  still has sign-offs pending.
- **A provisional-pricing warning banner**, shown only when the SOW's pricing isn't yet locked in the
  pricing tool.
- **Read-only mode** for viewing a historical version or a finalized SOW, with a label explaining why.
- Save button (disabled unless there are unsaved changes), a download dropdown (DOCX or PDF), a
  copy-link button, and an "⚠ Unsaved changes" indicator.
- **Add-in navigation**: after an "add-in" (a pre-written optional clause) is inserted into the draft,
  the editor scrolls to and briefly highlights it, matched by a `data-add-in-id` attribute in the HTML.

### The CKEditor-to-Lexical parity table

The audit produced a 41-row table (full detail in `audit_frontend.md` section 5) mapping every single
one of the above CKEditor features to its planned Lexical equivalent, with a decision column. 37 of 41
rows are marked **"built"** — meaning Lexical (or the Lexical ecosystem, `@lexical/table`,
`@lexical/list`, `@lexical/rich-text`, `@lexical/history`) already has a direct equivalent, or the
behavior is pure React/CSS with no editor dependency at all (the watermark, banner, save button, and
similar are just React state — they port over unchanged regardless of which editor is underneath).

**4 rows are marked "TBD" — genuine open product decisions, not technical unknowns:**

1. **Table properties dialog** (border, background, overall dimensions) — CKEditor has a built-in
   modal for this; Lexical has nothing equivalent out of the box. Build a custom dialog, accept the
   feature loss, or offer a simpler context menu?
2. **Table cell properties dialog** (per-cell border, background, padding) — same question, per cell.
3. **Table column drag-resize** — `@lexical/table` has an *experimental* resize feature; is it stable
   enough to ship, or does this need a custom-built drag handle?
4. **Arbitrary pasted fonts** — CKEditor today preserves *any* font family pasted in from an external
   document, not just the 9 listed options. Should Lexical do the same, or normalize everything down
   to the approved list?

### `DraftEditorScreen.tsx` — the save/autosave flow and the swap point (1284 lines)

This is the component that *wraps* `DraftEditor`, and it is where the real complexity around saving
and concurrency lives — none of which has anything to do with CKEditor specifically, which matters
because this component is **not** being replaced; only the `<DraftEditor>` it renders will be swapped
for the new Lexical editor.

**The save and autosave flow, in plain terms:**

- Every keystroke increments an internal "edit generation" counter and marks the draft as dirty.
- A 15-second timer checks: is there unsaved content, and is nothing already saving? If so, save.
- Saving sends the content along with an `expected_revision` value — a hash of the draft as it was
  last fetched. The backend rejects the save if someone else has saved in the meantime (see
  `backend.md`'s explanation of `save_sow_draft`), which is the system's only current protection
  against two people silently overwriting each other.
- Because the generation counter is captured *before* the save request goes out, if new edits happen
  while a save is in flight, the system can tell afterward that "this save is now stale" and leaves the
  dirty indicator on, rather than silently discarding the newer edits.

**The swap point** — the single place in the entire frontend where the editor component is
instantiated — is `DraftEditorScreen.tsx`, lines 1097–1159. Everything around it (the save logic,
autosave timer, historical-version read-only handling, download, copy-link) stays exactly as it is;
only the `<DraftEditor ...>` JSX tag itself will eventually become `<LexicalDraftEditor ...>` with a
matching set of props. `SowStudioPage.tsx` renders `DraftEditorScreen` with no feature flag at all
today — the eventual Lexical/CKEditor switch will need one (planned: `FE_SOW_LEXICAL_EDITOR`, added at
CP-19, not yet created).

**Six service files read or write the draft's HTML directly**, independent of which editor produced
it, using the browser's `DOMParser`: inserting and renumbering "add-in" clauses, detecting whether a
given add-in is already present, syncing signature-block Name/Title fields from intake data, detecting
which "open items" (missing-field warnings) are now resolved, and two navigation helpers. Three of
these touch CKEditor-specific DOM selectors (`.ck-editor__editable`) that will need a Lexical
equivalent or a shared CSS class added to both editors' editable regions.

**8 risks for the transition were identified** (full list in `audit_frontend.md` section 15),
including: the optimistic-revision-lock approach has no retry/merge strategy today (a failed save just
shows an error and asks the user to reload); CKEditor is known to strip `data-*` attributes during
serialization, which the add-in-presence check already has a text-matching fallback for; and the
backend's DOCX/PDF renderer will need to handle whatever HTML Lexical produces, which may differ subtly
from CKEditor's (different whitespace, different attribute ordering).

---

## Part 2 — Shared contract code (CP-05, CP-06)

With the audit complete, the next step was writing down — as actual, type-checked code, not just
prose — everything that the browser, the future Node.js collaboration service, and FastAPI all need to
agree on. This code lives at `blend_ode/src/sow_module/shared/` and is treated as the **one true copy**
(see "Why shared code lives here" below).

### Why this code lives in the frontend repository

Create React App cannot import any file from outside its own `src/` folder — this is a hard limitation
of how CRA's build tooling works, not a style choice. At the same time, the Node.js collaboration
service (which lives in the *backend* repository, `blend_ode_be`) needs to use the exact same type
definitions, or the two sides will eventually disagree about what a message on the wire means. The
resolution: the frontend repo owns the source of truth, and a script
(`blend_ode_be/sow_module/sync_shared.sh`) copies it into the backend repo, writing a SHA-256 hash file
in both places. A test in each repo fails if its copy's hash no longer matches what was recorded — this
catches anyone who edits the vendored copy by hand instead of editing the source and re-running the
sync script.

**Hard rule, enforced by code review discipline rather than tooling:** never edit
`blend_ode_be/sow_module/collab/src/shared/*` directly. Always edit
`blend_ode/src/sow_module/shared/*`, then run `bash blend_ode_be/sow_module/sync_shared.sh`.

### What's in `shared/`

All of this is pure TypeScript with **no import from React, antd, Node.js, or the DOM** — confirmed
both by inspection and by the fact that it type-checks under a `tsconfig` that doesn't even include the
`dom` or `node` type libraries, so any accidental use of a browser or Node global would fail to
compile.

- **`roles.ts`** — the four roles (`viewer`, `commenter`, `editor`, `sow_admin`), the nine capabilities
  a role can grant, and the `roleToCapabilities` function (see below).
- **`protocol.ts`** — every message that can travel over the WebSocket connection between the browser
  and the Node collaboration service, and the meaning of each possible connection-closing code.
- **`errors.ts`** — a shared vocabulary of error codes (e.g. `stale_revision`, `access_removed`,
  `ticket_expired`) and which HTTP status each one maps to, so the browser, Node service, and FastAPI
  all describe the same failure the same way.
- **`bridge.ts`** — the request/response shapes for the handful of internal HTTP calls between FastAPI
  and the Node service, and the structure of the short-lived "ticket" used to authenticate a WebSocket
  connection (see `backend.md` for the full security reasoning).
- **`html-contract.json`** plus **`html-contract.ts`** — a machine-readable description of exactly
  which HTML tags, attributes, CSS classes, and inline style properties the new editor is allowed to
  produce, derived directly from the CKEditor audit above and from what the backend's HTML-dependent
  code (export, quality checks, sign-off) actually requires (see `backend.md` Part 2 for the backend
  side of this). The `.ts` file is a thin typed loader around the JSON — the JSON itself is the
  artifact that both a future frontend serializer and a future backend validator will check output
  against, so they can never silently drift apart.
- **`role-table.json`** — the role-to-capabilities table (below), exported as plain data so a later
  backend test can confirm it hasn't drifted from the frontend's version.

### The role table — `roleToCapabilities` (CP-06)

CP-05 deliberately left this function as a stub that returns **no capabilities for any role** — a
"fails closed" placeholder, so that nothing downstream could accidentally rely on a role having more
access than intended before the real table existed. CP-06 replaced the stub with the real
implementation:

| Role | Capabilities | Who this is, today |
|---|---|---|
| `viewer` | read | Anyone who can see the deal but shouldn't edit; also anyone viewing a locked/historical SOW |
| `commenter` | read, comment, suggest | External/guest reviewers |
| `editor` | read, comment, suggest, edit, resolve_thread, checkpoint | Blend staff working on the deal |
| `sow_admin` | everything above, plus rename, manage_access, finalize | Not yet defined — see open questions below |

Two rules make this safe to reason about:

1. **Each role is a strict superset of the one before it** — a `sow_admin` can do everything an
   `editor` can, an `editor` can do everything a `commenter` can, and so on. This is tested directly.
2. **A "features" override can only ever remove a capability, never add one.** The intended use is a
   future kill-switch (e.g. temporarily disabling `finalize` for everyone during an incident) — it
   must never be usable to *grant* extra access by mistake. This asymmetry is enforced in the function
   itself, not just by convention, and is covered by a dedicated test.

An unknown/misspelled role returns an empty capability list rather than throwing or defaulting to
anything — "fail closed," same philosophy as the original stub.

**Why a `rename` capability was added, specifically:** the CP-06 prompt required "editor can edit but
not rename." In the *current* CKEditor-based system, any editor can rename a SOW as a side effect of a
normal save (the `sow_name` field is just part of the same save payload as the content). This module
introduces a stricter rule — only `sow_admin` can rename — which is a genuine behavior change from
today's system, flagged explicitly in `contracts.md` and the prompt log as something worth confirming
rather than something discovered unchanged from the existing code.

### Tests (`src/sow_module/testing/`)

- **`roles.test.ts`** — 9 tests: each role's capabilities are checked explicitly (a viewer cannot
  comment or edit; a commenter can comment but not edit; an editor can edit but not finalize or
  rename; a `sow_admin` can do everything); the superset relationship between roles is checked
  generically; the "features only remove, never add" rule is checked both for turning something off
  and for attempting (and failing) to turn something on that a role doesn't already have; and an
  unknown role is confirmed to return nothing.
- **`jsonSnapshot.ts`** — a small helper that compares a piece of data against a committed JSON file
  (`role-table.json`), and can be told to regenerate that file via an environment variable
  (`UPDATE_SOW_MODULE_SNAPSHOTS=1`) when the table is deliberately changed. This exists so the
  role table that the backend will eventually compare itself against is always generated *from the
  same code* that defines the role table, rather than being hand-copied and liable to drift.
- **`sharedDrift.test.ts`** (from CP-00/06/07 tooling) — confirms Create React App's Jest runner
  actually discovers and runs tests placed anywhere under `src/sow_module/`, and separately confirms
  the `shared/` folder's hash still matches what `sync_shared.sh` last recorded.

### Confirmed, not assumed

Two load-bearing facts were explicitly checked rather than assumed, because the whole shared-code
approach depends on them being true:

- `shared/` type-checks cleanly with `npx tsc --noEmit -p src/sow_module/tsconfig.shared.json` under
  TypeScript's `strict` mode.
- Create React App's built-in Jest test runner does pick up test files placed under
  `src/sow_module/testing/`, with no configuration changes needed.

### What is still open on the frontend side

- The 4 "TBD" items from the parity table (table/cell property dialogs, column resize stability,
  arbitrary pasted fonts) are product decisions, not yet made.
- Who exactly should hold the `sow_admin` role is unresolved (see `backend.md` and the root
  `README.md` for the full explanation — it's a backend/product question, not a frontend one).
- The stricter `rename` rule (only `sow_admin`, not every `editor`) needs explicit confirmation, since
  it changes current behavior.
- The existing frontend test suite has two failing suites
  (`DraftEditorScreen.integration.test.tsx`, `SowStudioPage.test.tsx`) that appear to be pre-existing
  (timeouts, not related to anything this module touches — no file outside `src/sow_module/` imports
  from it) but this has not yet been confirmed against a clean checkout of the branch.
- No Lexical code has been written yet. Everything so far is shared contract types, tests for those
  types, and the audit that justifies them. The actual editor component comes later in the checkpoint
  sequence.
