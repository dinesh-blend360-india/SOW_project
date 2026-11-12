# Phase G report: comments, suggestions, guests (CP-54 to CP-61)

Date: 2026-11-12. Status: built and tested in code. Browser checks and live Snowflake checks are still pending.

## What is in place
- Design decision D-B1 (option A): threads and suggestions live in the shared document and are saved at checkpoint into the native columns.
- Mentions: typeahead from the participants list, stored by email, delivered through the native notification code (idempotent, email only when enabled).
- Comments: anchored to blocks with a quoted range; hover highlight; reply; resolve and reopen; edit and delete own; admin redaction; orphan badge.
- Audit: native event names; payloads hold ids only.
- Suggestions: create (commenter and above), accept and reject (editor and admin); accept is one undoable edit; sign-off refused while any is pending.
- Guests: invitation and code page at /sow-live-guest; commenter tickets only; invitation re-checked on every session check.
- Revocation: the existing revoke path notifies the Owner, which disconnects the guest (tested). Editors are not affected.
- Finalized SOWs: text read-only for everyone; commenters keep comment and resolve; viewers are disconnected.

## Role matrix (CP-61)
- Capability table: viewer, commenter, guest, editor, admin against 10 actions (view, comment, reply, mention, suggest, edit, insert content, checkpoint, rename, finalize). Checked in tests/test_sow_module_role_matrix.py.
- Socket layer: edit, comment and checkpoint for viewer, commenter, editor, admin (12 tests in server.test.ts).
- Component layer: CommentsPanel (viewer, commenter, author, admin states), LexicalDraftEditor, guest page (5 tests).
- Not covered by automated tests: rename and finalize at the socket and REST layers (the server endpoints themselves); insert content at the socket layer.

## Test results (2026-11-12)
- Frontend sow_module: 347 passing.
- Node service: 235 passing, 7 skipped (need live Snowflake).
- Backend: 635 passing, 9 pre-existing failures (duration and template tests).

## Open items
- Redacted text still exists in older snapshots (purge built, off by default, not applied; see pending.txt A1).
- Edit and delete of comments are not audited (no native event names).
- Guest revocation is instant on the Owner side only when the Owner is reachable; the 5-second poll is the fallback.
- Browser checks for every flow (TODO_FOR_YOU).
- Antd components do not run under this jsdom setup; some component tests use plain stand-ins.
