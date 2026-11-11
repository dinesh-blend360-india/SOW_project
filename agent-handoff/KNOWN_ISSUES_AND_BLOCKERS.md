# Known Issues and Blockers

Each item records what's known, where it lives, and what (if anything) would resolve it. Items are
not ranked by severity in general, but the first one is the most urgent and is placed first
deliberately.

---

## 1. Real secrets committed to `blend_ode`'s tracked `.env` files and `MsalConfig.ts`

- **Repository**: `blend_ode`
- **Impact**: High. `blend_ode/.env.development` and `.env.production` contain what verification
  confirmed are real, working-looking secrets: a plaintext database password (`DB_DSN`), a
  `REACT_APP_AGENTS_API_KEY` (different value per environment, both committed), and in
  `.env.development`, a CKEditor Premium trial license token that decodes as a real JWT with a live
  `usageEndpoint`. `src/config/MsalConfig.ts` additionally has a hardcoded Azure AD `clientId` and
  tenant GUID, also tracked.
- **Reproduction**: `git show <any-historical-commit>:.env.development` in `blend_ode`, or simply
  `cat .env.development` in a checkout — the file is tracked, not gitignored at the time it was
  committed.
- **Why it's still a problem**: `.gitignore` was later updated to list these files, but that does
  not retroactively untrack already-committed content or remove it from history. The secrets are
  still reachable by anyone who can clone the repo and look at its history.
- **Possible resolution**: rotate every exposed credential (DB password, Agents API key, CKEditor
  license token, and treat the Azure AD client ID/tenant as at minimum needing an audit of what
  it's scoped to); then scrub history (`git filter-repo` or equivalent) if the org's policy requires
  it, coordinated with whoever owns each credential and the GitHub remotes. This is a decision for
  the project owner, not something to silently fix as a side effect of an unrelated task.

## 2. No CI pipeline runs tests automatically, in either repo

- **Repository**: both
- **Impact**: Medium-high. `blend_ode_be`'s only GitHub Actions workflow (`main.yml`) is a manual
  (`workflow_dispatch`) build-and-deploy job with no test or lint step. `blend_ode` has no CI
  workflow at all (only a PR template). All testing today is developer-run, which means a
  regression can reach `main`/a deployed environment without ever being caught by a pipeline.
- **Possible resolution**: add a CI workflow (triggered on PR) in each repo that runs the commands
  in `TESTING_AND_VALIDATION.md` and fails the PR on test failure — note the 9 pre-existing backend
  failures would need to be xfail'd or fixed first, or the gate would be permanently red.

## 3. No `.env.example` in `blend_ode`

- **Repository**: `blend_ode`
- **Impact**: Medium. Combined with issue #1, a new developer currently has no safe way to discover
  which `REACT_APP_*` variables are needed without reading the secret-containing tracked files.
  `ENVIRONMENT_AND_SECRETS.md` in this handoff system now documents the variable names as a
  stopgap, but a real `.env.example` with placeholder values should still be created in that repo.
- **Possible resolution**: once issue #1's rotation is done, create `.env.example` from the rotated
  file with every real value replaced by a placeholder, and commit that instead.

## 4. `REACT_APP_CKEDITOR_*` environment variables appear to be dead configuration

- **Repository**: `blend_ode`
- **Impact**: Low-medium (mostly a cleanliness/confusion risk, not a functional bug today).
  `DraftEditor.tsx` hardcodes `licenseKey: 'GPL'` in its editor config; the
  `REACT_APP_CKEDITOR_ENABLE_CLOUD_SERVICES`/`_CLOUD_TOKEN_URL`/`_LICENSE_KEY` variables are not
  referenced in any non-test runtime source file found — only in `DraftEditor.test.tsx`, which
  manipulates `process.env` for test setup. This suggests either the Premium/cloud-services
  integration was never finished, or it's wired up somewhere not found in this inventory.
- **Possible resolution**: confirm with whoever owns the CKEditor licensing whether Premium/cloud
  services are actually meant to be active; either wire `DraftEditor.tsx` to read the license key
  from env, or remove the unused variables (and rotate/retire the committed license token per
  issue #1 either way, since it's currently exposed regardless of whether it's wired up).

## 5. Two Snowflake migration folders with an unclear relationship

- **Repository**: `blend_ode_be`
- **Impact**: Medium. `migrations/` has 12 sequential files (`001` through `012`, the active one —
  confirmed by the SOW module's own CP-16 work choosing migration number 012 next in that folder).
  A separate `db/migrations/` folder exists with only `002_pricing_template_deletion.sql` and
  `003_fractional_duration.sql` — it's unclear whether this is an older, abandoned folder, a
  parallel system for a different purpose, or something still actively used by a process not
  discovered in this inventory.
- **Possible resolution**: ask the project owner directly whether `db/migrations/` is still live;
  do not assume either way, and do not delete or add to it without that confirmation.

## 6. `Pipfile` vs. `requirements.txt` — unclear which is authoritative for the backend

- **Repository**: `blend_ode_be`
- **Impact**: Low-medium. The README instructs `pipenv install`; the Dockerfile (what's actually
  deployed) and CI both use `requirements.txt` directly via `pip install`. If the two dependency
  lists ever diverge, a developer following the README could end up with a different environment
  than what's deployed.
- **Possible resolution**: diff the two files' package lists; standardize on one (most likely
  `requirements.txt`, since it's what's proven to work in the deployed container) and update the
  README to match, or regenerate one from the other and document the regeneration step.

## 7. The SOW collaboration module's `checkpoint` endpoint doesn't independently re-verify the fencing token

- **Repository**: `blend_ode_be`
- **Impact**: Low today (the feature isn't live yet), but worth tracking before it becomes load-
  bearing. `collab_internal.checkpoint()` accepts `fencingToken`/`upToSeq` in its request but does
  not check them against the live `SOW_COLLAB_LEASE` row — FastAPI's Snowflake connection targets
  the main application schema, not `BLEND_INTERNAL_IT_APPS.NEXDRAFT` where the lease lives. This is
  a documented, explicit scope decision (not an oversight) made during CP-20 of the SOW module's
  build — see `sow_docs/tracking/decisions.md` ADR-007/012 and `CHANGELOG.md`'s CP-20 entry.
- **Current mitigation**: the Node service itself enforces fencing before any batch write (CP-18's
  `appendUpdates`), so a stale/zombie writer cannot reach FastAPI with the wrong result in practice
  today — but FastAPI itself has no independent check.
- **Possible resolution**: build a second, narrowly-scoped Snowflake connection in FastAPI into
  `NEXDRAFT` purely to re-verify the fencing token at checkpoint time, if/when this is judged worth
  the added complexity. Not currently scheduled — see `TASK_BOARD.md`.

## 8. Open product/role-mapping question: who exactly is `sow_admin` (finalize-capable)?

- **Repository**: `blend_ode_be` (and conceptually `blend_ode`, since the role model is shared)
- **Impact**: Medium for the SOW collaboration module specifically. The code today treats
  `FE_ADMIN` as the only elevated flag for this purpose (a documented stand-in used in CP-19/20 of
  the module's build), but `sow_docs/tracking/decisions.md`'s Q2 — "should `sow_admin` be `FE_ADMIN`
  only, a deal-level role, or everyone who can edit today?" — is explicitly still unanswered by the
  user.
- **Possible resolution**: get an explicit answer to Q2 from the project owner; until then, do not
  assume the current `FE_ADMIN`-only mapping is final — it's a placeholder, not a decision.

## 9. 9 pre-existing backend test failures (see `TESTING_AND_VALIDATION.md` for the exact list)

- **Repository**: `blend_ode_be`
- **Impact**: Low-medium — they're date-dependent template assertions and one prompt-wording
  assertion, not obviously load-bearing, but nobody has fixed or triaged them, and their presence
  makes a genuinely new regression harder to spot by eye in a full test run.
- **Possible resolution**: triage each one; the date-dependent ones likely need either a frozen
  test clock or a relative-date assertion instead of a hardcoded string.

## 10. Frontend has two near-duplicate axios wrapper modules for SOW Studio

- **Repository**: `blend_ode`
- **Impact**: Low. `src/config/ApiService.ts`'s `sowStudioApi` and
  `src/pricing-tool/services/nativeEditorApi.ts` both implement the same base-URL-fallback and JWT-
  interceptor logic independently.
- **Possible resolution**: consolidate into one shared client if/when either is next touched; not
  urgent on its own.

## 11. antd and Bootstrap CSS are both loaded globally

- **Repository**: `blend_ode`
- **Impact**: Low — a possible source of unexpected style bleed/specificity conflicts, not
  confirmed to currently cause a visible bug.
- **Possible resolution**: none proposed; flagged for awareness only.

## 12. The SOW collaboration editor is behind a feature flag and not yet verified end to end

- **Repository**: `blend_ode` and `blend_ode_be`
- **Impact**: Medium. The Lexical editor, its WebSocket client and the Owner checkpoint all exist and
  pass unit and server tests. Nothing has been run with two real users. The two-user test in
  `sow_docs/tracking/phaseD_report.md` is still to do.

## Update 2026-11-09
- **RESOLVED:** pricing pages crash with "Cannot read properties of null (reading 'replace')" when a deal has no project start date (8853099).
- **RESOLVED:** editor import error for `LexicalErrorBoundary` (485d3cc). Collab service would not start on Windows (166120c).
- **OPEN:** `/pricing_tool/hubspot_deals_sync_info` returns 500. The browser reports it as CORS because error responses lose the CORS headers. Cause unknown until the backend traceback is captured.
- **OPEN:** live editor returns 403 for users without `FE_SOW_LEXICAL_EDITOR`/`FE_ADMIN` or deal access. This is an access rule, not a bug.
- **OPEN:** `blend_ode/src/App.tsx` was truncated once outside this session. Recovered from git. Check that no one else edits it.

## 13. This Windows checkout's `core.autocrlf=true` breaks hash-exact-match tests in both repos

- **Repository**: both (`blend_ode`'s `sharedDrift.test.ts`/`roles.test.ts`, `blend_ode_be`'s
  `test_sow_module_shared_drift.py`, and `sow_module/run_checks.sh`'s `sync_shared.sh --check` step)
- **Impact**: Low-medium — it's a false failure, not real drift, but it will keep failing on any
  machine with this git config until fixed, and could be mistaken for a real regression.
- **Found**: 2026-11-11, while establishing a test baseline before CP-40 work. `git config --get
  core.autocrlf` returns `true` on this checkout. Several `sow_module/shared` fixture files and the
  vendored `collab/src/shared` copy show as locally modified purely from LF→CRLF normalization —
  confirmed with `git diff --ignore-all-space`, which shows zero real content difference. The
  drift/parity tests compute a SHA-256 hash of file contents, which differs between LF and CRLF, so
  they fail even though the actual tracked content is unchanged.
- **Possible resolution**: add a `.gitattributes` entry forcing LF for `sow_module/shared/**` (and
  the vendored collab copy), or set `core.autocrlf=false`/`input` for these repos. Not done this
  session — it's outside CP-40's scope and changes shared config, so it needs the project owner's
  sign-off rather than a silent fix alongside unrelated work.

## Update 2026-11-11
- **OPEN:** a change to the draft made outside the editor is refused by the Owner and shown as a notice;
  the room is not reloaded. Users must reload the page. Needs a design before the Owner can reset a room.
- **OPEN:** awareness has no throttle yet, and presence and cursors are not verified with two browsers.
- **OPEN:** the server does not enforce the 3-level list depth cap; only the editor and the TypeScript
  allowlist do (`html_allowlist.py` is a streaming sanitizer and needs a tree-based rule).
- **OPEN:** merged-cell tables do not export correctly to docx or pdf (the exporters flatten tables).
- **OPEN:** `blend_ode_be/.env.example` has no `COLLAB_*` variables. The Node service's own
  `sow_module/collab/.env.example` is missing `COLLAB_CHECKPOINT_QUIET_MS` (default 30000 in code).
- **RESOLVED for print:** verified with a generated PDF (one A4 page). Earlier print output was Letter size
  and included the toolbar.
