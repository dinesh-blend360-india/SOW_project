# Task Board

Statuses: `NOT_STARTED`, `IN_PROGRESS`, `BLOCKED`, `IN_REVIEW`, `COMPLETED`. Only tasks grounded in
the module's own build plan, a documented open question, or a verified gap found during this
inventory are listed — nothing here is invented scope. Use `templates/TASK_ENTRY_TEMPLATE.md` for
new entries.

**Rewritten 2026-11-11, end of session** (not just appended to) — the previous version had ~10
stacked "Update" sections with a stale top (e.g. CP-21 still shown `IN_PROGRESS` after CP-48 was
already done). Full history is in `SESSION_LOG.md` / `sow_docs/tracking/prompt_log.md`.

---

## IN_PROGRESS

*(none — nothing was left mid-edit at session end)*

---

## BLOCKED

### SOW-CP39-MANUAL: Run the two-user manual test
- **Priority**: High (the module's own acceptance gate for everything built in Phase D onward)
- **Affected repository**: both
- **Dependencies**: backend + collab service running together; two real SSO-authenticated accounts
  with `FE_SOW_LEXICAL_EDITOR`/`FE_ADMIN` and access to the same deal
- **Relevant files**: `sow_docs/tracking/phaseD_report.md` (the script and a results table to fill in)
- **Description**: Never run. Needs a human (or two) with real accounts — no agent session so far
  could complete this alone. Not a code task.
- **Status**: BLOCKED on scheduling/access, not on anything technical.

---

## NOT_STARTED — deferred by explicit choice (re-evaluate before starting)

### SOW-CP44: Socket routing across multiple Owner instances
- **Priority**: Low for now
- **Affected repository**: blend_ode_be (`sow_module/collab`)
- **Description**: Only matters once more than one Owner instance is actually deployed behind a load
  balancer. Deferred 2026-11-11 by explicit user/agent budget agreement, not a blocker.

### SOW-CP46: Scale test harness
### SOW-CP47: Scale measurements (10–100 simulated editors) and approved gates
- **Priority**: Low for now (CP-47 is the single most compute/token-expensive item in the whole plan)
- **Affected repository**: blend_ode_be (`sow_module/collab`)
- **Description**: Deferred 2026-11-11 by explicit user/agent budget agreement — do this close to
  actual production rollout, not speculatively.

---

## NOT_STARTED — next in the build plan (not yet begun)

### SOW-CP49-ENDPOINT: Server-callable insert (`POST /sows/{sow_id}/collab/insert`), CP-49 task 2
- **Priority**: Low until a backend job with no user online needs it
- **Affected repository**: both
- **Dependencies**: a new durable pending-insert table (migration needs owner approval, CP-16/17) and a
  retention decision (CP-66). Browser-side CP-49 is done (see COMPLETED).
- **Description**: Deferred. The editor applies inserts because HTML-to-Lexical conversion exists only in
  the browser; a service call with nobody online needs somewhere durable to wait.

### SOW-CP52: Regeneration, intake sync, reset to latest in a live room
### SOW-CP53: Intake messages, fuzzing, Phase F close report
- **Priority**: Medium (CP-51 specifically is correctness-on-real-data, worth prioritizing over 49/50)
- **Affected repository**: both (mostly blend_ode_be for 51/52)
- **Relevant files**: `sow_docs/build_plan/SOW_EDITOR_BUILD_PROMPTS_v1.md` (CP-50 to CP-53 sections)
- **Description**: Not started — see the build plan for each checkpoint's own task list. CP-52 also owns
  routing the whole-draft replacement flows (continue, intake sync, suggested-fix accept, restore) into the
  live editor.

### Phase G (CP-54–61): Mentions, comments, suggestions, guests
- **Priority**: Medium. **BLOCKED on D-B1** (CP-54): owner must answer `sow_docs/tracking/design_collab_features.md` before CP-55 starts
- **Description**: Not started. CP-59 (suggestions / track-changes-style editing) is the hardest
  single feature left in the whole plan — budget it accordingly.

### Phase H (CP-62–66): Versions, finalize lock, export parity, audit
### Phase I (CP-67–71): Full hardening — test matrices, load tests, security, accessibility
### Phase J (CP-72–74): Cutover tooling, rollback, CKEditor removal
- **Description**: Not started. CP-68 (load tests) carries the same cost profile as CP-47 — defer
  similarly until near rollout.

---

## NOT_STARTED — standing gaps and decisions

### SOW-ROLE: Resolve decisions.md Q2 (who is sow_admin / finalize-capable)
- **Priority**: Medium
- **Affected repository**: blend_ode_be
- **Dependencies**: a product decision from the project owner
- **Relevant files**: `blend_ode_be/src/sow_module/collab_session.py` (`resolve_role`),
  `sow_docs/tracking/decisions.md` (Q2)
- **Description**: Code currently maps `FE_ADMIN` → `sow_admin`, everyone else who passes
  `assert_deal_access` → `editor`. A documented stand-in, not a resolution.
- **Acceptance criteria**: `decisions.md` Q2 has a recorded answer; `resolve_role` and its tests match.

### SOW-FENCE: Independent fencing-token re-verification at checkpoint time
- **Priority**: Low (documented, deliberate gap while the feature isn't live)
- **Affected repository**: blend_ode_be
- **Dependencies**: a second, narrowly-scoped Snowflake connection into `NEXDRAFT` from FastAPI
- **Relevant files**: `src/sow_module/collab_internal.py` (`checkpoint`)
- **Description**: See `KNOWN_ISSUES_AND_BLOCKERS.md` #7.

### SOW-RENEW-PRECISION: Per-document lease renewal result, not a batched count
- **Priority**: High
- **Affected repository**: blend_ode_be (`sow_module/collab`)
- **Dependencies**: live Snowflake access to verify the query change safely
- **Relevant files**: `data/snowflake-store.ts` (`renewLeases`), `owner/lease-manager.ts` (`renewAll`)
- **Description**: See `KNOWN_ISSUES_AND_BLOCKERS.md` #14. `renewAll()` can't tell "every held lease
  was actually renewed" apart from "the call succeeded but renewed none" — it optimistically refreshes
  both cases alike. CP-42's `onFenced` real-time backstop covers the worst consequence in practice;
  the underlying imprecision remains.
- **Acceptance criteria**: the store reports which specific `dealId`s were actually renewed;
  `renewAll()` only refreshes those and `forget()`s the rest.

### SOW-PERSIST-RACE: Fix PersistQueue's overlapping-flush data loss
- **Priority**: High
- **Affected repository**: blend_ode_be (`sow_module/collab`)
- **Relevant files**: `owner/persist-queue.ts`
- **Description**: See `KNOWN_ISSUES_AND_BLOCKERS.md` #15. Pre-existing (CP-22), surfaced by CP-42's
  review. No in-flight guard per document; a successful flush deletes the whole pending array, not
  just the rows it sent — an update enqueued mid-flush is silently lost.
- **Acceptance criteria**: in-flight guard per `dealId`; a flush removes only the rows it sent; a test
  proving an edit enqueued mid-flush survives to the next tick.

### SOW-FENCE-ATOMIC: Make the Snowflake fencing check atomic with the write
- **Priority**: Medium-High
- **Affected repository**: blend_ode_be (`sow_module/collab`)
- **Dependencies**: live Snowflake access to verify safely
- **Relevant files**: `data/snowflake-store.ts` (`appendUpdates`)
- **Description**: See `KNOWN_ISSUES_AND_BLOCKERS.md` #16. The fencing `SELECT` and the write
  `MERGE` are two separate statements, not one transaction.

### INFRA-SECRET-ROTATE, INFRA-ENV-EXAMPLE, INFRA-CI, INFRA-MIGRATIONS-CLARIFY, INFRA-DEPS-CLARIFY, INFRA-CRLF-HASH
- **Description**: Unrelated to the SOW module specifically but relevant to anyone working in these
  repos — see `KNOWN_ISSUES_AND_BLOCKERS.md` #1–6 and #13 for full detail on each. Not re-detailed
  here to avoid a second copy that can drift.

---

## IN_REVIEW

*(none)*

---

## COMPLETED

### SOW-CP52: Regeneration, intake sync, reset to latest in a live room — done 2026-11-12 (not browser-tested)
- **Affected repository**: both
- **Description**: Save-then-replace for regenerate, intake sync, continue and accept-suggestion; "Reload latest" with
  revision check and discard confirmation; saved-content hash so a reopened room is replaced or marked stale.
  See `prompt_log.md` CP-52. Two-browser checks still to be run by the owner.

### SOW-CP51: Existing drafts open in Lexical — done 2026-11-12
- **Affected repository**: both
- **Description**: Conversion report for SOW admins, no-op checkpoint check for unedited older drafts (stored
  HTML is never rewritten by opening), batch fidelity script, Open Items navigation in the live editor, and
  save parity (open items, add-ins present and SOW name stored by live-editor saves). See `prompt_log.md` CP-51.

### SOW-CP50: Streaming preview and smart paste — done 2026-11-12
- **Affected repository**: blend_ode
- **Description**: Streaming preview block (local ghost, one commit, cancel and error leave the document
  unchanged) and Markdown smart paste with undo-to-plain. No screen starts a stream yet (no streaming LLM
  endpoint exists). See `prompt_log.md` CP-50.

### SOW-CP49: Generated content insert (editor side) — done 2026-11-12
- **Affected repository**: both
- **Description**: `insertContent` and add-in insertion in Lexical mode audited (hash only, Option A),
  idempotent, editors and admins only. Owner handles `CONTENT_INSERTED`. See `prompt_log.md` CP-49.

### SOW-CP00–CP43, CP-45, CP-48: SOW collaboration module build-out
- **Affected repository**: both
- **Description**: All acceptance checks passed at the time each checkpoint was completed. See
  `CHANGELOG.md` for the itemized commit history and `CURRENT_PROJECT_STATE.md` for what's actually
  built and working today. Two carve-outs, not failures: the `SOW_COLLAB_*` Snowflake migrations
  (011/012) are written and tested but **not yet applied** (a separate, explicitly-gated action); the
  CP-39 two-user manual test has never been run (see `SOW-CP39-MANUAL` above).
- **Explicitly deferred, not part of "complete"**: CP-44, CP-46, CP-47 (see the deferred section above).

### DOCS-HANDOFF: Create and maintain this agent-handoff documentation system
- **Affected repository**: root
- **Description**: Created 2026-11-06, kept current every session since, rewritten (not just
  appended) 2026-11-11 for this handoff. See `SESSION_LOG.md` for the full session history.

### SOW-CP53: Intake messages, fuzzing and Phase F close — done 2026-11-12
- Removal wording shared by paste and LLM insert; legacy-open notice for everyone; fuzzing at 2000 documents per format; `phaseF_report.md`. Regression green. Real-draft fidelity run still pending (needs exports).

### SOW-CP54: Decision gate (D-B1) and participants — partly done 2026-11-12, BLOCKED on D-B1
- Design written (`design_collab_features.md`); participants endpoint built and tested. The owner must answer D-B1 before CP-55.
