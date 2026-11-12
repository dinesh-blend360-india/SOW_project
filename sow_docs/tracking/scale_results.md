
## CP-68 local broadcast re-run (2026-11-13) -- local, not production-shaped
Harness: blend_ode_be/sow_module/collab/src/spikes/05-scale.ts. One Node process: the owner and all clients. 15 s per run.
| Clients | p50 ms | p95 ms | p99 ms | max ms | CPU avg | RSS after |
|---|---|---|---|---|---|---|
| 10 | 1 | 3 | 3 | 4 | 1.0% | 66 MB |
| 25 | 2 | 5 | 6 | 9 | 5.8% | 58 MB |
| 50 | 3 | 10 | 21 | 55 | 14.7% | 94 MB |
| 100 | 11 | 84 | 128 | 152 | 47.3% | 109 MB |
Not measured here (need Snowflake, which has no grant yet, or a separate harness): durable-ack p95 (gate <= 1500 ms), failover
(gate <= 30 s), zero loss (gate), checkpoint time at scale, revoke-to-close at scale, mention search, session open at scale.
Broadcast p95 at 100 clients is 84 ms here (72 ms in phase B): no formal gate exists for broadcast; this is informational.
