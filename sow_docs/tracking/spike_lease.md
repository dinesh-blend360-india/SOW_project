# Lease and fencing spike (CP-10)

**Status: PENDING LIVE RUN.** Same blocker as CP-08/CP-09 — no live Snowflake connection exists yet.
No acquire/renew latency, failover time, or pass/fail is reported here. Tracked in
`SOW_project/testing_pending_guide.md`.

## What exists

- `sow_module/collab/src/spikes/03-lease.ts` — fully implemented, typechecked and linted clean. Run
  with `npm run spike:lease`.

### What it does, once it can run

1. **Table**: `SOW_SPIKE_LEASE` (`deal_id`, `holder_id`, `fencing_token`, `expires_at`) — tries a
   hybrid table first, falls back to standard automatically, same pattern as the CP-09 script.
2. **Conditional acquire/renew**: a single `MERGE` statement, modeled directly on
   `crud.py`'s `_lock_sow_row_for_draft_write` — insert if no row exists, take over if the existing
   lease has expired, renew in place if already held by the caller. Wrapped in `acquireWithRetry`,
   which retries on a transient concurrent-DML error with a short backoff (expected on standard
   tables under contention; hybrid tables use row locking instead — **which kind actually happens is
   itself part of what this spike needs to observe live**, not something assumed here).
3. **Batched renew**: one `UPDATE ... WHERE holder_id = ? AND expires_at > ?` statement renews every
   room a given holder currently owns.
4. **Fenced write**: an `INSERT ... SELECT ... WHERE EXISTS (... fencing_token = ? AND expires_at > ?)`
   against a `SOW_SPIKE_LEASE_WRITE_LOG` table — inserts a row only if the caller's fencing token is
   still the current one.
5. **Race, 1 document**: `SPIKE_LEASE_HOLDERS` (default 5) separate Snowflake connections race to
   acquire the same document concurrently via `Promise.allSettled`.
6. **Race, 500 documents**: the same holders race across `SPIKE_LEASE_DOC_COUNT` (default 500)
   distinct documents.
7. **Zombie test**: holder A acquires a short-TTL lease, is simulated as paused (stops renewing) until
   it expires, holder B then acquires the same document, and holder A attempts a fenced write with its
   now-stale fencing token — this **must** insert 0 rows.
8. **Kill-and-failover test**: holder A acquires a lease and is simulated as killed (stops renewing);
   holder B polls every `SPIKE_LEASE_FAILOVER_POLL_MS` (default 1s) until it successfully acquires;
   elapsed time from kill to acquire is measured against a 30-second target.

## A note on "racing two processes"

The CP-10 prompt asks to race two separate processes. This script races multiple **separate Snowflake
connections from one Node process** instead, which exercises the same database-level conditional-write
race (the thing actually being tested) without the added complexity of orchestrating independent OS
processes. This is a reasonable stand-in for the database behavior, but is not identical to a true
multi-instance race — once this runs successfully in-process, it's worth also launching
`npm run spike:lease` from two separate terminals against the same scratch schema at the same time, to
confirm genuinely independent processes behave the same way, before treating the lease design as fully
proven.

## Still needed before this can run

- CP-08's Snowflake credentials and scratch schema.
- CP-08's hybrid-table result (decides which table kind this exercises).

## Reporting checklist for when it runs

- [ ] Acquire latency, 1-document race (p50/p95/p99, ms)
- [ ] Acquire latency, 500-document race (p50/p95/p99, ms)
- [ ] Batch-renew: rows affected by the single statement
- [ ] Zombie result: new fencing token after holder B's takeover; zombie fenced-write row count
      (**must be 0**)
- [ ] Failover time (ms) (**target <= 30,000ms**)
- [ ] Whether concurrent-DML retries were observed during the races (and how many), and whether that
      differed between hybrid and standard tables if both get tested
- [ ] **Pass/fail**: exactly one holder per document in both races, zombie count 0, failover within
      target — the script prints the combined gate result directly

## How to run this once credentials exist

```bash
cd blend_ode_be/sow_module/collab
npm run spike:lease
# optionally, in a second terminal at the same time, to test true multi-process racing:
npm run spike:lease
```
