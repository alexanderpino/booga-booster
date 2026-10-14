## Context

`ecs-foundation` (change 2) creates the `Simulation` module: `src/Simulation/FourDotnet.BoogaBooster.Simulation` plus `.Abstractions`, with entities, structure-of-arrays component storage, queries and a serial system loop (ADRs `docs/adr/game-0001-ecs-for-simulation-core.md` and `docs/adr/game-0002-data-oriented-component-storage.md`). `scene-graph` (change 3) adds flat parent-indexed transforms updated in topological order. Both are applied before this change. Their exact type names are not fixed yet, so everything below that touches them is stated as intent and the apply agent reconciles names against the archived specs of those changes.

The existing simulation is a fixed 1/120 s step (8.33 ms budget), deterministic, with randomness injected and time behind `TimeProvider` (`CLAUDE.md`, "Physics and the ride domain"). The existing tick histogram `boogabooster.ride.simulation.tick.duration` and counter live on `BoogaBoosterTelemetry` (`src/Shared/FourDotnet.BoogaBooster.Core/Observability/BoogaBoosterTelemetry.cs`) and `RideSimulationService.Tick` deliberately measures instead of spanning (`src/DigitalTwin/FourDotnet.BoogaBooster.DigitalTwin/Application/RideSimulationService.cs`). The job system follows that precedent.

Motivation and scope are in `proposal.md`. The decision record for the "no fibers" choice is `docs/adr/game-0003-work-stealing-job-system-without-fibers.md`; the determinism rules are in `docs/adr/game-0006-deterministic-parallel-simulation.md`. Both are written in parallel with this change; this design cites them and does not wait for them.

## Goals / Non-Goals

**Goals:**
- A job scheduler whose results are bit-identical for every worker count (1 for replay, N for speed) and for an inline serial mode.
- Low overhead: zero managed allocation per job in steady state, scheduling overhead small against the 8.33 ms tick budget.
- A system graph builder driven by declared read and write sets, so ECS systems get parallelism without hand-written locks.
- Enough observability (metrics, per-frame timeline) to see where a frame's time went, and to teach it.
- Contracts that a future browser worker pool could also satisfy.

**Non-Goals:**
- No job system in the Angular app, no Web Workers, no `SharedArrayBuffer`, no cross-origin isolation headers.
- No replacement of `Task`/`async` for I/O. Jobs are for CPU-bound simulation work; HTTP handlers, SSE and RabbitMQ stay on the ThreadPool.
- No NUMA awareness, no GPU compute, no distributed execution.
- No automatic discovery of data races. Declared access sets are checked in a debug mode, not proven.
- No change to `DigitalTwin`, `Queue` or `Weather`.

## Decisions

### D1. Continuations are stackless (async state machines), not fibers

A classic job system (Naughty Dog, GDC 2015) lets a job wait on a counter by switching to another fiber, which keeps the waiting job's stack alive. That needs user-mode stack switching. The .NET CLR has no supported fiber API (the old CLR fiber-mode hosting was removed; Win32 `ConvertThreadToFiber` interop corrupts CLR thread state, GC stack walks and thread-static data), and a browser JavaScript runtime has no stack switching at all. ADR `docs/adr/game-0003-work-stealing-job-system-without-fibers.md` records this.

We keep the programming model that makes fibers attractive (a job can wait on a counter without blocking its worker) by making the waiting job a **stackless coroutine**: a C# `async` method compiled to a state machine, with a custom awaitable `JobHandle` and a custom `AsyncMethodBuilder` (`JobTask`). When the awaited counter reaches zero, the builder re-queues the state machine's `MoveNext` as a new job on the releasing worker's deque. Nothing is ever resumed on the ThreadPool or a `SynchronizationContext`.

Trade-offs accepted: (a) only the async method's own frame is suspended, so a helper called from the job cannot itself suspend unless it is also `async`; (b) an async job allocates one state-machine box, so hot loops use plain struct jobs and counters, and async is for orchestration; (c) a job that awaits a regular `Task` would resume off the pool, so the builder offers a debug assertion for "resumed on a worker" and the docs forbid it.

*Alternatives rejected:* real fibers (unsupported, see above); `Task`/`ValueTask` on the ThreadPool (default scheduler ignores our workers, partitions by timing, allocates, cannot be pinned or counted); blocking `Wait` inside a job (deadlocks when all workers wait, the classic job-system trap); thread-per-system (no load balancing).

### D2. Per-worker Chase-Lev deques plus a global injection queue

Each worker owns a growable circular-array deque after Chase and Lev ("Dynamic Circular Work-Stealing Deque", 2005) with the memory-ordering corrections of Lê, Pop, Cohen and Zappa Nardelli (PPoPP 2013). The owner pushes and pops at the bottom (LIFO, cache-warm, depth-first for fork-join). Thieves steal from the top with a compare-exchange (FIFO, oldest and usually largest work first). The pop path needs a full fence between the bottom write and the top read; in .NET that is `Interlocked.MemoryBarrier()` or an `Interlocked` operation on `top`, not a plain `Volatile`, because .NET's memory model on ARM64 is weaker than x86. Tests must therefore run on ARM64 as well where CI offers it (see Risks).

External threads (the simulation driver, HTTP threads) cannot push to a worker deque, so they enqueue to a `ConcurrentQueue`-based **injection queue**. Workers drain it when their own deque is empty and also check it every 61st loop to avoid starvation under a busy deque (the Go scheduler's trick, chosen for the same reason).

The deque has a configurable initial capacity (default 1024, a power of two) and doubles when full up to a hard cap (default 65,536 per worker, `Simulation:Jobs:MaxQueuedJobsPerWorker`). Above the cap new jobs spill to the injection queue, so memory stays bounded and no job is lost.

Worker loop: pop own deque, else (every 61st iteration first) injection queue, else steal from other workers starting at a victim chosen by a per-worker xorshift generator seeded from `options.Seed ^ workerIndex`, else spin briefly (bounded `SpinWait`), else park on a per-worker event. A push wakes one parked worker only when no worker is currently "searching", which prevents a thundering herd. The victim seed affects performance only; the determinism tests randomise it to prove that (spec `job-determinism`).

*Alternatives rejected:* one shared `ConcurrentQueue` (contention on every job, no locality); a lock-based per-worker queue (simple, but the owner's hot path pays a lock); `System.Threading.Channels` (allocation and async overhead per item, no LIFO/steal split); the ThreadPool's own local queues (not controllable, see Benchmarks).

### D3. Counters, handles and a frame-scoped arena

A `JobHandle` is a small `readonly struct` holding a counter index and a frame generation. A counter holds an atomic remaining-count, a status (`Pending`, `Completed`, `Faulted`, `Canceled`), a lock-free continuation list (a Treiber stack sealed with a sentinel when the counter completes) and the captured exception. Scheduling a job increments its counter; a finishing job decrements it; the decrement that reaches zero publishes completion, releases waiters and releases dependent jobs by pushing them onto the releasing worker's deque (or the injection queue if the releaser is not a worker).

Counters live in a **per-frame arena** (a pre-sized array of counters reset by `EndFrame`), not a general pool. Frame-scoped lifetime removes ABA and reuse bugs, makes the zero-allocation target realistic, and matches how a game loop works: each tick opens a frame, runs the graph, completes and closes the frame. A handle from an ended frame fails the generation check and throws a stale-handle error rather than aliasing another job (spec `job-scheduling`). The arena grows between frames, never during one beyond a documented soft cap, and the cap is surfaced as a clear error, not silent corruption.

*Alternatives rejected:* ref-counted pooled counters (more states, harder to test, saves nothing for a frame-structured loop); `TaskCompletionSource` per job (an allocation per job and ThreadPool continuations).

Visibility: completing a counter and releasing a dependent job use release/acquire ordering (`Interlocked` or `Volatile.Write`/`Read` pairs) so a dependent job sees every write its dependencies made. This is what makes "declared write, then declared read" safe without locks.

### D4. Job shapes: struct jobs for the hot path, delegates for convenience

`IJob` is a single-method interface (`Execute` taking a context carrying the worker index, job ordinal, cancellation token and scheduler). Struct jobs are stored by value in per-type slabs so scheduling allocates nothing; a delegate overload exists for tests and one-off orchestration and is documented as allocating. `IRangeJob` is the parallel-for body (`Execute(int start, int end, in JobContext ctx)`), so the body gets a range and amortises the call over a chunk.

### D5. Deterministic partitioning and reduction (ADR game-0006)

`ParallelFor(count, grain)` creates `ceil(count / grain)` chunks with fixed boundaries, one job per chunk (or a small bounded number of range-jobs that split by halving, as long as the leaf boundaries stay exactly the fixed chunks). The chunk set is a pure function of `(count, grain)`. Work stealing balances the chunks, but a chunk never changes shape. `Parallel.For`'s dynamic partitioner by contrast chooses ranges from timing and processor count, which is why it cannot give reproducible floating-point reductions.

`ParallelReduce` writes one partial per chunk into an array indexed by chunk and folds it in ascending index order on a single follow-up job. The grain is therefore part of the result's definition, and the docs say so. The default grain is a documented constant (1024) and never derives from the processor count (spec `parallel-range-execution`).

A job that accumulates into shared state with `Interlocked` or `lock` would be race-free but order-dependent for floating point. The API gives no helper for it, and the docs show the per-chunk-slot pattern instead.

### D6. Job graph from declared read and write sets

`JobGraphBuilder` collects named system nodes with read and write sets expressed as component type identifiers (intent: `ComponentTypeId` from `ecs-foundation`) plus an optional set of named non-component resources. Edges are derived in registration order: an edge `earlier -> later` exists when write/read, read/write or write/write sets intersect. Registration order is the reference semantics, so the derived graph can only add forward edges, which makes cycles impossible by construction; explicit "after" relations are validated to point backward, and barrier nodes cut the graph into phases.

Building produces an immutable `JobGraph` with per-node dependency counts and dependency levels. A frame run seeds the nodes with no dependencies and, as each node's counter reaches zero, releases its dependents, so independent systems overlap. A system may start nested parallel-fors; its node completes when those nested jobs complete.

Conflict detection is per component type, which is conservative: two systems writing disjoint entity subsets of the same component are still ordered. Archetype- or chunk-level conflict tracking could relax this and is deliberately not in this change (Open questions).

Checked mode: the system context handed to a system (intent: an `ecs-foundation` `SystemContext`) wraps component access and validates against the declared sets, throwing an error that names the system and the component type. Unchecked mode compiles the checks away, so the hot path pays nothing.

*Alternatives rejected:* the user wires dependencies by hand (error-prone, defeats "declared sets"); ordering by phases only (loses overlap); inferring access from code (not reliable in C#).

### D7. Worker count, modes and the serial oracle

`JobSystemOptions` (bound from `Simulation:Jobs`): `Mode` (`Threaded` or `Inline`), `WorkerCount` (0 = auto = processors minus 1, minimum 1; 1 to 256 otherwise), `InitialDequeCapacity`, `MaxQueuedJobsPerWorker`, `PinWorkers`, `WorkerPriority`, `Seed`, `CheckedAccess`, `Timeline` (enabled, events per worker, history frames). Validation runs through `IValidateOptions<T>` at startup and fails fast.

`Inline` mode is a separate small scheduler implementing the same interface: it runs each job synchronously at submission, parallel-for chunks in index order, a graph in registration order. It is the **oracle** the tests compare every threaded configuration against and the mode `ride-on-ecs` uses for deterministic replay and debugging. `WorkerCount = 1` in `Threaded` mode additionally exercises the real scheduler code with no concurrency. Both must give the same results as any N.

The simulation driver thread is not a worker. It opens the frame, submits the graph, then blocks in `Complete` on the root handle. We chose "caller blocks" over "caller helps run jobs" for the first version: it keeps worker identity simple (timeline lanes, thread-local deque ownership) at the cost of one idle core, which is why auto = processors minus 1 reserves nothing extra for it. Caller participation is an optimisation noted under Open questions.

### D8. Cancellation, exceptions, shutdown

Cancellation is cooperative and counter-safe: a job dequeued after the token fired is not executed but still decrements its counter, so no waiter hangs. A running job sees the token in its context.

An exception is captured on the faulting job's counter; its dependents are skipped (they still decrement, so the graph drains) and complete as faulted with the same cause. `Complete` throws an `AggregateException` whose inner exceptions are ordered by **job ordinal** (scheduling order), never by arrival time, so the surfaced error is itself deterministic.

Shutdown has `Drain` and `Cancel` modes, rejects new work as soon as it begins, joins workers within a timeout, is idempotent, and `DisposeAsync` performs `Cancel`. Worker threads are background threads, so a stuck job cannot keep the process alive; the timeout error reports how many workers did not stop. Hosting: the Simulation module registers the scheduler as a singleton `IJobScheduler` and a hosted-service wrapper that shuts it down on host stop.

### D9. Affinity and priority are best-effort

`Thread.Priority` is set per worker. .NET has no portable per-thread affinity API (`Process.ProcessorAffinity` is process-wide), so pinning goes through a small `IThreadAffinity` abstraction with platform implementations using P/Invoke (`SetThreadAffinityMask` on Windows, `sched_setaffinity` on Linux). macOS gets a no-op. Every failure (missing privilege, unsupported OS, container CPU limits) is recorded in the per-worker report and never thrown. The P/Invoke declarations should use `LibraryImport` source generation if the currency check (below) confirms it is the documented default for the pinned SDK. Default is no pinning and `Normal` priority, because pinning helps benchmarks and hurts on shared dev machines.

### D10. Instrumentation: measure, do not span

Following `RideSimulationService.Tick` and ADR-0009: no `Activity` per job or per chunk. Each worker keeps plain local counters (executed per kind, stolen, busy ticks) and the frame-closing step adds them to the shared counters once per frame, so no atomic is paid per job. Instruments are created from `BoogaBoosterTelemetry.Meter` in a `JobSystemInstruments` class in the Simulation module's `Observability/` folder; OpenTelemetry is configured only in `ServiceDefaults`.

Metric names (intent, prefix `boogabooster.world.jobs.`): `executed` (tag `kind`), `stolen`, `faulted`, `canceled`, `frame.duration` (ms), `system.duration` (ms, tag `system`, capped at 64 distinct values, surplus tagged `other` so cardinality stays bounded), `queue.depth` (observable), `worker.utilization` (observable, 0 to 1).

The **timeline** records, when enabled, `JobTimelineEvent { systemIndex, jobOrdinal, worker, startTicks, endTicks, stolen }` into a per-worker ring buffer (default 4096 events, written only by its owner, no sharing, no atomics). The frame-closing step on the driver thread merges the buffers into a `JobFrameTimeline` DTO (plain values, millisecond offsets from frame start, `schemaVersion` 1, dropped-event count), and stores it in a bounded history (default 120 frames, about one second at 120 Hz). Recording uses `Stopwatch.GetTimestamp()`, reads no simulation state and writes none, so it cannot change results (spec `job-determinism` checks that). `IJobTimelineProvider` (`GetLatest()` and `GetRecent(int maxFrames)`, ascending by frame number) is in the Abstractions project; a query handler and `GET /simulation/jobs/timeline` expose the latest frame, per the module-anatomy rules in `CLAUDE.md` (endpoint has no logic, handler overrides `ExecuteAsync`).

### D11. Test strategy for concurrency

Concurrency tests are bounded and replayable: fixed iteration counts, fixed seeds, no `Thread.Sleep` for synchronisation (barriers and `ManualResetEventSlim` only), a generous cancellation-token timeout purely to turn a hang into a failure, and failure messages that print seed and iteration. Three layers: (1) deque model tests, single-threaded semantics first, then 1 owner plus 3 thieves over 1,000,000 items with an exactly-once bitmap; (2) scheduler tests for each spec scenario; (3) the **golden master**: a synthetic mini-simulation (4096 entities, 5 systems with a read-after-write chain, a parallel-reduce and independent systems, seed 20260101) hashed per frame with SHA-256 over component bit patterns, run for 120 frames under worker counts 1, 2, 4, 8, processor count and `Inline`, with injected delays and randomised steal seeds, compared with a checked-in file. A test also mutates one constant by one ULP and expects the comparison to fail, so the golden master cannot silently pass vacuously.

### D12. Benchmarks and acceptance numbers

A separate project `src/Tests/FourDotnet.BoogaBooster.Simulation.Benchmarks` (BenchmarkDotNet, a candidate dependency) holds the benchmarks. It is excluded from the default `dotnet test` run because wall-clock numbers are noisy on shared CI. Results are recorded in `docs/simulation/job-system-benchmarks.md` with machine, OS, .NET version and date. Allocation and ordering properties are deterministic and are asserted in the ordinary xUnit suite, not only benchmarked.

Honest framing first: the .NET ThreadPool already uses per-thread local queues with work stealing. We do not expect to beat it on raw throughput. What we buy is determinism, dependency counters, zero allocation, a controllable thread count and affinity, and a timeline. The acceptance numbers are chosen to prove "not meaningfully slower, and the extras are free", with hardware-dependent limits stated **relative to the baseline on the same machine**:

| # | Benchmark | Acceptance | Why this number |
|---|-----------|-----------|-----------------|
| B1 | 100,000 empty jobs, workers = processors minus 1, vs `Parallel.For` and vs `ThreadPool.UnsafeQueueUserWorkItem` with `IThreadPoolWorkItem` plus a countdown | Time per job at most 1.25 times the faster of the two baselines | An empty job measures pure overhead. Parity-with-slack is realistic against a heavily optimised runtime pool; 25 percent slack absorbs run-to-run noise (BenchmarkDotNet typically reports a few percent) |
| B2 | Parallel-for, 1,000,000 elements of ~50 ns work, grain 1024, vs `Parallel.For` with a range partitioner | Throughput at least 0.90 times `Parallel.For` | Compute-bound uniform work should be dominated by the body, so a 10 percent gap would signal a scheduling defect |
| B3 | Triangular (imbalanced) workload on the same sizes | Throughput at least 1.00 times `Parallel.For` | Stealing exists to win here; if it does not, the deque or victim choice is wrong |
| B4 | Frame overhead: a graph of 20 systems and 200 empty jobs, 4 workers | Median at most 100 microseconds, p99 at most 250 microseconds | 100 microseconds is 1.2 percent of the 8.33 ms tick budget; p99 below 3 percent keeps jitter invisible to the 120 Hz loop |
| B5 | Wake-up latency, submit to first job start, warm parked worker | p99 at most 50 microseconds | OS thread wake is typically tens of microseconds; 50 microseconds is a generous ceiling that still leaves room for several dependency levels per frame |
| B6 | Scaling: 5-system synthetic frame over 16,384 entities per system with 1, 2, 4, 8 workers | Speedup at 4 workers at least 2.4 times (60 percent efficiency); output hash identical in every run | With a serial fraction (reduce, barriers) of roughly 15 percent, Amdahl's bound at 4 workers is 1 / (0.15 + 0.85/4) = 2.76, so 2.4 leaves about 13 percent for overhead |
| B7 | Allocation per job, steady state | 0 bytes (asserted in xUnit, also reported by `MemoryDiagnoser`) | `ThreadPool.QueueUserWorkItem` with a delegate allocates; the claim is checkable and deterministic |

If a number fails, the task list says to profile and fix before relaxing it, and to record any relaxation, with the measured value, in the benchmark doc and in `Decisions` of the pull request. These thresholds are targets to calibrate on first run (task 12.3 records the baseline before the thresholds are written into the doc), not facts verified today.

### D13. TypeScript and Web Workers note

The frontend gets **no job system** in this change, and Web Workers stay out of scope. The contracts are shaped so they are not precluded:
- Partitioning is a pure arithmetic function of `(count, grain)` documented in `docs/simulation/job-system.md`, so a TypeScript port produces identical chunks.
- The timeline DTO and the graph description are plain JSON with `schemaVersion`, so a browser worker pool could emit the same shape and reuse the dashboard view.
- Jobs are specified as "pure function of declared inputs plus a range", not as .NET delegates, which maps onto message passing between workers.
- Browser constraints a later change must face: no fibers or stack switching, shared memory only through `SharedArrayBuffer`, which requires cross-origin isolation headers (`COOP`/`COEP`) on the Vite dev server and the production host, and the Angular zoneless setup must keep the main thread free. These are listed as a hand-off for a future change, not built here.

## Dependencies and contracts

**Requires from earlier changes (stated as intent, reconcile names at apply time):**
- From `ecs-foundation`: the `Simulation` module projects (`FourDotnet.BoogaBooster.Simulation`, `.Abstractions`) and `AddWorldModule()` / `MapWorldEndpoints()` entry points; a component type identifier (`ComponentTypeId`) usable in access sets; a system abstraction and a per-system context through which components are accessed (so checked mode can wrap it); a deferred structural-change mechanism (intent: command buffer) applied at sync points, which the graph models as a barrier; the test project `FourDotnet.BoogaBooster.Simulation.Tests`.
- From `scene-graph`: nothing is required to compile; the transform-propagation system registers into the graph like any other system. The hierarchy is processed level by level, so each level can be a parallel-for.
- From the repo: `BoogaBoosterTelemetry.Meter` and `SourceName` in `Shared/Core` (the Meter already exists), ServiceDefaults already registering that meter.

**Provides to later changes (intent):**
- `FourDotnet.BoogaBooster.Simulation.Abstractions`: `IJobScheduler`, `JobHandle`, `JobTask`, `IJob`, `IRangeJob`, `JobContext`, `JobSystemOptions`, `ShutdownMode`, `IJobGraphBuilder` / `JobGraph` / `JobGraphDescription`, `IJobTimelineProvider`, `JobFrameTimeline` and its span DTOs.
- `Simulation` module: `JobScheduler`, `InlineJobScheduler`, `JobGraphBuilder`, DI registration inside `AddWorldModule()`, `Simulation:Jobs` configuration, `GET /simulation/jobs/timeline`.
- Metrics listed in D10.
- `ride-on-ecs` consumes the graph builder, the provider and the metrics; `guests-and-needs` and `park-economy` consume the scheduler for large populations.

## Risks / Trade-offs

- **Memory-model bugs on ARM64** → The .NET memory model on ARM is weaker than x86 and a Chase-Lev deque is exactly where this bites. Mitigation: follow the Lê et al. fencing, keep the deque small and heavily tested (1M-item exactly-once stress), and run the stress tests on an ARM64 runner if CI offers one; otherwise record the gap in the PR.
- **Floating-point and `Math` determinism across machines** → The golden master proves independence from worker count on one machine. Bit-identical results across CPU architectures also depend on the JIT (fused multiply-add contraction, `Math.Sin` implementation). Mitigation: the golden hashes are generated and checked on the same runner class; cross-architecture equivalence is a tolerance concern recorded in ADR game-0006, not promised here.
- **Declared sets can lie** → A system may touch data it did not declare. Mitigation: checked mode in tests and in debug runs; the golden master and repeated-run tests catch the symptom (nondeterminism).
- **Async jobs resumed off-pool** → Awaiting a non-job awaitable breaks the model. Mitigation: debug assertion that a continuation runs on a worker; docs; async limited to orchestration.
- **Wall-clock benchmark noise** → CI cannot gate on timing. Mitigation: timing is reviewed from the recorded benchmark doc; only deterministic properties (allocation, ordering, exactly-once) gate the build.
- **Oversubscription** → The ASP.NET host also runs on the ThreadPool and the 120 Hz loop. Auto worker count leaves one core free; tests that spawn many workers cap at the processor count.
- **Thread affinity side effects** → Pinning on a shared or containerised host can hurt. Mitigation: off by default, failures reported not thrown.
- **Tiny workloads lose to overhead** → A world with a handful of entities is faster inline. Documented; `ride-on-ecs` measures this honestly and may choose `Inline` or 1 worker for a single ride.
- **Third-party library candidates** → BenchmarkDotNet is a *candidate*. At apply time it must pass the currency check: not deprecated in the version the repo's SDK pins, official docs name no different default for micro-benchmarking, and upstream not archived with a release in the last 12 months. P/Invoke declarations must likewise be checked against the documented default (`LibraryImport` versus `DllImport`). No version is asserted here because none was verified.

## Migration Plan

Purely additive. No data migration. The scheduler is registered but unused until `ride-on-ecs`. Rollback is reverting the change: no persisted state, no wire contract, no published spec is touched. `Simulation:Jobs` defaults (`Mode` = `Threaded`, `WorkerCount` = 0) apply when configuration is absent.

## Open Questions

- Should the calling thread help run jobs while it waits (reduces idle cores, complicates timeline lanes)? Deferred: the first version blocks, and benchmark B6 shows whether the lost core matters.
- Should conflict detection refine to archetype or chunk level to allow two writers of one component on disjoint archetypes? Deferred: needs `ecs-foundation` storage details; per-type is correct and simple.
- Is a runtime "replay verifier" (run each frame twice, inline and threaded, compare hashes) worth shipping as a debug option? Deferred: it doubles frame cost and the golden master already covers CI; revisit after `ride-on-ecs` shows how often declared sets drift.
