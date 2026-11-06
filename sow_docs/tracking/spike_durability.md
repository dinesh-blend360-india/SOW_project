# Crash durability and client-held-edits spike (CP-11)

**Status: PENDING LIVE RUN.** Same blocker as CP-08/09/10 — no live Snowflake connection exists yet.
No kill count, loss count, or duplicate count is reported here. Tracked in
`SOW_project/testing_pending_guide.md`.

## What exists

- `sow_module/collab/src/spikes/04-durability.ts` — fully implemented, typechecked and linted clean.
  Run with `npm run spike:durability`.

### What it does, once it can run

1. **Real Yjs, not synthetic bytes.** Each simulated client (`SimClient`) owns a real `Y.Doc`; every
   edit fires Yjs's own `update` event, and that real update byte array — assigned a sequence number —
   is what travels to the simulated owner and what gets persisted. This is what makes the merge-
   idempotency property genuinely testable rather than assumed.
2. **The client holds every update it sends until it is told `DURABLE {upToSeq}`.** `SimClient.pending`
   is the held buffer; `onDurable(upToSeq)` is the only thing that clears entries from it.
3. **The owner (`SimOwner`) batches and flushes** on the same timer pattern as CP-09, but writes with a
   `MERGE INTO ... WHEN NOT MATCHED THEN INSERT` keyed on `(deal_id, seq)` instead of a plain bulk
   insert — so if a client resends an update the owner already durably wrote (because the owner
   crashed after writing but before the `DURABLE` ack reached the client), the resend is a no-op at
   the storage level rather than a duplicate row. This is the mechanism, not an assumption, that
   should make the "duplicates must be 0" property hold.
4. **The owner is killed repeatedly** (`SPIKE_DURABILITY_KILLS`, default 5 rounds): edits are generated
   continuously, then the owner's flush timer is stopped immediately — deliberately without waiting for
   an in-flight batch to finish — so some updates are left queued and un-persisted at the moment of
   the kill. A fresh `SimOwner` then starts, and every client resends whatever is still in its
   `pending` buffer (the reconnect-and-resend behavior the real system will need).
5. **Final verification**, after a last kill-free drain so every client's buffer should empty:
   - **Durably-acked loss**: for every `(dealId, seq)` a client was ever told was `DURABLE`, is it
     actually present in `SOW_SPIKE_DURABILITY_LOG`? Must be 0.
   - **Duplicates**: group the stored table by `(deal_id, seq)` and count groups with more than one
     row. Must be 0 (this is the `MERGE`-based dedup actually being exercised, not just claimed).
   - A secondary sanity count — updates still unacknowledged after the final drain — is also printed;
     it isn't one of the two required numbers but a nonzero value here would itself indicate a bug in
     the simulation's drain logic, worth investigating before trusting the other two numbers.

## Still needed before this can run

- CP-08's Snowflake credentials and scratch schema.

## Reporting checklist for when it runs

- [ ] Number of kill rounds actually executed (should equal `SPIKE_DURABILITY_KILLS`, default 5)
- [ ] Durably-acked loss count (**must be 0**)
- [ ] Duplicate `(deal_id, seq)` count (**must be 0**)
- [ ] Unacknowledged-at-end count (sanity check; investigate if nonzero)
- [ ] **Gate result**: both required counts are 0 — the script prints this directly
- [ ] Anything unexpected observed about hybrid vs. standard table behavior under the repeated
      kill/resend pattern (the script auto-detects which kind it got, same as CP-09/CP-10)

## How to run this once credentials exist

```bash
cd blend_ode_be/sow_module/collab
npm run spike:durability
```
