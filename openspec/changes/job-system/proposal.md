## Why

The ECS simulation core introduced by `ecs-foundation` (ADR `docs/adr/game-0001-ecs-for-simulation-core.md`) runs its systems one after another on a single thread. That is correct but wastes the cores a park simulation with thousands of guests and many rides will need, and it gives the `ride-on-ecs` migration nothing to schedule its systems with. `System.Threading.Tasks.Parallel` and the .NET ThreadPool cannot fill the gap: their partitioning depends on the thread count and on timing, so the same input can produce different floating-point results on different machines, which breaks the deterministic-replay rule this project is built on (`docs/adr/game-0006-deterministic-parallel-simulation.md`). We need a job system whose results do not depend on how many workers ran it, so parallel speed never costs us reproducibility.

## What Changes

- Add a **work-stealing job system** inside the `Simulation` module (`src/Simulation`, created by `ecs-foundation`): one Chase-Lev style deque per worker, a global injection queue, and idle-worker parking.
- Add **counters and dependency handles** (the Naughty Dog "counter" idea) with **stackless coroutine continuations**: a job can `await` a handle and resume as a new job when the counter reaches zero. Real fibers are not available on the .NET CLR and cannot exist in browser JavaScript, so continuations are built from a custom awaitable and async method builder instead (ADR `docs/adr/game-0003-work-stealing-job-system-without-fibers.md`).
- Add **parallel-for and parallel-reduce with deterministic range partitioning**: the chunk boundaries are a pure function of `(count, grainSize)` and never of the worker count.
- Add a **job graph builder** for ECS system scheduling: each system declares the component types it reads and writes, and the builder derives dependency edges so conflicting systems keep their registration order while independent systems run in parallel.
- Add **worker configuration** (`WorkerCount` of 1 for deterministic replay, N for performance, plus an `Inline` serial reference mode), **cancellation**, **exception propagation**, **graceful shutdown**, and best-effort **worker affinity and priority** options.
- Add **instrumentation**: OpenTelemetry metrics on the shared `BoogaBoosterTelemetry.Meter` and a per-frame **job timeline** (per-worker lanes) exposed through a provider and a `GET /simulation/jobs/timeline` endpoint, for a dashboard view built by `ride-on-ecs`.
- Add **stress and race tests**, a **thread-count-independence golden master** (ADR game-0006), and **benchmarks** against `Parallel.For` and the ThreadPool with measurable acceptance numbers.
- **Frontend (TypeScript): no job system in this change.** Web Workers are out of scope. The public contracts are shaped so a later Web Worker pool is not precluded (see design.md, "TypeScript and Web Workers note").
- No breaking changes. Nothing in `DigitalTwin`, `Queue` or `Weather` is touched; no existing system uses the job system until `ride-on-ecs`.

## Capabilities

### New Capabilities
- `job-scheduling`: Executing jobs on a configurable pool of work-stealing workers with exactly-once execution, counters and dependencies, awaitable continuations, cancellation, exception propagation, shutdown, and worker affinity and priority options.
- `parallel-range-execution`: Parallel-for and parallel-reduce whose chunk partitioning and reduction order are independent of worker count and timing.
- `system-job-graph`: Building and running a dependency graph of ECS systems from declared read and write component sets, with registration order as the reference semantics.
- `job-determinism`: The guarantee that simulation output is identical for every worker count and for the inline reference mode, verified by a golden master.
- `job-instrumentation`: Metrics, the per-frame job timeline, and the timeline endpoint, all observable without affecting simulation results.

### Modified Capabilities
<!-- None: no previously published capability changes its requirements. -->

## Impact

- **New code** in `src/Simulation/FourDotnet.BoogaBooster.Simulation/Jobs/` (scheduler, deque, counters, graph, instrumentation, endpoint) and `src/Simulation/FourDotnet.BoogaBooster.Simulation.Abstractions/Jobs/` (public contracts: `IJobScheduler`, `JobHandle`, `IJob`, `JobSystemOptions`, `IJobTimelineProvider`, timeline DTOs). Names are intent; `ecs-foundation` fixes the surrounding conventions.
- **New tests** in `src/Tests/FourDotnet.BoogaBooster.Simulation.Tests/Jobs/`, including a checked-in golden master file. **New benchmark project** `src/Tests/FourDotnet.BoogaBooster.Simulation.Benchmarks/` (not part of the default `dotnet test` run).
- **Dependencies:** one candidate third-party package, BenchmarkDotNet, for the benchmark project only. It must pass the currency check at apply time (see design.md).
- **Configuration:** a new `Simulation:Jobs` options section (`WorkerCount`, `Mode`, `PinWorkers`, `WorkerPriority`, timeline settings).
- **Docs:** `docs/simulation/job-system.md` (design walkthrough) and `docs/simulation/job-system-benchmarks.md` (recorded results).
- **Consumers:** `ride-on-ecs` (the next change) schedules the Octopus systems with it; later changes (guests, economy) inherit it.
- **Not affected:** the Angular app, the SSE telemetry contract, and all published specs under `openspec/specs/`.

> **Quality bar:** this change is for a workshop demo, not a production release. See `docs/demo-scope.md` for what may be deferred and what may not.
