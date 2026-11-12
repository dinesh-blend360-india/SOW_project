# Design: comments, suggestions, guests and mentions (CP-54)

Status: **proposal, waiting for D-B1.** CP-55 (mentions) does not start until the owner answers D-B1.

## 1. What the native editor stores today (read from the code, 2026-11-12)

Tables (all `SOW_NATIVE_*`, NEXDRAFT schema):

| Table | What it holds | Written by |
|---|---|---|
| `SOW_NATIVE_DOCUMENTS` | One row per deal: `DOCUMENT_JSON`, `HTML_CONTENT`, `COMMENT_THREADS`, `SUGGESTIONS`, `SOURCE_VERSION`, `REVISION` | `save_native_editor_state`, `checkpoint_native_editor`, `reset_native_editor_to_latest` (`native_collaboration.py`) |
| `SOW_NATIVE_GUEST_INVITATIONS` | Guest invitations: email, name, expiry, revocation | `create_guest_invitation`, `revoke_guest_invitation` |
| `SOW_NATIVE_GUEST_SESSIONS` | Sessions for a verified guest | `verify_guest_invitation`, `_create_guest_session` |
| `SOW_NATIVE_MENTION_DELIVERIES` | One row per mention notification, with an idempotency key and error text | `_send_new_mention_notifications` |
| `SOW_NATIVE_AUDIT_EVENTS` | Review and state changes (`_audit`, `_audit_review_changes`) | every write path |
| `SOW_NATIVE_EDITOR` | Workspace state (`_select_workspace`) | save and checkpoint |

Comment threads and suggestions are JSON arrays in `COMMENT_THREADS` and `SUGGESTIONS`. A mention is an entry in a comment's `mentions` array, and `_comment_mentions` reads it.

## 2. Who can do what

- Every native route calls `assert_deal_access` (same rule as the draft).
- Editors and admins save state and checkpoint. Guests save only their own review (`save_guest_review`), validated by `_validate_guest_review`: they can add their own comments and suggestions, and cannot edit or remove others' entries.
- Mention notifications are sent only to people in `_participants` (`allowed_mentions`). Guests get a signed link (`_signed_guest_mention_token`).
- Email is sent only when `ENABLE_SOW_EMAIL_SENDING` is true.

## 3. What it lacks for the live editor

- **Real time:** the native editor saves whole JSON documents. Two people editing comments at once overwrite each other (last save wins, guarded only by the revision).
- **Block-stable anchors:** comments point at text ranges. After an edit the range can move or disappear. The live editor has block IDs (`data-block-id`, CP-33), which can anchor a comment to a block.

## 4. Proposal (default for D-B1)

1. **Threads and suggestions live in the Yjs document**, so they sync live like text. Anchors are block IDs plus character offsets within the block.
2. **Persisted at checkpoint** into the existing `COMMENT_THREADS` and `SUGGESTIONS` columns, in the same JSON shape the native editor already reads. A FastAPI internal endpoint does the write, and it reuses the existing audit and mention-notification code by import, not by copy.
3. **Guest invitations, mention deliveries and audit events reuse the existing tables and endpoints unchanged.**
4. **Native editor behaviour does not change, and its data is not migrated.** The live editor and the native editor read the same columns, so a SOW can be viewed in either.

## 5. Participants list (CP-54 task 3, built)

`GET /sows/{sow_id}/collab/participants` (in `src/sow_module/router.py`, same gate as the collab session) returns `{participants: [...]}` from `_participants()`: the current user, the deal owner, solution and delivery leads, the Book 2026 contact, collaborators and pricing contributors (type `blend`, role `edit`), and live guests (type `guest`, role `comment_suggest`). Revoked and expired guests are excluded in the SQL. Tests: `tests/test_sow_module_participants.py`.

## 6. Question for you (D-B1)

> Should comment threads and suggestions in the live editor be stored in the Yjs document (live sync, anchored to blocks) and written to the existing `COMMENT_THREADS` and `SUGGESTIONS` columns at checkpoint, as proposed above?
>
> Options:
> - **A (recommended):** as proposed. Live sync, existing columns, no data migration, native editor unchanged.
> - **B:** keep comments and suggestions in the native editor only, and build no live comments in the live editor for now.
> - **C:** store them in a new table for the live editor, and sync between the two later. More work, and two sources of truth.

Until D-B1 is answered, CP-55 (mentions) and CP-56 (comment threads) stay on hold.
