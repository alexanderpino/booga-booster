## Why

The Octopus ride (the result of `octopus-motion`) still lives in `DigitalTwin` as DDD aggregates (`Ride`, `GreatMill`, `Hub`, `Gondola`, `Seat`) stepped by one thread. The ECS core (`ecs-foundation`), the scene graph (`scene-graph`) and the job system (`job-system`) now exist, but nothing real runs on them. Migrating the ride is the first proof that the ECS-plus-jobs architecture (ADRs `docs/adr/game-0001-ecs-for-simulation-core.md`, `game-0002-data-oriented-component-storage.md`, `game-0006-deterministic-parallel-simulation.md`) can carry a real, tested, deterministic simulation, and it gives every later park change a working template. The migration must be a strangler fig: the ride is live software with ~330 backend tests and 20 published specs, so the old path stays until the new one is proven equivalent.

## What Changes

- Add an **ECS representation of the Octopus** inside the `DigitalTwin` module: components for transform, angular state (mill, hub, gondola), arm elevation, motor drive, brake, gondola, seat and passenger reference; entities for one ride, four main arms, four hubs, sixteen gondolas and thirty-two seats.
- Add **systems with declared read and write sets**, scheduled by the job system as a graph: command application, lifecycle, motor torque, rotational dynamics, cam elevation, arm kinematics, gondola pendulum, G-force, transform write and telemetry projection.
- Add an **anti-corruption layer**: an `IRideStore` implementation backed by the World (`EcsRideStore`), so the existing handlers, endpoints, the SSE telemetry stream, the loading coordinator and the lifecycle state machine see no difference. The lifecycle policy is extracted from `Ride` into a pure shared component so both engines use identical rules.
- Add a **feature flag** `DigitalTwin:Simulation:Engine` with values `Legacy` (default until the final step), `Ecs` and `Shadow`. Shadow runs both engines on the same commands, answers from the legacy engine, and counts divergence.
- Add a **golden-master equivalence suite**: the same seeded scenarios run on both engines and the telemetry is compared per tick in three tiers (bit-identical where float order is preserved, a defined relative tolerance for summed values, exact for discrete values). The same suite proves the ECS output is identical for every job-system worker count.
- Run the existing store-level tests against both engines as **contract tests**, so the existing behaviour specs keep holding on the new path.
- Add a **performance comparison** (benchmarks, old versus new, serial versus parallel, single ride versus a scaled-out world of many rides) with acceptance numbers.
- Add an **engine timings dashboard**: a `GET /ride/engine/timings` endpoint and a PrimeNG view (system durations, tick budget, per-worker job timeline).
- Add a **removal plan** for the legacy path, as a document and a follow-up change outline. The legacy code is **not removed in this change**; it is the fallback.
- **No breaking changes.** Endpoints, the SSE telemetry contract, the state machine and all published specs under `openspec/specs/` stay as they are.

## Capabilities

### New Capabilities
- `ride-engine-selection`: Choosing the simulation engine (Legacy, Ecs, Shadow) by configuration, with identical external behaviour in every mode and shadow-mode divergence reporting.
- `ride-ecs-simulation`: The Octopus modelled as ECS entities, components and job-scheduled systems with declared access, producing the same ride behaviour and telemetry as the aggregates.
- `ride-engine-equivalence`: The golden-master contract that the ECS engine matches the legacy engine within defined tolerances and is independent of worker count.
- `engine-timings-dashboard`: The engine timings endpoint and the PrimeNG dashboard view of tick, system and job timings.

### Modified Capabilities
<!-- None. Existing capabilities (ride-state-machine, ride-telemetry-streaming, engine-braking, gondola-boarding, ...) must keep their current requirements and pass unchanged on both engines. -->

## Impact

- **Backend, `DigitalTwin` module** (`src/DigitalTwin/FourDotnet.BoogaBooster.DigitalTwin/`): new `Ecs/` folder (components, systems, world factory, ACL store, telemetry mapper, shadow store, timings collector); a small extraction of the lifecycle policy and load helpers from `Domain/` (behaviour-preserving); `DigitalTwinModuleOptions` gains a `Simulation` section; `DigitalTwinModuleExtensions` selects the engine; a new `Features/GetEngineTimings/` query and a new route in `DigitalTwinEndpoints`.
- **Backend, `DigitalTwin.Abstractions`**: new `DataTransferObjects/EngineTimings/` DTOs. `RideTelemetry` and the SSE contract are unchanged.
- **Depends on** `Simulation.Abstractions` (extension surface for registering components and systems, the job scheduler and timeline provider) and, through it, the job system. `DigitalTwin` references the `.Abstractions` project only, per the modular-monolith rule.
- **Tests:** `src/Tests/FourDotnet.BoogaBooster.DigitalTwin.Tests/` gains contract tests, golden-master tests and ECS system tests; a new `src/Tests/FourDotnet.BoogaBooster.DigitalTwin.Benchmarks/` project (BenchmarkDotNet is a candidate and must pass the currency check).
- **Frontend** (`src/FourDotnet.BoogaBooster.App/`): a new lazy-loaded engine timings route with a PrimeNG panel (first real PrimeNG use), a data service, models, Vitest and axe-core specs. The existing dashboard, SSE source and 3D visualization are untouched.
- **Docs:** `docs/simulation/ride-on-ecs.md` (component and system tables, step order), `docs/simulation/ride-on-ecs-equivalence.md` (tolerance tiers and scenarios), `docs/simulation/ride-on-ecs-removal-plan.md`, `docs/simulation/ride-on-ecs-performance.md`.
- **Configuration:** `DigitalTwin:Simulation:Engine`, `DigitalTwin:Simulation:ShadowCompareEveryTicks`, and `Simulation:Jobs` settings for the ride's world.
- **Not affected:** `Queue`, `Weather`, `Controller`, the Aspire graph, the Angular dashboard and 3D view, and the published specs.

> **Quality bar:** this change is for a workshop demo, not a production release. See `docs/demo-scope.md` for what may be deferred and what may not.
