# Testing and Validation

Never state a test passed unless it was actually executed and the output was actually read. Every
result below is marked with when it was last actually run.

## Backend (`blend_ode_be`)

- **Framework**: pytest. No `conftest.py`, no `pytest.ini`/lint config — runs with pytest defaults.
- **Location**: 60 files, flat under `tests/`, plus `tests/fixtures/` for test data.
- **Run everything**:
  ```bash
  cd blend_ode_be
  python -m pytest tests/ -q
  ```
  **Last actually run** (this session, 2026-11-06): `555 passed`, `9 failed`. The 9 failures are
  **pre-existing and unrelated** to any change made in this session — confirmed by re-running a
  narrower baseline before and after each change and seeing the same 9 names recur:
  - `tests/test_pricing_duration_schema.py::test_user_deal_rounds_fractional_duration_in_months_to_a_whole_month`
  - `tests/test_pricing_duration_schema.py::test_duration_rounding_uses_half_up_semantics`
  - `tests/test_pricing_duration_schema.py::test_positive_duration_less_than_one_month_becomes_one_month`
  - `tests/test_sow_v4_templates.py::test_team_dates_location_and_currency[Retainer]`
  - `tests/test_sow_v4_templates.py::test_team_dates_location_and_currency[Time & Materials]`
  - `tests/test_sow_v4_templates.py::test_effective_date_uses_project_start_without_filling_signature_dates[Fixed Fee]`
  - `tests/test_sow_v4_templates.py::test_effective_date_uses_project_start_without_filling_signature_dates[Retainer]`
  - `tests/test_sow_v4_templates.py::test_effective_date_uses_project_start_without_filling_signature_dates[Time & Materials]`
  - `tests/test_sow_v4_templates.py::test_packaged_agent_instructions_match_runtime_prompt_constants`

  The `test_sow_v4_templates.py` failures appear date-dependent (they assert a hardcoded date
  string against "today's" rendered output) and the prompt-constants one appears to assert exact
  wording that has since drifted from the packaged template — neither was investigated further as
  it was out of scope for the sessions that found them. **Before trusting this list**, re-run the
  suite yourself — don't assume it's still exactly 9 by the time you read this; the point of
  recording this is so a 10th failure is immediately recognizable as new, not that this list is
  permanent.
- **SOW module subset** (narrower, used by `run_checks.sh`):
  ```bash
  python -m pytest tests/test_sow_module_smoke.py tests/test_sow_module_shared_drift.py \
    tests/test_sow_module_html_parity.py tests/test_sow_module_migration_011.py \
    tests/test_sow_module_migration_012_grants.py tests/test_sow_module_role_table_parity.py \
    tests/test_sow_module_collab_session.py tests/test_sow_module_collab_internal.py -q
  ```
  Last run: **115 passed**.
- **Live-Snowflake integration tests** (not run by default; see `blend_ode_be/sow_module/collab`'s
  Vitest suite below — there is no equivalent live-Snowflake pytest suite on the Python side as of
  this writing).

## Frontend (`blend_ode`)

- **Framework**: Jest via `react-scripts` (Create React App), with `@testing-library/react`.
- **Location**: mixed — colocated `*.test.ts(x)` files and `__tests__/` folders.
- **Run**:
  ```bash
  cd blend_ode
  npm test
  ```
  Interactive watch mode by default (CRA's standard behavior); pass `CI=true` to run once and exit,
  which is what `blend_ode_be/sow_module/run_checks.sh` does for the `sow_module`-scoped subset.
  **Last actually run** (SOW module subset only, via `run_checks.sh`, 2026-11-06): 5 test suites,
  **68 passed**. The full, unscoped `npm test` for the entire frontend app was **not run** in this
  session — do not assume it currently passes; run it before relying on that claim.

## SOW collaboration module's Node service (`blend_ode_be/sow_module/collab`)

- **Framework**: Vitest.
- **Run**:
  ```bash
  cd blend_ode_be/sow_module/collab
  npm run typecheck   # tsc --noEmit
  npm run lint        # eslint src
  npm test            # vitest run
  ```
  Last actually run (2026-11-06): typecheck clean, lint clean, **11 passed, 6 skipped** (the 6
  skipped are the live-Snowflake integration tests, gated behind `RUN_SNOWFLAKE_INTEGRATION=1`).
- **Live-Snowflake integration run** (opt-in, writes temporary tables to real Snowflake and drops
  them — requires real credentials and should not be run casually):
  ```bash
  RUN_SNOWFLAKE_INTEGRATION=1 npm test -- --run src/data/snowflake-store.integration.test.ts
  ```
  Last actually run: **6 passed**, against `BLEND_INTERNAL_IT_APPS.NEXDRAFT` with temporary
  `SOW_COLLAB_TEST_<id>_*` tables, confirmed cleaned up afterward (schema back to 0 tables). This
  run is what found and fixed a real binary-bind bug — see `CHANGELOG.md` CP-18 entry.

## Cross-repo check script

```bash
bash blend_ode_be/sow_module/run_checks.sh
```
Runs, in order: shared-code sync check (fails if the vendored copy in `blend_ode_be` has drifted
from `blend_ode`'s source of truth), frontend Jest (SOW module scope only), frontend shared
typecheck, collab-service typecheck/lint/Vitest, backend module pytest subset. **Last actually run**
(2026-11-06): **7/7 PASS**. This is the fastest single command to confirm the SOW module hasn't
regressed; it does not cover the rest of the app (Pricing Tool, the older SOW Studio flows, the full
frontend test suite).

## Build verification

```bash
cd blend_ode && npm run build
```
**Not run** in any session recorded here — confirm it succeeds before relying on that claim.

## API testing

No dedicated API test collection (Postman, etc.) was found in either repo. API-level testing today
is done through the pytest suite's `TestClient`-based tests (`fastapi.testclient.TestClient`,
following the pattern in `tests/test_sow_http_e2e.py`) — dependency-override based, not hitting a
live server.

## Regression testing

There is no CI pipeline that runs tests automatically (`blend_ode_be`'s only GitHub Actions
workflow is a manual deploy; `blend_ode` has none at all). Regression testing today is entirely
manual/developer-run. See `KNOWN_ISSUES_AND_BLOCKERS.md` for the implications of this.
