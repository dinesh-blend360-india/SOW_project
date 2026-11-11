# Current Project State

**Last updated: 2026-11-11, end of session — handing off to another developer.** This file was
rewritten (not just appended to) at this point to consolidate ~10 stacked "Update" entries into one
current picture. Full session-by-session history is preserved in `SESSION_LOG.md` and
`sow_docs/tracking/prompt_log.md` (per-checkpoint detail) — nothing was lost, just de-duplicated here.

## Architecture status

- **Pricing Tool and SOW Studio** (the pre-existing, CKEditor-era application): built, in production
  use, unmodified by any of the work below.
- **SOW collaboration module** (Lexical + Yjs live-editing, replacing CKEditor for SOW authoring): in
  active development. **CP-00 through CP-43, CP-45, and CP-48 of the module's own build plan
  (`sow_docs/build_plan/SOW_EDITOR_BUILD_PROMPTS_v1.md`) are complete.** CP-44, CP-46, CP-47 were
  explicitly deferred (see "Deferred by choice" below, not blocked or failed). CP-49 onward (Phases
  F tail, G, H, I, J) are not started.
- The Lexical editor is reachable only behind a feature flag (`FE_SOW_LEXICAL_EDITOR`, or `FE_ADMIN`
  with `REACT_APP_SOW_LEXICAL_EDITOR=true` at build time). With the flag off, the screen is
  byte-for-byte unchanged from the CKEditor-era behavior.

## What's built and working (verified by automated tests, not yet by real users)

- Shared TypeScript contracts (roles/capabilities, protocol, HTML allowlist, bridge types,
  intake pipeline) in `blend_ode/src/sow_module/shared/`, vendored into `blend_ode_be` via
  `sync_shared.sh`, parity-tested against the Python mirror.
- Full editor: toolbar, formatting, lists (3 levels), tables (insert/merge/shading, no drag-resize),
  paste/drop through the one intake pipeline, placeholders, find/replace, stable block IDs, page
  chrome (watermark/banner/read-only view), action bar, a public editor API (`getHtml`/`setHtml`/
  `insertContent`/`scrollToBlock`), and an add-in adapter.
- Live collaboration: the Node "Document Owner" service (Express + `ws` + Yjs) hosts real WebSocket
  rooms; single-writer leases with fencing; chunked sync for large documents; presence/cursor
  awareness (throttled); automatic reconnect with backoff and a fresh session per attempt; offline
  edits via `y-indexeddb` resend exactly once on reconnect.
- Owner checkpoint: explicit Save and quiet-period saves go through the existing `save_sow_draft`;
  an outside change to the draft is refused (not silently overwritten) and reported to clients.
- **The Snowflake-unreachable rule and graceful shutdown (CP-42)**: a room this process loses its
  lease on goes permanently read-only rather than risk a second writer; `SIGTERM`/`SIGINT` drains
  (flush, checkpoint, release leases, evict clients) within a bounded time with a hard-exit backstop.
- **Snapshot scheduling and log compaction (CP-43)**: a document snapshots after N durable updates or
  an idle period, then the update log is pruned (with a safety tail) — this had been built at the
  storage layer in CP-23 but never actually triggered until CP-43.
- **Live access enforcement (CP-45)**: a connection re-checks its own access every 60s; an explicit
  `POST /internal/access-changed` hook (which FastAPI's `notify_access_changed` was already calling,
  into nothing, since CP-20) now closes/downgrades affected sockets within the same request; a
  5-second revocation-poll backstop; a SOW becoming Finalized mid-session closes editors with 4410.
- **The intake pipeline (CP-48)**: one allowlist, one intake module, used identically by paste and by
  add-in/programmatic insertion; fuzz-tested (fast-check) across html/markdown/text; the Python
  parity pytest (`test_sow_module_html_parity.py`) has passed since CP-13.

## Deferred by choice, not blocked (re-evaluate before continuing the build plan)

- **CP-44** (socket routing across multiple Owner instances) — only matters once more than one
  instance is actually deployed behind a load balancer; premature now.
- **CP-46/CP-47** (scale test harness / measurements at 10–100 simulated editors) — CP-47 specifically
  is the single most compute/token-expensive item in the entire build plan. Do this close to actual
  production rollout, not now.
- These were a deliberate, user-agreed budget decision this session, not a technical blocker.

## Not yet started

Everything from **CP-49 onward**: LLM/generated-content insert (CP-49, needs a product decision on
whether to store LLM prompts/responses in the audit log before starting), streaming preview/smart
paste (CP-50), legacy CKEditor drafts opening safely in Lexical (CP-51, was in progress when this
session ended — see `TASK_BOARD.md`), regeneration/reset in a live room (CP-52), intake hardening
close-out (CP-53), and all of Phases G (mentions/comments/suggestions/guests), H (versions/finalize/
export), I (hardening/load/security/accessibility), and J (cutover tooling). See `TASK_BOARD.md` and
the build plan itself for the full per-checkpoint detail — not duplicated here to avoid drift.

## Never verified with real users (important — don't assume this works just because tests pass)

- **The CP-39 two-user manual test has never been run.** It needs the backend and collab service
  running together and two real SSO-authenticated accounts with `FE_SOW_LEXICAL_EDITOR`/`FE_ADMIN`
  access to the same deal — a human-driven step no agent session so far could complete alone. The
  script is in `sow_docs/tracking/phaseD_report.md`, with a results table ready to fill in.
- Real two-browser cursor/presence behavior, real network reconnect behavior, and the live access
  re-check/finalize-lock paths have only been tested with fakes (automated WebSocket clients and
  mocked FastAPI responses), never a real browser against the real backend.

## Known limitations and open gaps

See `KNOWN_ISSUES_AND_BLOCKERS.md` for the full, itemized list (16 items as of this writing). The
ones most relevant to anyone picking this module back up:
- **#1**: real secrets are committed and tracked in `blend_ode/.env.development`/`.env.production` —
  predates this module, unrelated, but still live and unresolved.
- **#13**: this (and possibly other) Windows checkouts have `core.autocrlf=true`, which makes two
  hash-exact-match parity tests and `sync_shared.sh --check` fail on pure line-ending noise, not real
  drift (confirmed via `git diff --ignore-all-space`). Not fixed — needs the owner's sign-off on a
  `.gitattributes` entry or a git config change, since it touches shared repo config.
- **#14–16**: three correctness gaps found by a targeted review of CP-42 (imprecise per-document lease
  renewal, a pre-existing `PersistQueue` overlapping-flush race from CP-22, a non-atomic Snowflake
  fencing check). All three need live-Snowflake access to fix safely — not attempted without it.
- **#7**: FastAPI's checkpoint endpoint does not independently re-verify the fencing token (mitigated
  by CP-42's `onFenced` real-time backstop, but the underlying gap remains — see `SOW-FENCE` and
  `SOW-FENCE-ATOMIC` in `TASK_BOARD.md`).
- Open product decision, still unanswered: who exactly is `sow_admin` (finalize-capable)? Code
  currently stands in with `FE_ADMIN`-only (see `SOW-ROLE` in `TASK_BOARD.md`, `decisions.md` Q2).

## Important implementation details worth knowing before continuing

- The SOW module's Node service connects to Snowflake with **different credentials and a different
  schema** (`BLEND_INTERNAL_IT_APPS.NEXDRAFT`) than the main FastAPI app.
- `checkpoint()`'s HTML sanitization always goes through `sow_module.html_allowlist
  .sanitize_draft_html` / `shared/allowlist.ts`'s `sanitizeHtml` — never the pre-existing
  `sanitize_rich_text` (that one strips highlights/placeholders the editor needs to preserve).
- The role-to-capability mapping, the HTML allowlist, and the intake contract are all parity-tested
  across TypeScript and Python — changing one side without the other (or without re-running
  `sync_shared.sh`) fails the drift tests (modulo the autocrlf false-failure above).
- Nothing in the SOW module writes to `SOWS`, `SOW_DRAFT_VERSIONS`, `HUBSPOT_DEALS`, or
  `SOW_NATIVE_*` except through the existing `save_sow_draft` call inside the `checkpoint` internal
  endpoint — a hard rule, not a current-state observation.
- Three `SOW_COLLAB_*` migrations (011, 012) are written and tested but **not yet applied** to
  Snowflake — that's a separate, explicitly-gated action requiring the database owner's approval.
- All three repos are on local commits not yet pushed to origin as of this handoff — see the latest
  `SESSION_LOG.md` entry for the exact commit list per repo and push status.

## Model/process notes for whoever continues this

- The user explicitly set **Sonnet 5.5 as the only model** for this module's work going forward — no
  Opus (cost reasons), no Haiku for actual code (this module is concurrency/security/data-integrity
  sensitive; a Haiku mistake here tends to be a silent, expensive-to-find bug). A careful self-review
  of the diff substitutes for a second-model review pass.
- Keep per-checkpoint tracker entries short (one tight paragraph), run targeted tests rather than the
  full `run_checks.sh` after every small change, and skip the production build check unless frontend
  bundling is actually touched — full verification still happens before any push.
