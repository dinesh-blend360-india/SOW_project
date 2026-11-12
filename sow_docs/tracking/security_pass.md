# Security pass (CP-69 and CP-70), 2026-11-13

Scope: local checks on the code and the test setup. Not a penetration test (that belongs to the Deployment block).
Rule from the plan: any HIGH finding blocks progress until it is fixed or accepted by the owner.

## Findings
| # | Severity | Finding | Evidence | Status / what is needed |
|---|---|---|---|---|
| F1 | HIGH (blocks progress) | The FastAPI Snowflake connection uses `insecure_mode: True`. This turns off certificate checks Snowflake normally does (for example, OCSP revocation checks). | `blend_ode_be/src/common/database.py:111` | NOT changed (per CP-69 task 5). The Node service does not copy the setting (checked: no `insecure` in sow_module/collab). **Owner decision needed:** fix in database.py, or accept the risk in writing. |
| F2 | MEDIUM (verify) | The Node service's Snowflake identity is not confirmed as least-privilege. `.env.example` leaves `SNOWFLAKE_USER` and `SNOWFLAKE_ROLE` blank. The November probe ran as the pricing app's dev user and role. That probe showed the `SOWS` table denied, but it did not test every application table. | `sow_module/collab/.env.example`; snowflake_findings.md | Needs a Snowflake check by the owner: grant the Node role only the `SOW_COLLAB_*` tables, and confirm it cannot read application tables. Not verified from this session. |
| F3 | MEDIUM | No rate limit on WebSocket messages. A flood of 2000 small messages is accepted (the server stays up, and the test records it). | server.test.ts 'a flood of small messages' | Proposal: a per-connection message budget, closing the socket on abuse. Not built; needs your OK on the limit values. |
| F4 | LOW | Empty `class` and `style` attributes can remain in exported HTML. They carry no value. | fuzz test (control handling) | Accepted. |
| F5 | INFO | Frames above 1 MB are closed with 1009; malformed frames get protocol_error and 1002. | server.test.ts 'WebSocket abuse' | Working as intended. |

## Checks run (all pass)
- Tickets: expiry, replay, tampering, wrong SOW, wrong user, cross-SOW reuse, wrong audience, wrong secret (ticket.test.ts, 10 tests).
- Internal endpoints: missing and wrong service key (Python 401 tests; Node flush and prepare/replace 401 tests).
- SQL: every Node statement uses `?` binds; the only interpolated names are table names built from configuration (`this.table()`). New Python SQL uses `%(name)s` binds (guest_invitation_is_live). No string-built SQL found in the new code.
- Sentinel secret: a ticket secret does not appear in logs or in FastAPI error text (sentinel-secret.test.ts, 2 tests).
- Content fuzzing on the server sanitizer: 1500 seeded random documents, 0 violations; control cases prove the checker can fail (test_sow_module_security_fuzz.py, 5 tests).
- WebSocket abuse: oversized frame, malformed frame, flood (test_server.test.ts, 3 tests).
- Role regression at REST: live endpoints refuse users without the feature; guest session refuses a missing token; internal endpoints refuse a wrong key (test_sow_module_endpoint_roles.py, 4 tests).
- Socket role matrix: viewer, commenter, editor, admin for edit, comment and checkpoint (server.test.ts, 12 tests).

## Frontend bundle (sentinel check)
See the result in the CP-69 log entry. The check looks for server-side secret values in the production bundle. Only `REACT_APP_*` variables should be inlined.

## Accepted risks (owner to confirm)
- F4 (empty leftovers).
- Pending decisions: F1, F2, F3.
