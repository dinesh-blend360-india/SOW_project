# Decisions (ADRs)

All statuses are **Proposed** until the user confirms. Later prompts follow these; if one changes, edit it here first and note it in `prompt_log.md`.

## ADR-001 HTML is the system of record
- Status: Proposed
- Decision: The saved SOW stays HTML in `SOW_DRAFT_VERSIONS.CONTENT`. Yjs state is working state that can be thrown away and rebuilt from the HTML.
- Why: Export (DOCX/PDF), quality checks, sign-off, finalize and Deal Desk all read HTML (audit_backend.md section 14).
- Consequence: Every checkpoint serializes to HTML that satisfies `html-contract.json`.

## ADR-002 Checkpoints go through `save_sow_draft`
- Status: Proposed
- Decision: The only write into `SOW_DRAFT_VERSIONS` is the existing `save_sow_draft` (crud.py:1875), called by the internal checkpoint endpoint with `expected_revision` and `create_version=False`.
- Why: It owns the revision lock, the row lock, review invalidation, change log and open-item reconciliation.
- Consequence: Revision checks keep working; the Owner must track `baseRevision` and handle 409.

## ADR-003 The Node service writes only `SOW_COLLAB_*`
- Status: Proposed
- Decision: The Owner touches only tables it owns (`SOW_COLLAB_*`). It never writes `SOWS`, `SOW_DRAFT_VERSIONS`, `HUBSPOT_DEALS` or `SOW_NATIVE_*`. Anything else goes through FastAPI internal endpoints.
- Why: One database, shared; keeps blast radius small and the audit trail in one place.

## ADR-004 D-B1: relationship to the native editor evaluation
- Status: Proposed (default requested by the user)
- Decision: Reuse the native editor's guest, mention and audit tables and flows; supersede its frontend later. The collab module adds only `SOW_COLLAB_*`.
- Conflicts to resolve before accepting:
  1. `CLAUDE.md` and the do-not-break list (audit_backend.md item 7) say the module must never read or write `SOW_NATIVE_*`. Reuse needs that rule amended to: the Node service never touches them; FastAPI may call the existing guest and mention functions (`authorize_guest_session`, invitations, mention delivery) and may write audit events through them.
  2. The DDL for `SOW_NATIVE_*` is **not in the repo** (migration 013 was removed). Reuse means depending on a schema that is only documented by the code that queries it.
- Alternative: copy the concepts into `SOW_COLLAB_*` tables (more tables, no dependency on undocumented DDL, duplicate guest flow).

## ADR-005 D-B2: Node collaboration service as a separate deployable
- Status: Proposed
- Decision: Node + Express + `ws` + Yjs in `sow_module/collab/`, deployed apart from FastAPI.
- Alternative considered and rejected: a Python-only relay. Validating and serializing a Lexical-shaped Yjs document needs the Lexical packages, which exist only in JavaScript; a Python relay could not validate what it relays.
- Consequence: A second deployable (container, health check, env, CI job) and the service-key bridge in `contracts.md`.

## ADR-006 Memory-first, batched durable log
- Status: Proposed
- Decision: The Owner holds the live Yjs document in memory, appends updates to an in-memory buffer and flushes in batches to `SOW_COLLAB_UPDATES`; `DURABLE {upToSeq}` is sent only after a batch commits. A checkpoint (HTML through FastAPI) runs on a timer, on explicit save, when the last client leaves and on shutdown.
- Why: Snowflake is poor at many tiny inserts; batching is mandatory. Batch size and interval (proposal: about 250 ms or 100 updates) are tuned at the persistence prompt.
- Consequence: Up to one batch interval of edits is at risk on a hard crash; clients keep unacknowledged updates until `DURABLE` covers them.

## ADR-007 Lease with fencing in Snowflake
- Status: Proposed
- Decision: One Owner per document, held by a lease row in `SOW_COLLAB_LEASES` taken with `MERGE` (Snowflake does not enforce unique constraints, so serialization comes from `MERGE`, the same technique as `_lock_sow_row_for_draft_write`). Each acquisition increments a fencing token. Every log append and every checkpoint carries the token; FastAPI and the persistence layer reject a stale token.
- Consequence: A paused or partitioned old owner cannot overwrite a newer one.

## ADR-008 D-B4: shared code is owned by the frontend repo and vendored into the backend repo
- Status: Proposed (already implemented in CP-06 tooling)
- Decision: `blend_ode/src/sow_module/shared/` is the source of truth (CRA only compiles `src/`). `sync_shared.sh` copies it to `blend_ode_be/sow_module/collab/src/shared/` with a SHA-256 manifest; drift tests fail CI if either side changes by hand.
- Alternative: a private npm package or a third repo (rejected for now: extra release process for a small amount of code).

## ADR-009 D-B3: the module owns its HTML allowlist
- Status: Proposed
- Decision: Checkpoints are validated by a new `html_allowlist.py`, driven by the vendored `html-contract.json`. The existing `sanitize_rich_text` is never used on checkpoints and never edited.
- Why: `sanitize_rich_text` drops every `class` and `style`, has no `mark`, `colgroup` or `figure`, and would destroy placeholders, highlights, font sizes and table layout (audit_backend.md section 14, row 1).
- Consequence: The approval preview (`api/public.py`, which still uses `sanitize_rich_text`) will not show highlights or placeholders (audit question Q2). That preview is out of scope here.

## ADR-010 Tracking files and the Node service live in the backend repo
- Status: Proposed (already the layout)
- Decision: `sow_module/_tracking/`, `sow_module/collab/` and the Python side (`src/sow_module/`) live in `blend_ode_be`; the React component and `shared/` live in `blend_ode`.
- Why: The Node service deploys alongside FastAPI and shares its CI; `SOW_project/` is not a repo.

## ADR-011 Permission changes
- Status: Proposed
- Decision: The Owner re-checks every connection with `resolve-access` every 60 seconds, plus an explicit `access-changed` hook that FastAPI calls on guest revoke (and on role change, deal-access change, finalize) for immediate effect.
- Consequence: Worst-case delay for ordinary role changes is 60 seconds; guest revoke is immediate if the hook is wired into the revoke function.

## ADR-012 Storage location for the module's tables
- Status: Proposed (location stated by the user 2026-10-30)
- Decision: All `SOW_COLLAB_*` tables live in schema `BLEND_INTERNAL_IT_APPS.NEXDRAFT`. SQL in the module uses fully qualified names. The migration (CP-17) is not applied without explicit approval.
- Facts from the code: the existing app connects with `SF_DATABASE` / `SF_SCHEMA` from env (`.env.example` shows `RESOURCE_REQUEST_TOOL` / `PRICING_TOOL_SANDBOX_HYBRID`) and every existing query uses unqualified names (`SOWS`, `SOW_DRAFT_VERSIONS`) that resolve through that session schema.
- Consequences: (1) module SQL must never rely on the session schema; (2) the Snowflake user and role must hold grants on both locations; (3) a checkpoint touches two schemas (`save_sow_draft` writes `SOWS` / `SOW_DRAFT_VERSIONS` in the existing schema; the lease and log are in NEXDRAFT), so they are separate transactions and the fencing token, not a shared transaction, is the safety mechanism; (4) the Node service needs its own connection settings (separate env names, set later by the user).
- User restriction (2026-10-30): only `BLEND_INTERNAL_IT_APPS.NEXDRAFT` and user `PRICING_APP_DEV_USER` may be used. If the app's session schema becomes NEXDRAFT, consequence (3) collapses to one schema (one transaction is then possible), but the existing SOW tables must exist in NEXDRAFT; see `SOW_project/ENV_REMINDER.md`.
- Open: `SF_USER` value (`PRICING_APP_DEV_USER`) is unconfirmed; no real `.env` exists on this machine to check.

---

## Questions awaiting the user

| # | Question | Default if you just say "default" |
|---|---|---|
| Q1 | Role mapping: guests = commenters with suggest, Blend participants = editors, finalizers = sow_admin. Correct? | As proposed in `contracts.md` section 7 |
| Q2 | Who may finalize? Today the gate is the same as editing (plus status Approved). Should `sow_admin` be FE_ADMIN only, a deal-level role, or everyone who can edit today? | FE_ADMIN plus the deal owner role (needs you to name it) |
| Q3 | D-B1: reuse the native editor's guest, mention and audit tables, knowing their DDL is not in the repo and CLAUDE.md currently forbids touching them? | Reuse through FastAPI only; amend CLAUDE.md |
| Q4 | D-B3: own `html_allowlist.py` driven by `html-contract.json`, never `sanitize_rich_text`? | Yes |
| Q5 | D-B4: frontend repo owns `shared/`, vendored into the backend repo with a hash check? | Yes |

Also open (not blocking CP-05): G1 guest gating by feature flag; Q-C1 `current_user` for checkpoints; Q-C2 who wins when CKEditor and Lexical both save one SOW; Q-N1 whether `/internal/*` can be blocked at ingress; TBD-1 to TBD-4 from audit_frontend.md; real saved draft samples to verify the unverified items in `html-contract.json`.

## User answers

2026-10-30:
- Q1 (role mapping): confirmed as proposed (guests = commenters with suggest, Blend participants = editors, finalizers = sow_admin), with this addition: for now only Blend people are added as participants. External (client) access is planned for later: an external gets a secret key used like a password, and their access is granted through an approval request. Not designed yet; it will reuse the guest path (ADR-004) and the commenter role, and needs its own ADR before it is built.
- Real saved draft samples: none available. The unverified items in `html-contract.json` stay unverified; verify against the first real drafts produced in testing.
- Q2 (who may finalize), Q3 (D-B1), Q4 (D-B3), Q5 (D-B4), Q7 (ingress blocking): not answered yet; ADRs stay Proposed.
