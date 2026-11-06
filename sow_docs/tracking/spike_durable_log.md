# Batched durable-log latency spike (CP-09) — the key architecture-gating risk

**Status: SUPERSEDED BY A USER DECISION (2026-11-06), still not run.** This checkpoint's entire
purpose was to measure real durable-ack latency against Snowflake and use it to decide the storage
tier. That measurement still hasn't happened — but the tier decision it was meant to inform has
already been made: the user chose standard tables (Tier 2) after `CREATE HYBRID TABLE` turned out to
need a separate grant `PRICING_APP_DEV_ROLE` doesn't have, and after CP-15's broadcast spike showed
nothing pathologically slow. See `migrations/011_sow_collab_live_workspace.sql`'s header comment and
`phaseB_report.md` for the full record. **This spike remains available to run later**, purely for
confidence/validation now that standard-table `CREATE TABLE` access exists — running it is no longer
blocking anything, since the decision it was gating has already been made without it.

## What exists

- `sow_module/collab/src/spikes/02-durable-log.ts` — fully implemented, typechecked and linted clean.
  Run with `npm run spike:durable-log`. Configurable via env (`SPIKE_DURATION_MS`, `SPIKE_DOC_COUNT`,
  `SPIKE_FLUSH_INTERVAL_MS`, `SPIKE_UPDATE_SIZE_BYTES`) so the same script can do a 30-second smoke
  test before committing to the full 5-minute run.
- `sow_module/collab/src/spikes/percentiles.ts` — p50/p95/p99 helper, shared with CP-10 and CP-11.

### What the script measures, once it can run

1. **The core gate**: 100 simulated documents, each sending an update every 1-2 seconds for 5 minutes,
   queued in memory and flushed to `SOW_SPIKE_UPDATE` (hybrid table if CP-08 found hybrid tables
   available, else a standard table — the script tries hybrid first and falls back automatically) as
   one multi-row bulk-bound insert every 500ms. Every update's enqueue-to-commit latency is recorded.
2. Single-row insert, 100-row insert, 1000-row insert timing.
3. A point read and a conditional update (simulating the optimistic-lock pattern `save_sow_draft`
   already uses).
4. Statements-per-minute during the simulation (for warehouse-cost reasoning).
5. A 5 MB write split into 1 MB chunked `BINARY` rows (conservative against the ~8 MB column limit
   CP-08 is expected to confirm), with a read-back byte-count check.

## Still needed before this can run

- CP-08's Snowflake credentials and scratch schema.
- Roughly 5-6 minutes of uninterrupted access to that schema for the full simulation (a shorter smoke
  run via `SPIKE_DURATION_MS` is recommended first).
- CP-08's hybrid-table result, so this report can note which table kind was actually exercised.

## Reporting checklist for when it runs

Fill in all of these from the script's actual printed output — do not estimate:

- [ ] p50, p95, p99 durable-ack latency (ms)
- [ ] min / max / mean durable-ack latency (ms)
- [ ] single-row / 100-row / 1000-row insert timing (ms)
- [ ] point-read timing (ms)
- [ ] conditional-update timing (ms) and rows-affected count
- [ ] statements per minute during the simulation
- [ ] 5 MB chunked-binary write time and read-back byte count (pass if it equals 5,242,880)
- [ ] **Gate result**: p95 durable-ack <= 1500ms? PASS or FAIL (the script prints this directly)
- [ ] Estimated warehouse cost per active hour — compute as
      `warehouse_credits_per_hour × (active_seconds / 3600)`, where `warehouse_credits_per_hour`
      depends on the warehouse size chosen for CP-08 (not yet known) and `active_seconds` is how long
      the warehouse actually ran hot during the test (visible in Snowflake's query history /
      warehouse metering, not computed by the script itself)
- [ ] **Recommendation**: Tier 1 (as planned, hybrid tables), Tier 2 (single-owner task, standard
      tables), or escalate to the user — based on the gate result and the hybrid-table availability
      from CP-08, not guessed here

## How to run this once credentials exist

```bash
cd blend_ode_be/sow_module/collab
# quick smoke test first:
SPIKE_DURATION_MS=30000 SPIKE_DOC_COUNT=10 npm run spike:durable-log
# then the full spike:
npm run spike:durable-log
```
