# Testing Pending Guide

This file tracks every piece of work in the SOW Editor Module that **cannot be verified yet** because
it needs something only the user can provide — live credentials, real data, two actual browsers/
processes, or a product decision — plus exactly what's needed to unblock it and the command to run
once it's unblocked. It is append-only, like the checkpoint tracking files in
`sow_docs/tracking/`. When an item is finally run for real, move it from "Pending" to
"Resolved" with the date and the result — don't delete the history.

Read this alongside `sow_docs/narrative/README.md` (the narrative) and
`sow_docs/tracking/prompt_log.md` (the full checkpoint-by-checkpoint log). This file
exists specifically so "what still needs a live test" is visible in one place without reading every
tracking file.

---

## How an item gets here

A checkpoint's code and tests can be fully written and typecheck/lint/unit-test clean, and still not
be *proven* — proof sometimes requires something this non-interactive, credential-less development
environment cannot supply. When that happens, the code is written in full, the result is marked
**PENDING LIVE RUN** (never a fabricated number or a guessed pass/fail), and an entry is added here.

---

## Pending

### CP-08 — Snowflake connection and capability probes
- **What's built:** `sow_module/collab/src/spikes/{snowflakeClient,00-connect,01-capabilities}.ts`,
  fully coded, typechecked, and linted clean. `npm run spike:connect` and `npm run spike:capabilities`.
- **Resolved 2026-11-06:** both the `BLEND_INTERNAL_IT_APPS`/`NEXDRAFT` `USAGE` grants landed.
  `spike:connect` succeeds; `spike:capabilities` ran live: regular `CREATE TABLE`, `BINARY` storage,
  and `TIMESTAMP_TZ` all work for `PRICING_APP_DEV_ROLE`; the real `SOWS` table is correctly denied
  (the intended outcome); `CREATE HYBRID TABLE` is denied (a separate grant, not requested — see the
  tier decision below). `NEXDRAFT` was confirmed clean (0 tables) both before and after the run.
- **Still open, informational only:** whether to request `CREATE HYBRID TABLE` later. Not needed —
  the tier decision below was made without it.
- **Full detail:** `sow_docs/tracking/snowflake_findings.md`.

### Tier decision (CP-08/09) — RESOLVED by the user, 2026-11-06
**Standard tables (Tier 2).** `CREATE HYBRID TABLE` is denied for `PRICING_APP_DEV_ROLE`; rather than
request that grant, the user chose to proceed with standard tables, given CP-15's broadcast spike
showed nothing pathologically slow. **CP-09's durable-ack latency gate was never measured** — this is
a decision made without that number, recorded plainly as such in `migrations/011`'s header comment,
`spike_durable_log.md`, and `phaseB_report.md`. CP-09, CP-10, and CP-11 remain available to run later
(the grants now in place would let them actually execute) purely for extra confidence; they are no
longer blocking anything.

### CP-09 — Batched durable-log latency spike (no longer architecture-gating; optional follow-up)
- **What's built:** `sow_module/collab/src/spikes/02-durable-log.ts` — a 500ms-batch flush queue
  against a `SOW_SPIKE_UPDATE` table, plus single/100-row/1000-row insert timing, a point read, a
  conditional update, and a 5 MB chunked-`BINARY` write. Typechecked and linted clean. Not yet run.
- **Status:** the tier decision this was meant to inform has already been made (above). Running this
  would add confidence, not change the decision.
- **Full detail:** `sow_docs/tracking/spike_durable_log.md`.

### CP-10 — Lease and fencing-token spike (optional follow-up)
- **What's built:** `sow_module/collab/src/spikes/03-lease.ts` — conditional lease acquire, a batched
  renew statement, a fenced write, a two-process race (1 document, then 500), a "zombie holder"
  simulation, and a kill/failover timing measurement. Typechecked and linted clean. Not yet run.
- **Needed if run:** two separate running processes racing for the same document, which a single
  assistant session cannot itself orchestrate as two independent OS processes.
- **Full detail:** `sow_docs/tracking/spike_lease.md`.

### CP-11 — Crash durability and client-held-edits spike (optional follow-up)
- **What's built:** `sow_module/collab/src/spikes/04-durability.ts` — the CP-09 flusher plus a
  simulated client using real Yjs updates, repeated owner kills, and a check that every
  durably-acknowledged update survives while every unacknowledged update is held and merges exactly
  once on reconnect. Typechecked and linted clean. Not yet run.
- **Full detail:** `sow_docs/tracking/spike_durability.md`.

### CP-15 — Persistence-integrated and real-failover portions of the scale spike (optional follow-up)
- **What's built and RUN (2026-11-06):** the 100-client WebSocket broadcast measurement
  (`spike:scale`) and the proxy-hop overhead measurement (`spike:proxy-hop`) — both local, no
  Snowflake dependency, both with real measured numbers in `spike_scale.md`.
- **What's still not run:** persistence overhead under the same load (integrating the CP-09 flusher),
  and real failover timing using the actual CP-10 lease instead of a hardcoded routing table. No
  longer blocking — see the tier decision above.
- **Full detail:** `sow_docs/tracking/spike_scale.md`, `phaseB_report.md`.

### CP-25 — Two-browser live-editing proof (first real UI milestone)
- **Not started yet** (this entry is a placeholder added in advance, since it's a well-known upcoming
  blocker). Requires two actual browser windows/sessions, a running Node collaboration service, and a
  running FastAPI instance with the session endpoint implemented. Cannot be verified by an automated
  test alone — the CP-25 prompt itself calls for "a manual two-browser test note in the log."
- **Needed to unblock:** CP-08 through CP-24 complete and running locally; a person (or a scripted
  two-client harness) to actually open two sessions.

### CP-38 / CP-39 — Real screen integration and two-user manual test
- **Not started yet** (placeholder). CP-39 explicitly requires "a two-user manual test script with the
  flag on, written into `phaseD_report.md`" — this is a product/QA step, not something achievable by
  one non-interactive session.

### General: real saved draft HTML samples
- **What's affected:** `html-contract.json`'s "unverified" list (table inline styles, `colgroup`, cell
  `text-align`) and CP-12's 23-fixture set (`blend_ode/src/sow_module/shared/fixtures/html/`).
- **Needed to unblock:** at least 20 sanitized real values from `SOW_DRAFT_VERSIONS.CONTENT`, per
  CP-12's own instructions. The user confirmed (2026-11-01) that none are available yet, so CP-12
  ran against **synthetic** fixtures instead (all 23 pass, zero defects — see
  `spike_html_fidelity.md`), explicitly marked `synthetic-` and not a substitute for the real thing.
  Revisit once real samples exist.

### CP-37 — `@lexical/headless`'s default DOM (happy-dom) silently drops tables
- **Found during CP-12** (not CP-37 itself; recorded here in advance since CP-37 is where it will
  actually matter). `@lexical/headless`'s `withDOM()` convenience helper uses `happy-dom`; every
  table-containing fixture round-tripped through it lost its table entirely, with no error. Swapping
  to a directly-constructed `jsdom.JSDOM` fixed it with no other code change — confirmed the gap is
  in `@lexical/headless`'s bundled DOM, not in this module's `allowlist.ts`/`htmlFidelity.ts`.
- **Needed to unblock:** nothing further right now — `scripts/check-html-fidelity-node.ts` already
  uses `jsdom` directly and passes 23/23. This item exists so CP-37 (the Owner's server-side HTML
  export, described in the build plan as using "headless Lexical") deliberately chooses `jsdom` (or
  confirms another safe option) instead of accidentally reaching for `withDOM()`'s default.
- **Full detail:** `sow_docs/tracking/spike_html_fidelity.md`, "A fourth finding."

---

## Resolved

*(nothing yet — entries move here, with date and result, once a pending item is actually run)*
