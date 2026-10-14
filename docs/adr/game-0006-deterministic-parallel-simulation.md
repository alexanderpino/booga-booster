# game-0006: Deterministic parallel simulation

- Status: Proposed (the user must accept)
- Date: 2026-10-14
- Deciders: project owner
- Related: game-0002, game-0003, game-0005

## Context

The existing physics is deterministic by construction: fixed 1/120 s symplectic step, injected samplers, `TimeProvider`. Parallel execution (game-0003) threatens this: floating-point addition is not associative, so summing in a different order gives different bits; work stealing makes execution order vary. Without determinism we lose replay tests, golden masters, reliable bug reports and trustworthy workshop demos.

## Decision drivers

1. Same inputs + same seed => identical state, independent of thread count and scheduling.
2. Must not forfeit parallel speed-up.
3. Testable with automation.
4. Works across runs on the same machine; cross-machine bit-identity is a stretch goal.

## Considered options

### Option A: Run single-threaded
- Pros: trivially deterministic.
- Cons: defeats the job system, limits park size.

### Option B: Allow non-determinism, test with tolerances
- Pros: free speed.
- Cons: flaky tests, irreproducible bugs, cannot golden-master.

### Option C: Deterministic parallelism by construction (recommended)
Rules in the Decision.

### Option D: Fixed-point / integer arithmetic everywhere
- Pros: cross-platform bit-identical.
- Cons: rewrites the existing double-based physics; unreadable code; out of proportion.

## Decision

Option C with these rules, each enforced by test or analyser where possible:

1. **No shared mutation**: a job writes only to its own chunk range or to its own output slot; reads of data written in the same phase are forbidden (the system graph in game-0003 enforces read/write declarations).
2. **Fixed-order reduction**: any aggregate (total revenue, total G-force statistics, queue counts) is computed by per-chunk partial results stored in an array indexed by chunk number, then combined by one thread in ascending chunk order. No `Interlocked.Add` on floats, no `Parallel.For` with thread-local accumulators combined in completion order.
3. **Structural changes** deferred and applied in worker-index then record order (game-0002).
4. **Float rules**: use `double` for physics; avoid `Math.FusedMultiplyAdd` unless used everywhere consistently; no use of non-deterministic intrinsics whose result varies by hardware path (verify at apply time which `System.Math`/`MathF` functions are bit-stable across the CPUs we target; unverified here); no reliance on `float.Sum` via vectorised LINQ with unspecified order; SIMD loops must use a fixed lane count and fixed horizontal-reduction order, independent of the machine's vector width (`Vector<T>.Count` varies; use `Vector128` explicitly or avoid).
5. **Randomness**: all random draws come from seeded samplers injected through an interface (existing pattern). One stream per entity or per system derived from (world seed, system id, entity index) via a counter-based generator, so the draw does not depend on which worker processes the entity.
6. **Time**: only `TimeProvider` and the fixed step; no wall-clock reads in systems.
7. **Iteration order** is the storage order (game-0002); no iteration over `Dictionary`/`HashSet` where order matters.
8. **Replay**: the input log (commands with tick numbers + seed) fully determines the run.

## Consequences

- Good: replay and golden-master tests become possible; bugs are reproducible; scheduling cannot change results.
- Bad: reductions are serial at the end of a phase (small cost); counter-based RNG is slightly less ergonomic; discipline needed in code review.
- Neutral: cross-machine identity is not promised (see Risks); same-machine/same-build identity is.

## Risks

- JIT and CPU differences (FMA contraction, x87 vs SSE, different libm implementations) can change double results across machines. Mitigation: record the golden master per platform if needed, and keep tolerance tests as a second layer. How .NET 10 guarantees (or not) floating-point reproducibility across hardware is unverified here; check official docs at apply time.

## Validation

- **Thread-count independence test**: run 10 simulated seconds (1,200 ticks) of the full park with seed S at 1, 2, 4, 8 and 16 workers; hash all component columns after every 100th tick; all hashes identical (exact equality, not tolerance).
- **Golden master**: committed hash sequence for scenario "octopus 16 gondolas, 600 guests, seed 42"; CI fails on any change, update requires a deliberate PR note.
- **Replay test**: record a session's command log, replay, final state hash equals recorded.
- **Reduction test**: sum of 1,000,000 seeded doubles via the fixed-order reducer returns bit-identical results on 1 vs 16 workers (and demonstrate that a naive parallel sum does not, as a teaching test).
- **Analyser/architecture test**: no `Interlocked` on floating-point types and no `Parallel.*` inside the simulation module.
- **Soak**: 100 repeated runs at random worker counts, zero hash differences.

## References

- docs/adr/game-0002-data-oriented-component-storage.md, game-0003-work-stealing-job-system-without-fibers.md
- Floating-point non-associativity: IEEE 754 general knowledge.
- Gaffer On Games, "Floating Point Determinism" - from memory, not fetched, unverified.
