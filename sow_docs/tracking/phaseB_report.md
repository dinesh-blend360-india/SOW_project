# Phase B report: collaboration spikes (CP-08 to CP-15)

**Overall decision: TIER DECIDED by the user (2026-11-06); broader Phase B spikes still not fully
run.** The `BLEND_INTERNAL_IT_APPS` grant landed. A live capability check found regular `CREATE
TABLE` works for `PRICING_APP_DEV_ROLE` but `CREATE HYBRID TABLE` is denied (needs its own separate
grant). Rather than request it, **the user chose to proceed with standard tables (Tier 2) as the
final decision**, after CP-15's broadcast spike showed nothing pathologically slow — a deliberate
call made without CP-09's durable-ack latency number, which still hasn't been measured and is no
longer blocking this decision. This report records that plainly rather than treating a user decision
as if it were a measured one. CP-09, CP-10, and CP-11's full live runs remain open as a separate,
non-blocking item — they would add confidence, not change what's already been decided.

## What every spike in Phase B actually found

| Checkpoint | What it's for | Status | Key result |
|---|---|---|---|
| CP-08 | Snowflake connection, hybrid-table availability, access denial | **Partially run** | Connection succeeds with real credentials (`PRICING_APP_DEV_USER`). Capability probes (hybrid table, binary limit, application-table denial) blocked — not by credentials, but by a missing `USAGE` grant on `BLEND_INTERNAL_IT_APPS` (isolated and confirmed 2026-11-06; see `snowflake_findings.md`) |
| CP-09 | Durable-ack latency — **the architecture-gating measurement** | **Not run** | Code complete, typechecked, linted. No number exists. Gate: p95 ≤ 1500ms, unmeasured |
| CP-10 | Lease acquire/renew, fencing, zombie-writer rejection, failover timing | **Not run** | Code complete. No number exists |
| CP-11 | Crash durability: 0 durable-acked loss, 0 duplicates | **Not run** | Code complete, including a real-Yjs-update simulation. No number exists |
| CP-15 | 100-client broadcast fan-out | **Run — local, not production-shaped** | p50 9ms / p95 72ms / p99 170ms broadcast latency; see `spike_scale.md`. Explicitly does not include the CP-09/CP-10 Snowflake-backed persistence or failover paths |
| CP-12 | HTML round-trips through Lexical with no defect | **Run, GO** | 23/23 fixtures, zero defects (`spike_html_fidelity.md` Part 1) |
| CP-13 | Backend parity with the TypeScript allowlist; HTML GO/NO-GO | **Run, GO** | 81/81 tests pass; verdict recorded: HTML stays the system of record (`spike_html_fidelity.md` Part 2) |
| CP-14 | Paste/LLM-text intake through one allowlist | **Run** | 25 tests + 400 fuzzed inputs pass; two real defects found and fixed (`spike_intake.md`) |

**The pattern worth naming:** every spike that *could* run locally (CP-12, CP-13, CP-14, and CP-15's
broadcast portion) found real, non-hypothetical bugs by actually executing code against real
libraries and real-shaped input — not by reasoning about what should work. Every spike that needs live
Snowflake (CP-08's capability probes, CP-09, CP-10, CP-11, and CP-15's persistence/failover portions)
is still unmeasured, for the same single reason: a missing database grant, not a missing design.

## The tier decision

**Made by the user, 2026-11-06: Tier 2 (standard tables).** Tier 1 (hybrid tables) vs. Tier 2 was
always meant to be decided by CP-08's hybrid-table probe and CP-09's latency gate. CP-08's probe did
run, partially: `CREATE HYBRID TABLE` is confirmed denied for `PRICING_APP_DEV_ROLE` (needs its own
separate grant), while regular `CREATE TABLE` works. CP-09's latency gate never ran. Given that, and
given CP-15's broadcast spike showed nothing pathologically slow, the user chose to proceed with
standard tables rather than request the additional hybrid-table grant. This is recorded as a decision
made with partial information, not a fully measured one — see `migrations/011_sow_collab_live_workspace.sql`'s
header comment, which carries the same explanation for anyone reading the migration directly.

## Gate values for later phases (as specified; no longer blocking the tier decision)

- CP-09 gate: p95 durable-ack ≤ 1500ms — **unmeasured**, optional follow-up, not blocking
- CP-10 gate: failover ≤ 30s, zero zombie writes — **unmeasured**, optional follow-up, not blocking
- CP-11 gate: zero durable-acked loss, zero duplicates — **unmeasured**, optional follow-up, not blocking
- CP-15 local broadcast (informational, not a hard gate): p95 72ms at 100 clients on one machine,
  single-process — **measured**, see above

## Re-baseline note

Per the CP-15 prompt's own instruction: "the re-baseline note (team size and schedule to be confirmed
by the user)." **Not yet confirmed.** This report is being produced with the Snowflake-dependent
numbers missing; once the grant lands and CP-08/09/10/11/15's remaining portions are actually run, the
team size and rollout schedule this report's numbers should be re-baselined against needs to come from
the user, not assumed here.

## GO / NO-GO: **GO on the tier decision (user-made, 2026-11-06); CP-09/10/11's full live runs remain open separately**

The tier question is resolved: standard tables, by explicit user decision. This is not the same as
saying CP-09's durable-ack gate, CP-10's failover gate, and CP-11's crash-durability gate have been
measured and passed — they haven't, and `migrations/011`'s own comment says so plainly so a future
reader isn't misled into thinking a side-by-side performance comparison happened. Those three spikes
remain available to run (grants now exist for standard-table `CREATE TABLE`, so they're no longer
blocked the way they were on 2026-11-01), and would add real confidence in the chosen tier, but they
are no longer gating anything — the user has made the call and asked to proceed.
