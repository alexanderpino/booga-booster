# game-0005: Simulation authority and state synchronisation

- Status: Proposed (the user must accept)
- Date: 2026-10-14
- Deciders: project owner
- Related: game-0001, game-0006

## Context

The simulation (fixed 1/120 s step) will run thousands of guests plus rides. The Angular + Three.js client renders it. Today telemetry flows backend -> browser over SSE at 30 Hz. We must decide who owns the truth and how state reaches the screen.

## Decision drivers

1. Single source of truth for the economy and physics (no cheating, no divergence).
2. Bandwidth and CPU on the client with thousands of guests.
3. Smooth visuals despite 30 Hz updates.
4. Fits the existing backend (modular monolith, SSE telemetry).
5. Replayability and testability (game-0006).

## Considered options

### Option A: Backend-authoritative with snapshots and deltas (recommended)
Server simulates; sends full snapshots periodically and deltas in between; client interpolates.
- Pros: one truth; client is a view; deterministic tests on the server; no client trust issues.
- Cons: server CPU scales with park size; latency on user inputs (placing a ride is a command, acknowledged by the next snapshot); bandwidth must be managed.

### Option B: Client-authoritative
Client simulates, server stores results.
- Pros: no streaming; zero latency feel.
- Cons: cheating, divergence between clients, economy cannot be trusted, contradicts the existing digital-twin backend, duplicates physics in TS.

### Option C: Hybrid (server authoritative for economy/ride state, client-predicts cosmetic motion)
- Pros: smaller streams (client extrapolates walking along paths).
- Cons: two implementations of movement that can disagree; more test surface; helps only after the simple option proves insufficient.

## Decision

Option A. Server owns all simulation; the client sends commands (build, set price, change ride speed) and receives state. Hybrid prediction is deferred until measurements justify it.

### Wire design

- Tick 120 Hz internally; publish at 30 Hz (every 4th tick), as today.
- **Keyframe** (full snapshot) every N seconds and on connect; **delta** otherwise carrying only changed components, keyed by stable entity index + generation; sequence numbers allow loss detection.
- **Interest management**: client reports viewport/area; the server sends guests within it plus rides in full. Far guests are sent at lower rate (for example 5 Hz).
- **Quantisation**: guest position as two uint16 over the park extent (park 1 km across -> ~1.5 cm resolution), heading uint8, state uint8.
- **Interpolation**: client renders ~100 ms (about three 30 Hz snapshots, tunable) in the past and linearly interpolates (slerp for rotations) between the two surrounding snapshots; extrapolation capped at one snapshot interval.
- **Transport**: binary frames. SSE carries text only, so binary would need base64 (+33% overhead; general knowledge, verify) and SSE is one-directional (fine: commands go over normal HTTP POST). WebSocket carries binary frames and is bidirectional. WebTransport is a candidate but its browser/ASP.NET support was NOT verified here. Initial pick: keep SSE for ride telemetry (existing), add a WebSocket binary channel for the park stream; decide finally in the apply-time spike.

### Bandwidth maths (guests, 30 Hz)

Per-guest record: 2 x uint16 position + uint8 heading + uint8 state = 6 bytes (index implicit by position in a keyframe; in a delta add 4-byte index = 10 bytes).

| Scenario | Calculation | Rate |
|---|---|---|
| 5,000 guests, keyframe every tick, binary | 5,000 x 6 B x 30 | 900 KB/s ~ 7.2 Mbit/s |
| Same as JSON (~60 B/guest, assumed) | 5,000 x 60 B x 30 | 9 MB/s ~ 72 Mbit/s |
| Binary delta, 10% moving per tick | 500 x 10 B x 30 | 150 KB/s ~ 1.2 Mbit/s |
| Interest managed, 300 guests in view, binary | 300 x 10 B x 30 | 90 KB/s ~ 0.72 Mbit/s |
| 1,000 guests in view, 1,000 clients each | 1,000 x 10 B x 30 x 1,000 | 300 MB/s ~ 2.4 Gbit/s server egress |

Conclusion: JSON full snapshots are not viable; binary deltas with interest management are. For a workshop (a handful of clients) all binary variants are cheap; the server-egress row shows why interest management and shared encoded frames matter if this ever scales. All JSON size figures are assumptions to be measured.

## Consequences

- Good: simple trust model; the client is a thin renderer; matches the digital-twin backend.
- Bad: server cost grows with guests; command latency; a new binary protocol to version and document.
- Neutral: the 30 Hz rate stays; sim rate (120 Hz) and publish rate are decoupled.

## Validation

- Measure: bytes/s per client at 5,000 guests with a 300-guest viewport <= 100 KB/s (target).
- Measure: p95 server publish time per frame < 2 ms at 5,000 guests.
- Visual: interpolation shows no jitter > 1 px at 60 fps render with 30 Hz input (screenshot/e2e check with Playwright).
- Loss test: dropping 5% of deltas recovers on the next keyframe with no permanent divergence.
- Decoder round-trip test: encode(decode(x)) equals x for random seeded snapshots.

## References

- docs/adr/game-0006-deterministic-parallel-simulation.md
- Gaffer On Games, "Snapshot Interpolation" / "State Synchronization" - cited from memory, NOT fetched, unverified.
- Valve Source engine networking (entity interpolation ~100 ms) - from memory, unverified.
