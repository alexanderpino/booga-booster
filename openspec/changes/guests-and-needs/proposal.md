# Proposal

## Why

After `ride-on-ecs`, `park-world-grid` and `ride-catalog` the park has rides, paths and a catalog, but nobody walks in it. Today "guests" are `Person` records (number, name, weight) that the `Queue` module drops into one ride's queue on a weather-scaled timer; they have no position, no needs, no choices and no life outside the queue. The tycoon loop (more rides and better prices produce happier guests, who produce more income) needs thousands of simulated guests that choose, queue, ride, eat, litter and leave, and the client has to show them smoothly. This is the heaviest per-tick workload of the roadmap, so it is also the first honest test of the ECS, the job system and the state-sync design (ADRs `docs/adr/game-0001-ecs-for-simulation-core.md`, `game-0002-data-oriented-component-storage.md`, `game-0003-work-stealing-job-system-without-fibers.md`, `game-0005-simulation-authority-and-state-sync.md`, `game-0006-deterministic-parallel-simulation.md`). The mechanics are informed by the sourced research in `docs/research/rct-gameplay.md`; the rules are re-specified in our own words with our own numbers.

## What Changes

- **Guests become ECS entities** in the `Simulation` module (`Guests/` folder): structure-of-arrays components for position, path motion, needs (hunger, thirst, bladder, fatigue, nausea, happiness with a happiness target), wallet, preferences (intensity window, nausea tolerance, queue tolerance, thrift), intent and current target, party membership, queue ticket, and a four-slot thought history. Traits come from a seeded `IGuestSampler` (the `IPersonGenerator` pattern: pure function of seed and guest number). Names are derived on demand, never stored.
- **Spawn process** replaces the `Queue` arrival planner when the new switch `Simulation:Guests:Mode` is `Park` (default stays `LegacyQueue`): arrivals per second are driven by park rating, entrance fee against value for money, weather (`NiceWeather`, same `Ceiling * nice^exponent` shape as today), park capacity, awards and marketing. Guests arrive as parties of 1 to 5, as today. Migration is defined step by step; the `weather-updated` and `group-queued` integration events keep working in both modes.
- **Path-finding on the footpath graph** with deterministic integer **flow fields** (one Dijkstra distance field per destination class: each ride entrance, nearest food, drink, toilet, first aid, information kiosk, park exit), cached by graph version, computed incrementally under a per-tick node-expansion budget scheduled on the job system. Target: 5 000 concurrent guests (stress ceiling 10 000), justified in `design.md`.
- **Decision making**: ride choice heuristic (excitement, novelty, distance, expected wait, price against value) behind hard gates (open, affordable, weather, intensity window, nausea tolerance, queue tolerance, not the previous ride); facility use (food, drink, toilet, first aid, information kiosk); a rule set for leaving the park.
- **Queueing and boarding handoff**: ticket-ordered queues with periodic ranking, party contiguity, waiting-time mood penalty (exponential after the guest's tolerance, as `docs/feature-requests/passenger-happiness.md` asks), abandonment, a `RideBoardingPort` handshake with the ride (generic boarding plus an `IRideQueueService` adapter so `RideLoadingCoordinator` and its specs keep working unchanged), payment on boarding, and ride-outcome satisfaction.
- **Thoughts, litter and vomit**: bounded thought history per guest, a global mood feed, litter from empty containers, vomit from nausea, surrounding-cleanliness mood effects, and a throughput hook for cleaning staff supplied later by `park-economy`.
- **Guest state stream** to the browser per ADR game-0005: binary quantised keyframes and deltas at 30 Hz over a WebSocket channel, viewport interest management, distance tiers (30, 10, 5 Hz), sequence numbers, loss recovery, a measured bandwidth budget.
- **Crowd rendering** in Three.js: instanced meshes, three levels of detail, snapshot interpolation 100 ms in the past, frame-time and draw-call budgets, a bandwidth meter.
- **Guest inspection UI** with PrimeNG (first full PrimeNG feature after the arm panel and timings dashboard): select a guest from the canvas or from an accessible list, see needs, wallet, preferences, intent, thoughts; park-wide guest summary and mood feed. Axe-core a11y specs.
- **Determinism and performance**: thread-count-independence and golden-master tests with seeded samplers, replay from an input log, and numeric per-tick budgets with benchmarks.
- **BREAKING (configuration only, off by default)**: in `Park` mode the `Queue` background filler is disabled by the new option `Queue:Filler:Enabled=false`, and `IRideQueueService` is served by the simulation. Default behaviour, endpoints and all published specs are unchanged.

## Capabilities

### New Capabilities
- `guest-entities`: Guest components, identity, seeded trait sampling, party membership, caps and despawn.
- `guest-spawning`: The arrival process driven by rating, fee, weather, capacity, awards and marketing, and the migration from the `Queue` arrival planner.
- `guest-pathfinding`: Deterministic budgeted flow-field path-finding on the footpath graph, queue-path rules and cache invalidation.
- `guest-needs-and-mood`: Need dynamics, the happiness-target mechanic, nausea, tolerance classes and thresholds.
- `guest-ride-choice`: Gates and scoring that decide which ride a guest walks to.
- `guest-facilities-and-exit`: Shop, drink, toilet, first aid and kiosk use, purchases, and the rules for leaving the park.
- `guest-ride-queueing`: Queue tickets, ranking, tolerance, boarding handoff, payment, ride outcome, and compatibility with the `Queue` module contract.
- `guest-thoughts-and-litter`: Thoughts, mood feed, litter, vomit, cleanliness effects and cleaning throughput.
- `guest-state-stream`: The binary keyframe/delta wire protocol, interest management, tiers and bandwidth budget.
- `guest-crowd-rendering`: Instanced, LOD-ed, interpolated crowd rendering and its frame budgets.
- `guest-inspection-ui`: The PrimeNG guest inspection panel, park summary, mood feed and their accessibility.
- `guest-simulation-determinism`: Seeded samplers, thread-count independence, golden masters and replay.
- `guest-performance-budget`: Per-tick, memory and bandwidth budgets, benchmarks and the 5 000-guest stress scenario.

### Modified Capabilities
- `weather-driven-queue-fill`: adds a requirement that the background filler can be switched off by configuration so the simulation can own arrivals.

## Impact

- **New code**: `src/Simulation/FourDotnet.BoogaBooster.Simulation/Guests/**` (components, systems, flow fields, sampler, stream encoder, adapter), `Simulation.Abstractions` DTOs under `DataTransferObjects/Guests*/`, `src/Tests/FourDotnet.BoogaBooster.Simulation.Tests/Guests/**`, benchmarks under `src/Benchmarks/FourDotnet.BoogaBooster.Simulation.Benchmarks/Guests/`, `src/FourDotnet.BoogaBooster.App/src/app/park/guests/**` (stream client, crowd renderer, inspection panel), `docs/simulation/guests-and-needs.md`.
- **Edited code (small)**: `Queue` module gains `QueueModuleOptions.FillerEnabled` and a guard in `RideQueueFillerService` (no other Queue behaviour changes); `Simulation` composition gains guest systems and a `GuestsOptions` section; `Api` host maps the guest stream and inspection endpoints through `MapSimulationEndpoints()`; `proxy.conf.js` gains a WebSocket entry for the stream; `CLAUDE.md` and `.github/copilot-instructions.md` (status and module rows; `ai-tooling-sync` check).
- **Reads from other modules** through `.Abstractions` only: `Weather.Abstractions` (`IWeatherConditionProvider`), `Queue.Abstractions` (`IRideQueueService`, DTOs; implemented by the simulation in `Park` mode), `IntegrationMessages` (`GroupQueuedIntegrationEvent`).
- **Dependencies**: no new runtime NuGet packages planned; BenchmarkDotNet (already a candidate from `ecs-foundation`) and any client test helpers are candidates that must pass the currency check at apply time. WebSocket support is built into ASP.NET Core and the browser.
- **Not affected**: `DigitalTwin` physics and telemetry, `Weather` module, existing endpoints, existing published specs other than the one ADDED requirement above.
- **Later changes**: `park-economy` consumes `ParkMetrics`, `RideTraffic`, purchase events and writes `ParkRating`, `ParkPricing`, `SpawnModifiers`; `coaster-editor` rides plug into the same boarding port.

> **Quality bar:** this change is for a workshop demo, not a production release. See `docs/demo-scope.md` for what may be deferred and what may not.
