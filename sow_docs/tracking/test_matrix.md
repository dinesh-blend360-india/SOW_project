# Test matrix (CP-67), 2026-11-13

Method: each cell names the test file that proves it. "Pre-existing" means the test failed before this work too.

## 1. Roles x actions (three layers)
| Action | Viewer | Commenter / guest | Editor | SOW admin | Test |
|---|---|---|---|---|---|
| View | yes | yes | yes | yes | role matrix (socket), role_table parity |
| Comment, reply | no | yes | yes | yes | server.test.ts 'role matrix (socket)'; CommentsPanel tests |
| Mention | no | yes | yes | yes | live_comments tests; CommentsPanel mention test |
| Suggest | no | yes | yes | yes | suggestions.test.ts; live-comments suggestion rules |
| Accept/reject suggestion | no | no | yes | yes | CommentsPanel suggestion tests; validation tests |
| Edit text | no | no (disconnected) | yes | yes | role matrix (socket); validation tests |
| Insert content | no | no | yes | yes | editorApi tests (read-only refusal); insert-audit |
| Checkpoint | no | no (access_denied) | yes | yes | role matrix (socket) |
| Rename | no | no | no | yes | test_sow_module_role_matrix.py (capability table) |
| Finalize | no | no | no | yes | capability table; finalize flush tests |
| Redact comment | no | no | no | yes | CommentsPanel admin tests; live-comments redaction tests |
Layers: component (CommentsPanel, LexicalDraftEditor, SowLiveGuestReview), REST (role_table, guest-session, participants, finalize routes), socket (server.test.ts). Rename and finalize are covered at the capability and REST layers only (no socket layer).

## 2. Document lifecycle
| Step | Test |
|---|---|
| Draft created and loaded | seed.test.ts; collab_internal tests |
| Live edit | server.test.ts (edit, two editors) |
| Checkpoint (in place) | test_sow_module_role_matrix.py (in-place); checkpoint.test.ts |
| Finalize (flush first, lock) | test_sow_module_finalize_flush.py; server.test.ts 'finalize integration' |
| Sign-off and publication | test_sow_signoffs.py, test_sow_publication.py (existing, passing); lexical publication test |
| New draft (regenerate) | test_sow_module_replace_hooks.py; room-replace tests |

## 3. Intake sources
| Source | Test |
|---|---|
| Paste | paste.test.ts; intake.test.ts; intake.fuzz.test.ts |
| LLM insert | editorApi tests (audit, idempotency, removals) |
| Add-in | add-in catalog passes intake (removalMessage.test.ts) |
| Legacy open | legacy-baseline.test.ts; ConversionReportNotice tests |
| Fuzzing | intake.fuzz.test.ts (2000 per format) |

## 4. Failure cases
| Case | Test |
|---|---|
| Snowflake slow or down | outage-guard.test.ts; checkpoint unavailable tests |
| Lease lost (fencing) | checkpoint-coordinator.test.ts (read-only room); persist-queue.test.ts |
| FastAPI down | checkpoint.test.ts (unavailable); flush refusal (finalize flush tests) |
| Owner down at finalize | test_sow_module_finalize_flush.py (503) |
| Drain / shutdown | server.test.ts shutdown tests (existing) |
| Revocation | server.test.ts 'guest revocation'; access-recheck tests |

## 5. Full suites (2026-11-13)
- Frontend, whole project: 664 passed, 162 failed, 3 skipped. The 162 failures are in 8 suites that fail identically on the baseline commit before this work (App.test, CopyPricingTemplateButton, PricingTemplateDetails, DealDeskScreen, DraftEditorScreen integration, RiskFlagsTab, SignOffScreen integration, SowStudioPage). Pre-existing.
- Backend, whole project: 675 passed, 9 failed. The 9 are the known pre-existing duration and template tests.
- Node service: 247 passed, 7 skipped (need live Snowflake).

## Accepted exceptions
- Rename and finalize have no socket-level test (REST and capability tests cover them).
- Colour contrast is not checked automatically.
- Live Snowflake behaviour (lease, fencing, audit rows, guest expiry) is not covered by automated tests.
