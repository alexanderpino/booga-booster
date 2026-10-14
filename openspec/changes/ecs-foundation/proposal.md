# Proposal

## Why

BoogaBooster's physics lives in DDD aggregates (`GreatMill`, `Hub`, `Gondola`) that are fine for 21 rotating bodies, but the roadmap (Octopus ride, then a park with thousands of guests and many rides) needs a simulation core that handles 100 000+ entities inside the 1/120 s tick budget (8.33 ms) and can later be run in parallel without changing results. A data-oriented Entity-Component-System (ECS) with structure-of-arrays storage is that core (ADR `docs/adr/game-0001-ecs-for-simulation-core.md`, `docs/adr/game-0002-data-oriented-component-storage.md`). Everything after it (scene graph, job system, ride-on-ecs, the park changes) builds on it, so it has to land first and has to be small, measured and teachable.

## What Changes

- New backend module `Simulation` (`src/Simulation/FourDotnet.BoogaBooster.Simulation` + `.Abstractions`, tests in `src/Tests/FourDotnet.BoogaBooster.Simulation.Tests`) registered in `src/BoogaBooster.slnx` and composed by the `Api` host through `AddSimulationModule()` / `MapSimulationEndpoints()`.
- ECS core: generational entity ids, explicit component registration with stable names, archetype-based structure-of-arrays storage, allocation-free chunked queries with `With`/`Without` filters, and a deferred entity command buffer for structural changes.
- System scheduling interface with declared read/write component access, a dependency-ordered execution plan and a single-threaded executor. The plan is built so the later job system can run independent systems in parallel and produce identical results.
- Fixed-step tick driver (1/120 s) built on `TimeProvider`, drift-free, with a catch-up cap and a hosted background service (disabled by default until a system exists).
- Deterministic iteration order, a canonical state hash, and binary save/restore of the whole world as the snapshot/serialization hook for later state synchronisation (ADR `docs/adr/game-0005-simulation-authority-and-state-sync.md`), plus a JSON projection of components marked as replicated.
- Diagnostics: tick, system and entity metrics on the shared `BoogaBoosterTelemetry` meter, and `GET /simulation/diagnostics` and `GET /simulation/snapshot` endpoints.
- Benchmark project (BenchmarkDotNet candidate) with explicit performance budgets, and an allocation-free-iteration unit test.
- Angular: only a thin typed reader/validator for the JSON snapshot (`src/app/world/snapshot/`). **No client-side ECS** (decision and reasons in `design.md`).
- Documentation: `docs/simulation/01-ecs-foundation.md` teaching guide; CLAUDE.md status table gains a `Simulation` row.

No existing module, endpoint or telemetry stream changes. `DigitalTwin` keeps its DDD physics until `ride-on-ecs`.

## Capabilities

### New Capabilities
- `ecs-entity-lifecycle`: Generational entity ids, component registration, component add/remove/set, structural-change safety and the deferred command buffer.
- `ecs-queries`: Filtered, chunked, allocation-free queries and the deterministic iteration order guarantee.
- `ecs-system-scheduling`: The system interface, declared component access, dependency-ordered execution plan, sequential executor.
- `fixed-tick-driver`: The 1/120 s fixed-step driver over `TimeProvider`, catch-up cap, hosted service.
- `ecs-world-snapshot`: Canonical state hash, binary save/restore, JSON projection of replicated components, and the typed client snapshot reader.
- `simulation-diagnostics`: Metrics and read-only diagnostics endpoint for the simulation core.
- `ecs-performance-budget`: Measurable throughput and allocation budgets for the ECS and how they are measured.

### Modified Capabilities
<!-- None. No published capability changes its requirements. -->

## Impact

- **New code**: `src/Simulation/**`, `src/Tests/FourDotnet.BoogaBooster.Simulation.Tests/**`, `src/Benchmarks/FourDotnet.BoogaBooster.Simulation.Benchmarks/**`, `src/FourDotnet.BoogaBooster.App/src/app/world/snapshot/**`, `docs/simulation/01-ecs-foundation.md`.
- **Edited code (small)**: `src/BoogaBooster.slnx` (three project entries), `src/FourDotnet.BoogaBooster.Api` (project reference, two `Program.cs` calls), `CLAUDE.md` and `.github/copilot-instructions.md` (status row; `ai-tooling-sync` check).
- **Dependencies**: no new runtime NuGet packages in the module; `BenchmarkDotNet` is a benchmark-project-only candidate dependency (currency check at apply time).
- **Shared telemetry**: uses `BoogaBoosterTelemetry.Meter`; `ServiceDefaults` already collects that source name, so no host OTEL change.
- **Later changes**: `scene-graph` (components and lifecycle observer), `job-system` (execution plan, command-buffer merge order), `ride-on-ecs`, and every park change consume the contracts listed in `design.md`.

> **Quality bar:** this change is for a workshop demo, not a production release. See `docs/demo-scope.md` for what may be deferred and what may not.
