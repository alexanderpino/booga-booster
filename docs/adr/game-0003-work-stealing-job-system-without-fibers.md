# game-0003: Work-stealing job system without fibers

- Status: Proposed (the user must accept)
- Date: 2026-10-14
- Deciders: project owner
- Related: game-0002, game-0006

## Context

Systems over chunks (game-0002) are data-parallel and some depend on others (transform update before collision before guest decisions). We want a scheduler that runs this graph on all cores with low overhead and deterministic results. Production game engines (the best-known public description is Naughty Dog's GDC 2015 talk on fiber-based job systems) use fibers so a job can wait without blocking a worker thread. We must decide what to do on .NET and in the browser, where fibers are not an option.

## Decision drivers

1. Scale across cores with small jobs (a chunk, microseconds each).
2. Express dependencies without blocking threads.
3. Deterministic outcome regardless of scheduling (game-0006).
4. Works on the backend (.NET 10) and has a credible browser variant.
5. Teachable: the algorithm fits on a few slides.
6. No unsafe or unsupported runtime hacks.

## Why real fibers are rejected

1. **.NET (CLR).** A fiber switches stacks in user mode on one OS thread. The CLR tracks per-thread state (managed thread identity, stack walking for GC root scanning, exception handling, thread statics, security/execution context) and assumes one managed thread equals one OS thread. Switching stacks under it with the Win32 Fiber API (`ConvertThreadToFiber`, `CreateFiber`, `SwitchToFiber`) from managed code is therefore unsupported and unsafe: the GC may not find roots on a parked fiber stack and it does not run cross-platform. The old .NET Framework 2.0 hosting API had a fiber-mode hosting contract (`ICLRTask::SwitchIn/SwitchOut`, `IHostTaskManager`, per the Microsoft Learn ICLRTask page: ".NET Framework Versions: Available since 2.0"); that is a .NET Framework unmanaged hosting API, not something managed code or modern .NET offers. Status: the Learn page was read (2026-10-14); the claim that modern .NET (CoreCLR) offers no fiber support is NOT verified against an explicit Microsoft statement and must be re-checked before quoting it publicly. Also Linux/macOS have no Win32 Fiber API; `ucontext` style switching is a native concern the CLR does not coordinate with.
2. **Browser JavaScript.** Each realm is single-threaded with an event loop; there is no user-mode stack switching primitive. Parallelism requires Web Workers. Sharing memory between workers requires `SharedArrayBuffer`, which MDN documents as available only in a secure context AND when cross-origin isolated (`crossOriginIsolated` true); without it the constructor is hidden and `postMessage` of a `SharedArrayBuffer` throws. Isolation is enabled by serving the document with `Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy: require-corp` (MDN page states this "from general knowledge"; confirm values on the linked MDN pages at apply time). `Atomics` is always available. Consequence: all cross-origin subresources (CDN scripts, fonts, glTF from another host) must send CORS/CORP headers.
3. **What fibers would have bought** (waiting jobs do not hold a thread) is replaced below by dependency counters plus continuations, which cover our needs because jobs are non-blocking by design.

## Considered options

### Option A: .NET ThreadPool / `Parallel.For` / `Task`
- Pros: built in, supported, well-tested; `Parallel.ForEach` over chunks is a few lines.
- Cons: global queue contention and work distribution not under our control; scheduling and partitioning are not guaranteed identical between runs; allocation per task unless carefully pooled; dependency graphs need `ContinueWith`/`await` allocation.

### Option B: `System.Threading.Channels` pipeline
- Pros: clean producer/consumer, back-pressure.
- Cons: designed for async streaming, not for fine-grained data-parallel jobs; per-item overhead too high for microsecond jobs.

### Option C: Own work-stealing scheduler with dependency counters and stackless continuations (recommended)
- Pros: low overhead, no blocking, deterministic partitioning, same algorithm reusable in TS workers, teachable.
- Cons: we maintain concurrency-sensitive code (lock-free deque correctness); needs thorough tests; coroutines are stackless so awaiting is only possible at defined points.

### Option D: Fibers via native interop
- Rejected for the reasons above.

## Decision

Implement Option C:

1. **Workers**: N worker threads (N from configuration, default `Environment.ProcessorCount`, overridable for tests). Each owns a **work-stealing deque** (Chase-Lev style: owner pushes/pops at the bottom, thieves steal from the top with compare-and-swap). Idle workers steal from a victim chosen by a seeded round-robin; stealing affects only who runs a job, never the result (game-0006).
2. **Jobs** are structs/pooled objects: a delegate-free entry (`IJob.Execute(in JobContext)`), a **dependency counter** (atomic int of unfinished prerequisites) and a list of dependents. Finishing a job decrements its dependents' counters; the one that reaches 0 is pushed to the finishing worker's deque.
3. **Parallel-for** splits a chunk list (game-0002) into ranges; a parent job holds a counter equal to the number of ranges and fires its continuation at 0.
4. **Stackless coroutine continuations**: a job may be written as a C# iterator/state machine that yields a wait on a counter (`yield return wait`); the scheduler parks the state machine object (heap, pooled) and re-enqueues it when the counter hits 0. No stack is switched. Limit: yields only at the top level of the coroutine.
5. **System graph**: each system declares read/write component sets; the scheduler derives dependency edges at build time in registration order (so the same graph every run) and runs independent systems concurrently.
6. **Frontend variant**: a pool of Web Workers sharing component columns in `SharedArrayBuffer` + `Atomics` (counters, deque indices). It requires cross-origin isolation (headers above) and is only worthwhile if client-side work (e.g. interpolation, culling, skinning) profiles as a bottleneck. Because the backend is authoritative (game-0005) the frontend job system is **optional and deferred**; the fallback is single-threaded main-loop or one worker with `postMessage` transfer.
7. Everything is candidate-checked at apply time; no third-party scheduler is assumed.

## Consequences

- Good: no blocking, predictable overhead, determinism hooks in one place, showcases a well-known technique honestly adapted.
- Bad: lock-free code is a defect magnet; debugging concurrency; coroutines are less ergonomic than `await`.
- Neutral: a `ThreadPool`-backed executor stays available behind the same `IJobScheduler` interface as baseline and for comparison.

## Validation

- Throughput: parallel-for over 100,000 trivial elements scales >= 0.7x linear efficiency up to 8 workers on the CI reference machine (target to confirm).
- Overhead: scheduling one empty job < 1 microsecond amortised and zero steady-state GC allocations.
- Stress: 10,000 randomised dependency graphs (seeded) complete with each job executed exactly once; run under a repeat loop of at least 1,000 iterations without hang.
- Thread-count independence: simulation results identical for 1, 2, 4, 8 workers (game-0006).
- Baseline comparison: our scheduler vs `Parallel.ForEach` on the same chunk workload, reported in the PR.
- Frontend (if built): `crossOriginIsolated === true` asserted in an e2e test.

## References

- Naughty Dog, "Parallelizing the Naughty Dog Engine Using Fibers" (GDC 2015) - cited from memory, NOT fetched, unverified link.
- Chase and Lev, "Dynamic circular work-stealing deque" (SPAA 2005) - cited from memory, unverified.
- https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer (read 2026-10-14)
- https://learn.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/iclrtask-interface (read 2026-10-14; .NET Framework hosting API)
- docs/adr/game-0002-data-oriented-component-storage.md, game-0006-deterministic-parallel-simulation.md
