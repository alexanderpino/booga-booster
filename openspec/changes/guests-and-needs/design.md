# Design

## Context

See `proposal.md` for motivation. Current state that shapes the approach:

- **Guests today** are `Queue.Domain.Person` (number, name, weight 30 to 150 kg) created by `PersonGenerator` (Bogus, seedable via `QueueModuleOptions.RandomSeed`) inside `RideQueueFillerService`. Cycle count comes from `ArrivalPlanner` (4 to 8 people, scaled by `Ceiling * NiceWeather^Exponent`), parties of 1 to 5 from `PlanGroupSizes`. `RideLoadingCoordinator` (DigitalTwin) boards groups through `IRideQueueService.GetStatus/TakeGroupAsync`. Weather reaches the Queue by the `weather-updated` Dapr topic and is available in process through `IWeatherConditionProvider` (`Weather.Abstractions`, `WeatherConditionDto`: temperature, wind, sunshine, precipitation, regime, `NiceWeather`).
- **Simulation core** (from `ecs-foundation`, `scene-graph`, `job-system`, `ride-on-ecs`): module `Simulation` (`src/Simulation/FourDotnet.BoogaBooster.Simulation` + `.Abstractions`), archetype SoA storage, command buffer with a single sync point per tick, systems with declared read/write sets, fixed 1/120 s tick, canonical state hash, deterministic parallel executor. The engine rules for this change are ADRs `docs/adr/game-0001` (boundary: aggregates versus World), `game-0002` (hot toggles are fields, not add/remove; unmanaged components), `game-0003`/`game-0006` (no shared mutation, fixed-order reductions, counter-based RNG, no wall clock) and `game-0005` (backend authority, binary 30 Hz deltas, interest management, 100 ms interpolation).
- **Research** `docs/research/rct-gameplay.md` describes how a well-known park simulation behaves. Its labels (VERIFIED, INFERRED, UNKNOWN) are honoured in "Research traceability" below. This design takes mechanisms (a happiness target that moves slowly, an intensity window plus a nausea tolerance class, gates before a score, queue wait eroding mood) and replaces every number with our own, tuned for a workshop-length session and checked by the balancing harness of `park-economy`.
- **Frontend**: Angular 22 zoneless, signals, OnPush; PrimeNG 22 installed (first uses arrive with `octopus-motion` and `ride-on-ecs`); Three.js 0.169; Vitest with axe-core. The dev proxy (`proxy.conf.js`) forwards `/api` only.

## Goals / Non-Goals

**Goals:**
- Guests as ECS entities, 5 000 concurrent (10 000 stress), within a guest budget of about 1.5 ms per 8.33 ms tick.
- Deterministic, thread-count independent, replayable behaviour, pinned by a golden master.
- A teachable ruleset: every rule has a named parameter, a spec scenario and a test.
- Smooth, accessible client view of the crowd with a bandwidth the workshop network can carry.
- A reversible migration from the `Queue` arrival planner; nothing existing breaks while the mode is `LegacyQueue`.

**Non-Goals:**
- No staff entities, staff path-finding or per-guest crowding avoidance (staff are headcounts, see D14).
- No guest-to-guest interaction (collisions, conversations), vandalism, souvenirs, cash machines, benches, umbrellas, scenery appreciation (listed under "Dropped RCT features" in `docs/simulation/guests-and-needs.md`).
- No ride construction, ride ratings computation or ride breakdowns (`ride-catalog`, `coaster-editor`).
- No money ledger, loans, rating computation or awards (`park-economy`); this change only reads and exposes their inputs and outputs.
- No client-side guest simulation or prediction (ADR game-0005 option C deferred).
- No persistence of guests across restarts beyond the world save/restore of `ecs-foundation`.

## Decisions

### D1 - Legal posture: rules and facts, our own numbers, our own names

OpenRCT2 is GPL-3.0 and the game's name is a trademark. This change specifies behaviour in our own words, with numbers we chose and justify in time terms (for example "a typical guest becomes hungry after about 9 minutes"). No source code is copied or translated from OpenRCT2, no asset, sound, string or name from the game is used, and nothing in the repo or UI says "RollerCoaster Tycoon". Guest, ride, stall and thought names are original ("enjoyed the ride", "waited for ages"). Mechanisms (game rules and ideas) are not copyrightable subject matter, but a verbatim table of numbers is the closest thing to copying, which is why none of the research tables is reused as is. `docs/simulation/guests-and-needs.md` carries a "Provenance and licence" section stating this, and the apply PR description repeats it. Reviewers check the diff for copied constants (a task in group 15).

### D2 - Guest data: 11 unmanaged components, 188 bytes, fields for hot toggles

All components are `unmanaged` structs with auto-properties, registered by stable name (`guest.identity`, `guest.position`, ...). Sizes (to be asserted by a size test, budget 192 bytes):

| Component | Content | Bytes |
|---|---|---|
| `GuestIdentity` | guest number, party id, party index, party size, weight kg (30 to 150, from the legacy `Person` range), flags (has map) | 12 |
| `GuestPosition` | X, Y, Z fixed point Q16.16 metres | 12 |
| `GuestPathMotion` | from node, to node, edge progress Q16, progress per tick, wander arrivals counter | 20 |
| `GuestNeeds` | hunger, thirst, bladder, fatigue, nausea, happiness, happiness target, unhappy-seconds counter (all `ushort`, 0 to 10 000) | 16 |
| `GuestWallet` | cash in cents (`int`) | 4 |
| `GuestPreferences` | intensity min and max (hundredths), queue tolerance s, thrift permille, nausea class, fatigue scale, thrill class | 12 |
| `GuestIntent` | kind, leave reason, target entity id, target node, started tick | 20 |
| `GuestQueueState` | ticket, rank, enqueued tick, boarded cycle id, wait seconds | 16 |
| `GuestRideHistory` | 128-bit ridden mask by ride ordinal, last ride ordinal, rides count | 20 |
| `GuestActivity` | held item kind and timer, idle seconds, last thought tick | 8 |
| `GuestThoughts` | 4 x (kind, subject entity index, tick) as an inline array | 48 |
| **Total** | | **188** |

Hot toggles (intent kind, queuing) are fields: ADR game-0002 rule 5 forbids add/remove for them, and 5 000 guests changing intent every few seconds would otherwise churn archetypes. Singleton inputs and outputs (`ParkRating`, `ParkPricing`, `SpawnModifiers`, `StaffLevels`, `WeatherState`, `ParkMetrics`, `GuestCounters`, `RevenueAccumulator`) are components on one dedicated `ParkEntity` (the singleton pattern of ADR game-0001), so they are in the canonical hash and in save/restore. Litter items are entities with `LitterItem` and `GuestPosition`-like `LitterPosition`. Names are never stored: `GuestNameSource.NameFor(guestNumber)` is a pure function (seeded Bogus-like word tables; the Queue module's Bogus use is the precedent; the choice of library for names is a currency-check item, or own small tables).

*Alternative:* store `Person`-like objects in a managed list. Rejected: contradicts ADR game-0001/0002 and cannot be hashed or parallelised.

### D3 - Scales and units: integers everywhere that decides

Needs and mood 0 to 10 000 (`ushort`, centi-points; the UI shows percent), ratings in hundredths (the ride-catalog boundary unit, kept so numbers stay comparable with the research), money in cents (`int` for guest cash, `long` in the ledger), positions Q16.16 metres (`int`; range plus or minus 32 768 m, resolution 15 micrometres; speed 1.4 m/s is 764.6 raw units per tick, rounded once when the trait is sampled, then exact), path distances unsigned millimetres. No `float` or `double` feeds a decision. Rationale: integer arithmetic is bit-identical on every CPU, which removes the cross-machine risk ADR game-0006 accepts for physics. Doubles appear only in client projections.

### D4 - Time: ticks, cohorts and strides

The tick stays 1/120 s. Heavy per-guest logic runs in cohorts keyed by `(tick + entity index) mod stride`, which spreads cost evenly and is thread-count independent because it depends only on the entity index and the tick:

| Work | Stride | Why |
|---|---|---|
| Movement | every tick | smooth, exact fixed-point stepping, 5 000 x about 40 ns |
| Needs, mood, thresholds, leave rules | 120 ticks (1 s) | rates are defined per simulated second |
| Urgent-need check | 240 ticks (2 s) | reaction time of a few seconds is plausible and cheap |
| Ride decision | 480 ticks (4 s) | at 25 % trigger this is one search per guest per 16 s |
| Queue ranking and statistics | 30 ticks (0.25 s) | shuffling forward looks timely |
| Park metrics | 1 440 ticks (12 s) | the cadence at which the rating refreshes |
| Stream publish and projection | 4 ticks (30 Hz) | ADR game-0005 |
| Weather sample | 120 ticks (1 s) | `IWeatherConditionProvider` is not read per tick |

Mapping from the research's 40 Hz clock: durations are re-expressed in seconds, not ticks, so the 120 Hz tick needs no conversion factor (the arrival rate section keeps its per-second definition).

### D5 - Path-finding: integer flow fields, not per-guest A*

| | Flow fields per destination (chosen) | A* per guest |
|---|---|---|
| Work for 5 000 guests, 60 destinations, 20 000 nodes | 60 builds of about 20 000 expansions, redone only when the graph changes; stepping a guest is O(1) | a search per route; at one new route per guest per 16 s that is about 310 searches per second, each 100 to 500 expansions typical but unbounded worst case |
| Nearest facility by walking distance | one multi-source field per kind: the question "which toilet is nearest" is answered for every node at once | needs a search per facility or a reverse search |
| Determinism | distances are integers; ties by node index; one thread per field, no shared state | also fine with integer costs and (f, g, index) ordering, but result order depends on request order unless serialised |
| Budgeting | explicit expansions-per-tick budget, resumable | budget per search plus a request queue, with per-guest starvation to manage |
| Dynamic changes | rebuild on graph version, old field stays valid until the new one completes | per-guest re-plan |
| Weakness | memory 8 bytes per node per field (9.6 MB at 60 x 20 000); no per-guest cost (crowding) | per-guest preferences (avoid queues, scenery) are easy |

Chosen: flow fields. The number of destination classes (about 60) is far smaller than the number of guests, which is the break-even condition. Personal routing preferences are not needed: guest differences are in goals, not in routes. A* remains the documented fallback if per-guest costs (crowd avoidance) are added later.

Details: Dijkstra over the CSR path graph of `park-world-grid`; the heap is a binary heap keyed `(distance, node index)` stored in a pooled array; builds are objects with resumable state, advanced by one job per build per tick with a share of `FlowFieldExpansionsPerTick` (default 16 000) distributed in creation order (first builds get the remainder); the completed field is swapped in at the tick boundary. A guest at a node steps to the neighbour with the smallest `(distance, node index)`; mid-edge it finishes the edge first. Wandering uses a counter-based pick. Queue-path edges are masked out of all fields except the owning ride's; exit edges are one-way (the ride contract).

Scale justification for 5 000: the ADR game-0005 bandwidth maths and ADR game-0001 validation both use 5 000; the roadmap demo is "5 000 guests streaming"; a park of 40 to 50 attractions with a per-attraction capacity contribution of the order of 100 (assumption, `ride-catalog`) has a suggested capacity of 4 000 to 5 000, so the arrival process saturates there. 10 000 is a 2x stress margin. The research states a very large guest count threshold in the spawn code (more than 52 000, VERIFIED) that is irrelevant at our scale. Neither number is a fact about the original game.

### D6 - Spatial structures: rebuilt hashes, not stored indices

Two derived structures, both rebuilt from components and never serialised: the litter hash (8 m cells, rebuilt every 30 ticks, used by the needs cohort) and the stream hash (16 m cells over guests, rebuilt each published frame for interest queries). Both are counting-sort buckets over entity-index order, so their contents are deterministic and independent of thread count. `park-world-grid`'s tile spatial index is not used for guests because guests move every tick and the grid index is built for tiles.

### D7 - Queue model: tickets and periodic ranking, no stored queue lists

A queue is not a list in storage. Each ride entity holds `RideQueueStats { nextTicket, length, longestWaitSeconds }`; each queuing guest holds a ticket. Every 30 ticks a serial pass gathers queuing guests per ride in entity-index order, sorts each bucket by ticket (unique, so the sort is total and stable-free) and writes ranks and statistics. Reason: lists of `EntityId` are variable length and cannot be unmanaged components; side tables would need their own hashing and serialisation; tickets keep all state in hashable components. Cost: 5 000 guests, the pass touches only queuing guests (a few hundred typically). Party contiguity: tickets for a party joining in one tick are consecutive by party index; the join is processed in `(party id, party index)` order. Slots: the guest walks to `queueSlot[min(rank, slots - 1)]` of the ride's queue path (`IParkLayout.QueueSlots(ride)`); a full queue is "all slots occupied or `MaxQueueLength` reached" and is the analogue of the legacy `MaxQueueLength` of 500 (the legacy default is not reproduced because a physical queue path bounds the line).

### D8 - Ride boarding handshake and the `IRideQueueService` adapter

The ride side (catalog, Octopus on ECS) exposes a `RideBoardingPort` component: `SeatsOffered`, `SeatUnit`, `OfferCycleId`, `CompletedCycleId`. The ride writes an offer when it is ready to load; the boarding step takes whole parties in rank order that fit (look-ahead window of 10 parties, matching the legacy "ride is full when no group in the look-ahead window fits" idea), marks guests riding, deducts the price and reduces `SeatsOffered`. When the ride increments `CompletedCycleId`, every guest that boarded that cycle gets its outcome (D10) and walks to the exit node. A counter handshake avoids an event buffer between ride and guests and is replay-safe.

The Octopus is loaded by `RideLoadingCoordinator` through `IRideQueueService` (specs `ride-loading`, `gondola-boarding`: pairs, odd member alone, split of parties above 32). Those rules stay in DigitalTwin. In `Park` mode the simulation registers `EcsRideQueueService : IRideQueueService` (replacing `RideQueueService` in DI; both live behind the same abstraction, `Simulation` references `Queue.Abstractions` only): `GetStatus` reads the published projection (ranked groups with person number, generated name, weight), `TakeGroupAsync` enqueues a take command applied at the next tick boundary (at most 8.3 ms latency) and completes when applied, returning null if the group is gone, `EnqueueGroupAsync` spawns a party at the queue start. A group taken this way is boarded by the same internal routine as the generic handshake (price, ticket release, riding state), so there is one boarding implementation with two front doors. The existing Queue service contract tests are made generic and run against both implementations.

*Alternative:* migrate Octopus loading fully into ECS now. Rejected: `ride-on-ecs` deliberately keeps the coordinator; moving it is a later cleanup and would break "existing specs keep passing unchanged".

### D9 - Decision-making: gates, then a linear integer score

Gates (hard, in the order they cheaply exclude): status, queue full, reachable and within 60 m walking (or any with a map), not the previous ride, price against cash, price against value and thrift, weather, intensity window with happiness slack, nausea limit with slack, expected wait against queue tolerance. Score = `2 * excitement + novelty (150) - 4 * walking metres - 2 * expected wait seconds`, ties by ride ordinal. Why linear: it is explainable on a slide and every term has a unit and a magnitude (a ride with excitement 6.00 is worth 1 200 points, 100 m of walking costs 400, a 60 s wait costs 120). The research shows the original picks by excitement alone with gates; the distance and wait terms are our addition so that crowds spread across rides instead of all heading for the best one (a testable emergent behaviour: the spread test in group 8 runs 200 sequential decisions with two identical rides at 20 m and 50 m walking distance, where the 30 m difference is worth 120 points, i.e. 60 s of expected wait; it asserts that the nearer ride receives the first choices and the farther ride at least 20 % of all choices once the nearer queue's expected wait exceeds 60 s).

Price and value: a ride is acceptable if `price <= thrift ratio * value` with the ratio a trait between 1.2 and 2.0; satisfaction (D10) rewards price at most half the value and punishes price above value. Value is supplied by the catalog (an assumption, see contracts): this design does not define how value is computed from ratings.

### D10 - Ride outcome table and mood arithmetic

The satisfaction sum is the table in spec `guest-ride-queueing` (intensity fit, nausea fit, value, novelty, wait), clamped to [-4 000, +5 000]. Visible happiness snaps to the target on exit (the one place the visible value jumps), which makes "was it a good ride?" visible immediately in the UI. Nausea gain is `ride nausea (hundredths) * 10 * class factor / 100`. All numbers sit in `GuestParameters` with an XML doc line giving the intended in-game reading ("about 9 minutes to hungry").

The feature request `docs/feature-requests/passenger-happiness.md` is reconciled as follows (decision, not a silent change): a single preferred intensity is the window centre; the five-minute queue wait with exponential decay is the queue tolerance default (300 s) with an exponential drain table (D4 in the doc); nausea is driven by ride nausea and tolerance class (the research's mechanism) rather than by the gap between preferred and actual intensity; "25 percent nausea per second at maximum G" is not implemented here because it needs the live G stream of a specific ride and belongs to the Octopus ride definition in `ride-catalog` (it can write an extra nausea contribution through the same ride outcome path; recorded in Risks). The four-bar Rider Experience panel is satisfied by the summary and per-ride aggregates (average happiness of queuing guests and of riders per ride) exposed by the projection; the panel itself is a follow-up if the user still wants it in the dashboard.

### D11 - Facilities, items and weather

Facilities are ride-catalog entries without ratings and with a `FacilityKind` and an item (price, value, weather class). Choice: nearest by walking distance (flow field), no queue for shops (a guest buys on arrival; stall throughput is not modelled), consumption timers in `GuestActivity`. Weather enters in three places only: precipitation gate for unsheltered rides and open-air stalls, heat factor on thirst, cold gate for cold items (thresholds 22/30 degrees and 8 degrees are our numbers). The `NiceWeather` indicator drives only the arrival rate, as it does today, so the existing Weather integration keeps its meaning.

### D12 - Spawn: the replacement for the arrival planner

Rate = `3.0 * ratingFactor * feeFactor * weatherFactor * capacityFactor * (1 + award/1000) + marketing`, capped at 60. 3.0 guests per second at maximum rating with an average visit of about 30 to 40 minutes (fatigue 1 000 to 9 000 at 4 per second = 2 000 s, scaled 70 to 140 %) gives a steady state of roughly 3 x 2 000 = 6 000, which matches the 6 000 cap; a poor park therefore settles far below the cap, which is the intended economy feedback. These are starting values for the balancing harness, not facts. The draw is Bernoulli per tick on the party process (`rate / (3 * 120)`), which is exact for rates below 360 parties per second, uses one counter-based draw, and needs no accumulator state. Weather is read once a second through `IWeatherConditionProvider` and logged as an input (not a second Dapr subscription: two minimal-API endpoints with `WithTopic("weather-updated")` in one app is unverified Dapr behaviour and the in-process provider is simpler and already exists). The entrance fee is charged at the gate; a guest whose cash is below the fee is turned away. Fee default 1 000 cents is an own number.

Migration steps (spec `guest-spawning`): (1) mode switch and options validation; (2) `Queue:Filler:Enabled`; (3) spawner and gate; (4) `EcsRideQueueService`; (5) docs and rollback. Until step 4 lands, `Park` mode must not be enabled outside tests; the options validator rejects `Park` when no `IRideQueueService` implementation from the simulation is registered.

### D13 - Thoughts, mood feed and litter

Thoughts are enum codes plus an optional subject index: no text in the simulation (localisation and label tables live in `Abstractions` DTO mapping). Feed: a bounded ring of 200 entries in a derived, non-hashed structure filled during the projection publish (so it is a pure function of the thought rings), rate-limited deterministically. Litter: items are entities; containers produce litter with probability 50 % (our number), vomit from nausea. Effects: a happiness-target term per needs update using the litter hash, and the old-litter count (older than 180 s) in `ParkMetrics`. No guest "sees" litter outside 8 m. Items are capped at 3 000 so a runaway park cannot exhaust memory.

### D14 - Staff are levels, not entities (workshop simplification)

`StaffLevels { handymen, mechanics, security, entertainers }` is a singleton written by `park-economy`. Only the handyman count acts in this change (cleaning throughput 6 items per handyman per minute, oldest first). Mechanics belong to ride reliability (`ride-catalog`), entertainers and security are reserved for later. No staff path-finding or patrol areas. This keeps the thousands-of-guests cost model clean and removes a second agent type from the first guest change; it is listed under Non-Goals and in the doc as a deliberate simplification.

### D15 - Output events to `park-economy`

Per ADR game-0001 the World does not read aggregates and aggregates are fed by events. Per-guest money events at 5 000 guests would be too chatty, so guest payments are accumulated in singleton counters updated by a fixed-order reduction: `RevenueAccumulator { entrance, rides, shops by category, counts }`, `RideTraffic` per ride, `GuestCounters { arrived, left by reason, turned away, lost }`. `park-economy` reads and resets them at week boundaries through a logged command. One small event stream remains for the outside world: `group-queued` (existing topic) published after the tick through `IIntegrationEventPublisher` from a bounded channel so a slow broker never blocks the tick.

### D16 - State stream: encoder system, WebSocket, tiers

The encoder is a read-only system that runs every fourth tick after all writers, but encodes only for connected clients (zero cost otherwise). Per client it holds a last-sent table (quantised records per entity index) to build deltas, the client's view and tier cadence counters. Work shared between clients (the quantised record for every guest, once per frame, in an array indexed by entity index) is computed once; per-client work is selecting and copying records, which is why 20 clients cost little (spec `guest-performance-budget`). Back-pressure: one pending frame per socket, latest wins, next frame a keyframe. Record layout, header, tiers and numbers are in spec `guest-state-stream`; the bandwidth arithmetic (30 x 300 + 10 x 700 + 5 x 4 000 records per second x 10 bytes = 360 KB/s) improves on the ADR's full-rate figure (5 000 x 10 x 30 = 1.5 MB/s) by distance tiers, which is the design's answer to "most walking guests change every frame, so deltas alone do not save bandwidth".

Transport: WebSocket (built into ASP.NET Core and every browser; carries binary frames; the client sends view messages). The ADR defers the final pick to an apply-time spike: the spike (task group 11) measures SSE with base64 against WebSocket on the same frames and records the choice; WebTransport is out because its ASP.NET Core and browser support was not verified. Dev proxy: `proxy.conf.js` needs a `ws: true` entry (to be verified against the dev-server proxy documentation at apply time).

### D17 - Crowd rendering

Instancing with three level-of-detail meshes (about 120, 24 and 2 triangles), per-instance colour for the mood tier, updates in place into preallocated typed arrays, one `InstancedMesh` per level (at most 6 draw calls including a thought-bubble sprite batch and the selection ring). Interpolation per ADR game-0005: 100 ms behind, ring of 6 frames, extrapolation capped at 33 ms (one 30 Hz interval). Height comes from the terrain height field the client already has from `park-world-grid`. Buffer-update API specifics of Three.js 0.169 (partial update ranges) are verified at apply time against the pinned version. Procedural meshes now; glTF crowd models from `asset-pipeline` may replace them later through the same instancing seam. Walking bob is a vertex-shader or matrix animation driven by a per-instance phase, disabled under reduced motion.

### D18 - Inspection: projection, not tick-gated reads

`ecs-foundation` gives snapshot handlers a between-ticks gate that may wait one tick. Inspection polling by many clients must not stall the loop, so a projection system builds an immutable `GuestProjection` every fourth tick (detail for all guests in an array, ride-level aggregates, feed ring) and publishes it with a volatile swap. Cost: 5 000 guests x about 64 bytes copied 30 times a second is 9.6 MB/s of memory traffic. Endpoints (`Endpoints/` only parse and dispatch to query handlers deriving from `QueryHandler<,>`): summary, guest by id, nearby guests, mood feed. PrimeNG components are chosen after consulting the `primeng` MCP server (candidates: `p-panel` or `p-card` for the container, `p-progressbar` for needs, `p-table` for the nearby list and thoughts, `p-tag` for mood and intent, `p-drawer` for narrow screens); a11y requirements are in the spec.

### D19 - Determinism machinery

`CounterRng` is a small stateless 64-bit mixer (own code, a SplitMix64-style finaliser, about 10 lines, documented, with known-answer tests); `IGuestSampler` is the seam; stream ids are constants in one file with a uniqueness test. Reductions: per-chunk partial sums in an array indexed by chunk, combined by one thread in ascending order (ADR game-0006 rule 2). The thread-count independence test and the golden master are specified in `guest-simulation-determinism`. Flow-field builds are self-contained objects so running them on any worker yields the same arrays. One caveat is recorded in Risks: save and restore while a field build is in progress.

### D20 - System graph (declared access)

| System | Reads | Writes | Notes |
|---|---|---|---|
| `guest.inputs` | input log queue | `WeatherState`, `ParkPricing`, ride price/status commands | applies logged inputs at tick start |
| `guest.flowfields` | path graph (resource) | field cache (resource) | one job per build, budgeted |
| `guest.spawn` | `ParkRating`, `ParkPricing`, `SpawnModifiers`, `WeatherState`, `ParkMetrics`, `GuestCounters` | commands, `GuestCounters`, `RevenueAccumulator` | serial |
| `guest.needs` | `GuestPosition`, litter hash, `WeatherState` | `GuestNeeds`, `GuestIntent`, `GuestActivity`, `GuestThoughts` | cohort 120, parallel over chunks |
| `guest.decide` | `GuestNeeds`, `GuestPreferences`, `GuestWallet`, `GuestRideHistory`, ride components, fields | `GuestIntent`, `GuestPathMotion`, `GuestThoughts` | cohort 480, parallel |
| `guest.move` | fields, `GuestIntent` | `GuestPathMotion`, `GuestPosition`, `GuestIntent.arrived` bit | every tick, parallel |
| `guest.arrive` | `GuestIntent`, ride components | queue join, facility use, exit commands | serial, ordered by entity index |
| `guest.queue` | `GuestQueueState`, ride components | `GuestQueueState`, `RideQueueStats` | every 30 ticks, serial |
| `guest.board` | `RideBoardingPort`, `GuestQueueState`, `GuestWallet` | `GuestIntent`, `GuestWallet`, `RideTraffic`, `RevenueAccumulator`, `RideBoardingPort` | serial |
| `guest.outcome` | `RideBoardingPort`, `GuestRideHistory` | `GuestNeeds`, `GuestIntent`, `GuestRideHistory` | per completed cycle |
| `guest.litter` | `LitterItem`, `StaffLevels` | `LitterItem` (commands), litter hash | hash every 30 ticks, cleanup per minute |
| `guest.metrics` | all guest components (read) | `ParkMetrics` | every 1 440 ticks, fixed-order reduction |
| `guest.project` | all guest components (read) | projection (outside the world) | every 4 ticks |
| `guest.stream` | positions, intent | per-client frames (outside the world) | every 4 ticks, only if clients |

The job-system derives edges from these sets; the serial systems are the parts a later change may split.

## Dependencies and contracts

**Requires from `ecs-foundation`:** `World`, component registration with stable names and `unmanaged` structs, singleton pattern via a dedicated entity, `EntityCommandBuffer` (spawn and despawn at the single sync point), `ISimulationSystem`/`SystemAccess`/`SystemContext`, chunk queries with `ChunkCount/GetChunk(i)`, `IEntityLifecycleObserver` (release tickets on destroy), `FixedTickDriver`, canonical hash and `WorldSerializer`, `SimulationParameters`, diagnostics metrics, the between-ticks gate, test harness `RandomOperationSequence`, benchmark project. **Naming note:** the `job-system` and `ride-on-ecs` artifacts refer to the module as `World`; the module is `Simulation` (`FourDotnet.BoogaBooster.Simulation`), and this change uses `Simulation` throughout. Reconcile at apply time; the semantics are unchanged.

**Requires from `job-system`:** `IJobScheduler`, range jobs over chunks, the system-graph builder (access sets above), the timeline provider for the performance doc, the thread-count-independence harness (reused for the guest scenario), checked mode that verifies declared access.

**Requires from `scene-graph`:** nothing for guests (guests are not scene nodes; a flat position is cheaper and ADR game-0004 limits the graph to hierarchies). Ride scene nodes provide ride transforms to the client mirror; guests only need ride entrance and exit node positions from the grid.

**Requires from `ride-on-ecs`:** the Octopus as entities with `RideBoardingPort` semantics reachable from `RideLoadingCoordinator` (intent: the coordinator keeps calling `IRideQueueService`), `ISimulationGateway` for commands from DDD modules into the world and the event drain. The binary snapshot spike result (framing helper, quantisation) is reused by the stream encoder if it exists; if not, the encoder is written here.

**Requires from `park-world-grid` (intent, names to reconcile):** `ParkGrid` and a derived `PathGraph` in compressed-row form (node count, offsets, targets, edge length in millimetres, edge flags: normal, queue-of-ride, exit-only), node positions and terrain height, `GraphVersion`, `EntranceNodes`, `ExitNode`, per-ride `QueueSlots` (ordered nodes), tile size in metres, park extent for quantisation, terrain height sampling for the client. Exposed to systems through a small read-only service `IParkLayout` in `Simulation.Abstractions`. Graph edits apply at tick boundaries as logged inputs.

**Requires from `ride-catalog` (intent, names to reconcile):** per ride entity: `RideIdentity` (`RideId` Guid, stable ordinal 0 to 127), `RideStatus` (open, closed, broken, testing), `RideRatings` (excitement, intensity, nausea in hundredths), `RidePrice` (cents), `RideValue` (cents), `RideLayout` (entrance node, exit node, queue slots, sheltered flag), `RideTiming` (cycle seconds, riders per cycle, seat unit), `RideBoardingPort`, `GuestCapacityContribution`; for facilities `FacilityKind` and `FacilityItem` (price, value, weather class). The legacy Octopus queue id `11111111-1111-1111-1111-111111111111` must resolve to the Octopus ride entity so existing callers keep working. Ratings for the Octopus come from the digital twin (peak G mapping, research section 15.2), computed by the catalog, not here.

**Provides to `park-economy`:** singleton components `ParkMetrics` (guests in park, happy share, lost guests, old litter count, value for money, suggested capacity, average queue seconds, uptime inputs from rides), `GuestCounters`, `RevenueAccumulator`, per-ride `RideTraffic`; writable inputs `ParkRating`, `ParkPricing` (entrance fee), `SpawnModifiers` (award permille, marketing guests per second), `StaffLevels`; the logged read-and-reset command for accumulators; `ParkScenarioBuilder` (headless world with grid, rides, seeded guests) for the balancing harness; the golden-master scenario list; the thread-count harness extension.

**Provides to `coaster-editor`:** any ride entity with the boarding port is usable by guests unchanged.

**Provides to the client:** the stream endpoint, the `GuestFrameDecoder`, `CrowdRenderer`, inspection endpoints and DTOs (`GuestInspectionResponse`, `ParkGuestSummaryResponse`, `NearbyGuestsResponse`, `GuestFeedResponse`).

**HTTP and WebSocket surface:** `GET /park/guests/stream` (WebSocket), `GET /park/guests/summary`, `GET /park/guests/{index}/{generation}`, `GET /park/guests/nearby`, `GET /park/guests/feed`.

**Configuration:** `Simulation:Guests:Mode` (`LegacyQueue` | `Park`), `Simulation:Guests:MaxGuests`, `Simulation:Guests:Seed`, `Simulation:Guests:FlowFieldExpansionsPerTick`, `Queue:Filler:Enabled`.

## Research traceability

| Mechanism used | Research section and label | What this design does with it |
|---|---|---|
| Spawn rises with rating, falls with fee above value for money and with weather/capacity | s3 VERIFIED formula, s16.3 INFERRED mapping | Own formula and numbers (D12); weather retained as modifier as the research recommends (s15.4, INFERRED) |
| Happiness target moves slowly, visible follows | s4 VERIFIED, s17 | Mechanism only, own rates (D10) |
| Intensity window, nausea tolerance classes, value gate | s5.2 VERIFIED | Gate structure kept; windows, limits and ratios are our numbers |
| Satisfaction lookup after a ride | s5.3 VERIFIED | Structure kept; table is ours (spec `guest-ride-queueing`) |
| Queue wait erodes mood | s5.5 VERIFIED, units INFERRED | Native seconds; exponential per the feature request |
| Ride value formula, ride age factor | s6.5 "first-pass read, re-verify" | NOT used; value is supplied by the catalog (apply-time task: confirm the catalog's value definition with the user) |
| Default entrance fee | s3 INFERRED | Not used; our default 10.00 |
| Leave rules | s4 VERIFIED | Own thresholds and a lost-guest rule |
| Cash machine, souvenirs, vandalism, umbrellas | s4, s10 | Dropped (workshop simplification, s17) |
| Pathfinding budgets and path rules | s10 secondary (wiki, "(i expect)") | Not relied on |
| UNKNOWN items (climate tables, scenario defaults, hiring cost) | s18 | Not needed; where a number is needed it is ours and listed as an assumption |

Apply-time verification task: none of the research's "re-verify" numbers is a dependency of this change; if a later calibration against the original is wanted, re-verify the cited lines at the pinned commit first.

## Risks / open questions

- **Contracts from `park-world-grid` and `ride-catalog` are intent only** (both are scaffolded but empty). Names, component shapes and whether ride value is in the catalog are assumptions; the first apply task reconciles them and records differences in the PR. If `ride-catalog` defines a different boarding mechanism, only `guest.board` and the adapter change.
- **Dapr duplicate subscription.** Reading weather through `IWeatherConditionProvider` avoids a second `WithTopic("weather-updated")` endpoint; whether Dapr tolerates two endpoints for one topic in one app is unverified and deliberately not relied on.
- **Save and restore during a flow-field build.** Field cache is derived state and excluded from the hash, but build progress affects behaviour (guests wander until their field exists). Mitigation: save refuses while builds are pending and replays use input logs, not mid-build saves. An alternative (serialise builder state) is deferred.
- **Between-ticks gate versus projection.** Reading through the projection can show state up to 12 ticks old; acceptable for UI, not for tests (tests read the world directly).
- **Bandwidth when guests are all moving.** Most walking guests change every frame; the tier design relies on a camera that is not showing the whole park at full rate. A "whole park at 30 Hz" request would cost 1.5 MB/s (ADR arithmetic). The server may cap the near tier radius; documented knob.
- **Octopus-specific "25 percent nausea per second at maximum G"** from the feature request is not implemented here; it needs the live G stream and belongs to the Octopus ride definition. Decision recorded in D10; user may want it as a follow-up.
- **Group behaviour simplification.** Members move in lockstep and detach on individual urgency; no re-joining. Real social behaviour is out of scope.
- **Memory budget is tight (188 of 192 bytes).** If a later change adds guest data, the budget or the thought history (48 bytes) must be revisited.
- **Balance is unproven.** Spawn rate, need rates and prices are starting points; `park-economy`'s balancing harness is the acceptance mechanism. Until then expect tuning commits that update the golden master deliberately.
- **Currency checks at apply time** (nothing asserted here): BenchmarkDotNet (already a candidate from `ecs-foundation`), a name-generation source (Bogus is already pinned by the repo, so it passes the existing-pin test; a smaller own table is the alternative), client-side test helpers. WebSocket APIs are platform features; Three.js buffer-update APIs are checked against the pinned 0.169 documentation. No new runtime dependency is planned.
- **Style-guide check.** Folder `Guests/` inside `Simulation`, and whether `GuestsOptions` lives with the module options, are confirmed with the `4dotnet-csharp-style-guide` MCP server in task 1.2.
- **AI tooling sync.** CLAUDE.md status rows change (module description, new endpoints); `.github/copilot-instructions.md` mirror and the `ai-tooling-sync` check are part of the final group.
- **Open question for the user:** is Park mode intended to replace the legacy queue permanently once stable (then the `Queue` filler, planner and weather multiplier become removal candidates in a later change), or must `LegacyQueue` stay as a supported teaching baseline? Assumed: stays supported.

## Migration Plan

1. Land with `Mode=LegacyQueue` (default): nothing observable changes; new endpoints return an empty park.
2. Enable `Park` in development with a scenario file (rides and paths from `park-world-grid`/`ride-catalog` fixtures) and the stress scenario for demos.
3. Rollback: set `Mode=LegacyQueue` and restart; the filler resumes. No persisted data depends on guests, so no data migration exists.
4. Removal of the legacy filler is a separate future change after the user decides (open question).
