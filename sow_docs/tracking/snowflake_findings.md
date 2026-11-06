# Snowflake access and capability spike (CP-08)

**Status: RUN (2026-11-06).** Credentials exist, the connection works, and the capability probes have
run for real. One probe (`CREATE HYBRID TABLE`) is denied, which directly fed the tier decision
recorded in `phaseB_report.md` and `migrations/011`.

## Capability probe result (2026-11-06, after the BLEND_INTERNAL_IT_APPS grant landed)

`npm run spike:capabilities`, against the now-accessible `BLEND_INTERNAL_IT_APPS.NEXDRAFT` (confirmed
empty, 0 tables, both before and after this run):

| Probe | Result |
|---|---|
| `CREATE HYBRID TABLE`, insert, read, drop | **FAIL** — `SQL access control error: Insufficient privileges ... must have CREATE HYBRID TABLE granted on SCHEMA BLEND_INTERNAL_IT_APPS.NEXDRAFT` |
| `BINARY` max value size | **PASS** — stored and read back 4,194,303 bytes via a regular `CREATE TABLE` |
| `TIMESTAMP_TZ` round-trip | **PASS** — via a regular `CREATE TABLE` |
| Application-table denial (`SELECT * FROM SOWS`) | **Correctly denied** — `Object 'SOWS' does not exist or not authorized`. This is the intended, desired outcome (the role must not read application tables), even though the raw script logs it as `[FAIL]` since the probe function is written to throw on success and treat the resulting exception as the expected case. |

**What this proves:** regular `CREATE TABLE` (and DML on it) works fine for `PRICING_APP_DEV_ROLE` in
this schema. Only `CREATE HYBRID TABLE` specifically is blocked, and that is a distinct Snowflake
grant from plain `CREATE TABLE` — not a broader permissions problem.

**Decision made from this result:** rather than request the additional `CREATE HYBRID TABLE` grant,
the user chose to proceed with standard tables (Tier 2) as the final decision for this module's
tables — see `phaseB_report.md` and `migrations/011_sow_collab_live_workspace.sql`'s header comment
for the full reasoning. CP-09's durable-ack latency gate (which would have added a real
performance-based data point to this decision) was not run before this choice was made.

## Measured result (2026-11-06)

`blend_ode_be/sow_module/collab/.env` was created, copying the Node service's Snowflake identity from
the backend's own working `.env` (same user, `PRICING_APP_DEV_USER`, same key, same role and
warehouse), with the database/schema set to the module's target location per ADR-012. The file is
git-ignored; no value was committed.

**`npm run spike:connect` succeeded** — real, measured, not projected:

```
CURRENT_VERSION():   10.40.102
CURRENT_ROLE():      PRICING_APP_DEV_ROLE
CURRENT_WAREHOUSE(): BLEND_INTERNAL
```

**But the session's database and schema came back `null`.** Snowflake did not error at connect time;
it silently left them unset. Two further read-only diagnostics (`check-nexdraft.ts`,
`check-existing-db.ts`, both `SHOW`/`SELECT` only, no writes) isolated why:

- `SHOW DATABASES LIKE 'BLEND_INTERNAL_IT_APPS'` → **0 results.** `PRICING_APP_DEV_ROLE` cannot see
  this database at all.
- `SHOW DATABASES LIKE 'RESOURCE_REQUEST_TOOL'` (the database the *existing* FastAPI app actually
  uses) → **1 result, found.** Same role, same key, same warehouse — this succeeds.

**Conclusion, not a guess:** the credentials, the key-pair auth, and the role are all fine.
`PRICING_APP_DEV_ROLE` has simply never been granted `USAGE` on `BLEND_INTERNAL_IT_APPS`, because the
existing codebase has never pointed at it — it lives entirely in `RESOURCE_REQUEST_TOOL`. This is a
grants gap, not a credentials gap. **The user is requesting the grant.** Once
`GRANT USAGE ON DATABASE BLEND_INTERNAL_IT_APPS TO ROLE PRICING_APP_DEV_ROLE;` and the matching schema
grant are applied, `check-nexdraft.ts` can confirm in seconds whether `NEXDRAFT` has any tables or
data, and `01-capabilities.ts` (below) can finally run.

**`01-capabilities.ts` (the hybrid-table/binary/timestamp/denial probes) was not run.** It would fail
immediately on the same missing-database-access error, so running it now would add no information.

## What exists

- `sow_module/collab/src/spikes/snowflakeClient.ts` — shared key-pair-auth connection helper, modeled
  on `blend_ode_be/src/common/database.py` (same key-pair/JWT authenticator style). Never logs a
  credential value.
- `sow_module/collab/src/spikes/00-connect.ts` — runs `SELECT CURRENT_VERSION(), CURRENT_ROLE(),
  CURRENT_WAREHOUSE()` and prints the three results. Run with `npm run spike:connect`.
- `sow_module/collab/src/spikes/01-capabilities.ts` — four probes, each independently caught and
  reported (one probe's failure doesn't stop the others):
  1. **Hybrid table**: `CREATE HYBRID TABLE SOW_SPIKE_PROBE (...)`, insert, read, drop.
  2. **BINARY size limit**: writes a value just under Snowflake's documented 8 MB `BINARY` column
     maximum and confirms it round-trips.
  3. **TIMESTAMP_TZ**: round-trips `CURRENT_TIMESTAMP()`.
  4. **Application-table denial**: attempts `SELECT * FROM SOWS LIMIT 1` — **this probe is expected
     to fail**. A successful read here is the failure case and is reported as `UNEXPECTED`.

  Run with `npm run spike:capabilities`. Both scripts drop every `SOW_SPIKE_*` object they create,
  including on failure (`finally` blocks), and touch no other table.
- `snowflake-sdk` was added to `sow_module/collab/package.json`, pinned to `^3.4.1` — the version that
  fixes a known high-severity advisory in older `snowflake-sdk` releases (GHSA-qqj6-54q6-cxv6: the
  driver could write sensitive information to logs). `@types/snowflake-sdk` and `dotenv` were added as
  dev dependencies. No other dependency was changed.
- `.env.example` gained `SNOWFLAKE_SPIKE_SCHEMA` (and a commented `SNOWFLAKE_PRIVATE_KEY` alternative
  to the path variable), documented as **distinct from the module's permanent schema**
  (`BLEND_INTERNAL_IT_APPS.NEXDRAFT`, decisions.md ADR-012) because these spikes create and drop
  objects freely.

## What is still needed before this can run for real

1. **The `BLEND_INTERNAL_IT_APPS` database grant** (above) — the actual current blocker.
2. ~~A Snowflake user and role for the Node service~~ — **resolved**: the user confirmed reusing
   `PRICING_APP_DEV_USER` / `PRICING_APP_DEV_ROLE` (the same identity the FastAPI app uses), not a
   separate one, per the restriction "only `PRICING_APP_DEV_USER`" stated 2026-10-30.
3. ~~A warehouse name~~ — **resolved**: `BLEND_INTERNAL` (copied from the working backend `.env`).
4. **A scratch schema is still not available.** Per the user's restriction, only `NEXDRAFT` may be
   used at all — there is no separate throwaway schema for spike churn. `SNOWFLAKE_SPIKE_SCHEMA` is
   currently set to `NEXDRAFT` itself in `.env`, a deliberate deviation from the original CP-08 design,
   flagged in that file and in `testing_pending_guide.md`. Once the database grant lands, the
   capability probes will create and immediately drop `SOW_SPIKE_*` objects inside `NEXDRAFT`.
5. ~~A private key~~ — **resolved**: same key the backend app already uses.

## Hybrid-table result

**Not yet determined.** Tier selection (`spike_durable_log.md`, CP-09) cannot be made until this runs.
Per the CP-08 prompt: if hybrid tables are unavailable, the plan is "Tier 2 fallback likely needed:
confirm with the user" — that decision is also pending.

## How to run this once credentials exist

```bash
cd blend_ode_be/sow_module/collab
cp .env.example .env   # fill in SNOWFLAKE_* and SNOWFLAKE_SPIKE_SCHEMA; never commit .env
npm run spike:connect
npm run spike:capabilities
```

Paste the printed summary block back into this file under a new "## Measured result (<date>)" heading
when it runs; do not overwrite this pending section — follow the project's append-only rule.
