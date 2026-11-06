# 100-client room spike (CP-15)

**Status: PARTIALLY RUN (2026-11-06).** The broadcast-fan-out portion is real and measured. The
persistence-under-load and lease-based failover portions are still pending the `BLEND_INTERNAL_IT_APPS`
grant (see `snowflake_findings.md`) and are not measured here.

**Label, per the CP-15 prompt's own instruction: this is local, not production-shaped.** Every number
below comes from one Node process, on one development machine, with the "owner" and all 100 "clients"
running in that same process and talking over `localhost`. There is no real network, no separate
machines, and no load from anything else on the box. Treat these as a lower bound and a sanity check
that nothing is pathologically slow — not as a production capacity estimate.

## What was measured: broadcast fan-out (`npm run spike:scale`)

`sow_module/collab/src/spikes/05-scale.ts` starts a minimal WebSocket "owner" holding one `Y.Doc`,
connects 100 real Yjs clients to it, and has each client make a small edit every 2-4 seconds (steady
typing) while also sending an awareness ping once a second. The owner rebroadcasts every message to
every other connected client — the simplest possible fan-out, no batching, no filtering.

**30-second run, 100 clients:**

| Metric | Result |
|---|---|
| Broadcast latency p50 | 9 ms |
| Broadcast latency p95 | 72 ms |
| Broadcast latency p99 | 170 ms |
| Broadcast latency min / max | 1 ms / 250 ms |
| Broadcast latency mean | 19.4 ms |
| Sample size | 97,812 one-sender-to-one-receiver deliveries |
| Process RSS before / after | 66.0 MB / 137.4 MB |
| Process CPU (average over the run) | 47.7% (one process holding the owner AND all 100 clients) |
| Awareness messages delivered | 287,100 (~9,569/s combined) |

**Reading these numbers honestly:** the CPU and memory figures conflate the owner's real work with
100 simulated clients' Yjs overhead running in the same process — a real deployment would never do
this, so these are not owner-only resource figures. The broadcast latency itself (one real WebSocket
hop, in-process scheduling, no network) is the more meaningful number, and even that is optimistic
compared to a real network with real geographic distance between the owner and its clients.

## What was measured: proxy-hop overhead (`npm run spike:proxy-hop`)

`06-proxy-hop.ts` measures the added latency when a client connects to a non-holder instance and gets
proxied to the real holder — using a **hardcoded routing table as a stand-in for the CP-10 lease
lookup**, since the real lease needs the same pending Snowflake grant. This isolates the cost of one
extra local WebSocket hop, not the cost of a real lease lookup.

| | Direct to holder | Via proxy (non-holder → holder) |
|---|---|---|
| p50 | 0 ms | 0 ms |
| p95 | 1 ms | 1 ms |
| p99 | 1 ms | 1 ms |

**Added latency from the proxy hop itself: effectively 0ms at this scale, on loopback.** This is
expected — both "machines" are the same machine. It confirms the proxy mechanism doesn't add
meaningful overhead on its own; it says nothing about what a real network hop between two actual
instances would cost, and nothing about how long the real lease lookup (a Snowflake query) would add
on top of the hop once that part can be tested.

## A bug found and fixed while building this spike

The first version of `06-proxy-hop.ts` hung indefinitely: the proxy attached its "forward client
messages to the holder" handler only inside the upstream connection's `open` callback, so a client
message that arrived before that upstream connection finished opening was silently dropped, and the
client waited forever for a reply that would never come. Fixed by buffering client messages that
arrive before the upstream is ready, and flushing the buffer once it opens. This is the same category
of lesson as CP-12/13/14's findings: an untested assumption about event ordering produced a real,
reproducible failure, found only by actually running the code.

## Not measured (pending the Snowflake grant)

- **Persistence overhead under the same 100-client load** — integrating the CP-09 flusher so every
  client's edits are actually written to `SOW_SPIKE_UPDATE`, and seeing how durable-ack latency
  changes under concurrent load from 100 documents, not the single synthetic one CP-09 tested serially.
- **Real failover timing** — killing a real lease-holding owner process and timing how long a second
  instance, using the actual CP-10 lease, takes to take over and resume serving the 100 clients.
- **Two or three owner processes racing for real documents**, as the CP-15 prompt specifies, rather
  than the single in-process owner used here.

These three items are why **Phase B's overall GO/NO-GO is recorded as PENDING, not GO**, in
`phaseB_report.md` — the broadcast numbers above are encouraging, but the architecture-gating
decision (CP-09's tier choice) was never about broadcast fan-out; it was always about whether
Snowflake-backed persistence can keep up, and that still hasn't been measured.
