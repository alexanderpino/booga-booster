## Context

Today all simulation lives in `DigitalTwin`: DDD aggregates (`GreatMill`, `Hub`, `Gondola`) advance a planar physics model at a fixed 1/120 s step, deterministic, with time and randomness injected. That is right for 21 bodies. The roadmap (see ADRs `docs/adr/game-0001-ecs-for-simulation-core.md` .. `game-0007-...`) adds an Octopus ride, a scene graph, a job system and then a park with many rides and thousands of guests. The new simulation core is an ECS with data-oriented (structure-of-arrays) storage, in a **new** module; existing modules keep DDD and are not touched.

Repo constraints that shape the design: modular monolith with module + `.Abstractions` (CLAUDE.md "Module anatomy"), endpoints without logic and handlers with `ExecuteAsync`, one shared `BoogaBoosterTelemetry` meter/source, no public fields (STJ silently drops them), `TimeProvider` for time, xUnit v3 + Moq + Bogus (no FluentAssertions), >= 80 % line coverage on module libraries, Angular 22 zoneless/signals. Teachability is a first-class goal: every concept below maps to a small file and a test that names the rule.

## Goals / Non-Goals

**Goals:**
- A correct, small, measured ECS: ids with generations, component registry, archetype SoA storage, queries, command buffer, system scheduler, fixed tick driver.
- Determinism by construction: same operations give same ids, same iteration order, same state hash, on every run.
- Contracts the job system (`openspec/changes/job-system`) can parallelise without changing results (ADR `docs/adr/game-0003-work-stealing-job-system-without-fibers.md`, `docs/adr/game-0006-deterministic-parallel-simulation.md`).
- A snapshot/serialization hook usable for state sync later (ADR `docs/adr/game-0005-simulation-authority-and-state-sync.md`).
- Measurable performance budgets and allocation-free iteration.

**Non-Goals:**
- No parallel execution (single-threaded executor only), no job system.
- No scene graph or transforms (change `scene-graph`).
- No migration of `DigitalTwin` physics (change `ride-on-ecs`).
- No delta compression, no network transport, no client prediction (ADR game-0005 decides the sync model; this change only provides the hook).
- No client-side ECS in Angular.
- No managed-reference components, no component inheritance, no entity hierarchies (scene-graph owns hierarchy).
- No reflection-based auto-discovery of components or systems.

## Decisions

### D1 - Module `Simulation`, folders `Ecs/` not `Domain/`

New module at `src/Simulation/FourDotnet.BoogaBooster.Simulation` and `.Abstractions`; tests at `src/Tests/FourDotnet.BoogaBooster.Simulation.Tests`; benchmarks at `src/Benchmarks/FourDotnet.BoogaBooster.Simulation.Benchmarks`. The name is `Simulation` rather than `World` or `Game` because the park changes will introduce a `ParkWorld` grid concept (`park-world-grid`) and "World" would collide with it; the ECS container class inside is still called `World` (`Simulation.Ecs.World`). The engine code lives in `Ecs/` (and later `SceneGraph/`, `Jobs/`) rather than `Domain/`: an ECS is infrastructure-grade engine code, not a DDD bounded context, and forcing aggregates onto it would defeat SoA storage. `Features/`, `Endpoints/`, `Application/` and `SimulationModuleExtensions` follow the standard anatomy. The deviation is stated in the doc and the task list consults the style-guide MCP server before folder creation; if the server objects, the folder name changes, the design does not.

*Alternative:* put the ECS in `Shared/`. Rejected: Shared must not hold module logic and nothing else needs the ECS except through the module.

### D2 - Entities: `EntityId(int Index, int Generation)`, LIFO free list

`EntityId` is a `readonly record struct` (properties, not fields). Generation starts at 1; 0 means "never alive" so `default(EntityId)` is safely dead. Free slots form a LIFO stack: reuse is deterministic and cache-friendly (hot slot first). A slot at `int.MaxValue` generation is retired. `MaxEntities` defaults to 1 048 576 (2^20, 10x the 100 k target) and is an option; the slot table is a growable array of `{Generation, ArchetypeId, Row}`.

*Alternative:* 64-bit packed id. Rejected: no benefit here, harder to read in tests; the JSON snapshot carries `index` and `generation` as two numbers anyway.

### D3 - Storage: archetype structure-of-arrays (decision per ADR game-0001 / game-0002)

Two mainstream designs satisfy "ECS with SoA":

| | Archetype SoA (chosen) | Sparse set per component |
|---|---|---|
| Iterate N entities with 3 components | One linear pass over 3 dense arrays per matching archetype. Best locality, SIMD-friendly, trivially chunked. | Iterate the smallest set, then random-probe the others (indirection per component). |
| Add/remove a component | Move the entity row between archetypes (copy all columns). O(components). | O(1), no move. |
| Query matching | Match archetypes by component set (cached), cost per archetype. | Intersect sets per query. |
| Iteration order | Archetype creation order then row order: easy to make deterministic. | Order of the driving set; changes with swap-remove in each set independently. |
| Partitioning for jobs | Slice rows of a chunk: disjoint by construction. | Slices of the driving set, but other sets are shared random reads. |
| Fragmentation risk | Many component combinations produce many small archetypes. | None. |

The workload is "many entities, few structural changes per tick, many read/write passes per tick" (guests moving, rides animating). Iteration and parallel partitioning dominate, structural change is the exception (and is deferred and batched through the command buffer anyway). That points to archetypes. The weak spot (frequent add/remove of a flag) is handled by tag components that still move rows but are cheap, and is measured by benchmark B4. If B4 or B3 miss budget by 2x the change stops for review (spec `ecs-performance-budget`); sparse sets stay the documented fallback for high-churn components.

Layout: one `Archetype` per distinct sorted set of `ComponentTypeId`. Each archetype owns `EntityId[]` plus one `Column<T>` (a `T[]` with count and capacity) per data component. Growth doubles capacity. There is no hard chunk allocation; "chunks" are *views* of at most `ChunkRows = 1024` consecutive rows (1024 x 56 B = 56 KB for the 3-component benchmark entity, within L2; the number is a tuning constant in `SimulationParameters`, justified and re-measured in the benchmark task). Views, not separate allocations, keep growth simple now; moving to true fixed-size memory chunks later does not change the query API.

Components are `unmanaged` structs, written with auto-properties (`public double X { get; set; }`), never public fields. Systems access them by `Span<T>`/`ref T`, so property inlining by the JIT makes this free, and the same structs serialise via raw bytes (D9), not STJ.

### D4 - Component registry: explicit, ordered, named

`ComponentRegistry.Register<T>(name, schemaVersion, replication?)` assigns `ComponentTypeId` = registration order and freezes on first `World` creation. Stable string names are the snapshot identity (binary table and JSON), so renaming a C# type never breaks a snapshot. No assembly scanning: order of registration is visible code, hence deterministic. Hard limit 256 types (a `byte`-sized id is not required, the limit just bounds the archetype key). `RegisterTag(name)` for zero-data markers.

### D5 - Queries: description object, cached archetype list, ref-struct chunks

A `QueryDescription` (required read/write, `With`, `Without`) is compiled once into a `Query` that caches the matching archetype list and re-matches only when the world's `ArchetypeVersion` changed. Iteration yields `Chunk` ref structs exposing `ReadOnlySpan<T> Read<T>()`, `Span<T> Write<T>()`, `ReadOnlySpan<EntityId> Entities`, `Count`. `Query.ChunkCount`/`GetChunk(i)` give the indexable form for the job system. Plain `foreach` over a `ChunkEnumerator` struct avoids delegates and allocation. In a system, a `QueryHandle` created from the system's *declared* access validates spans against the declaration (D7).

### D6 - Structural changes: immediate API outside ticks, command buffer inside

`World.CreateEntity/Destroy/Add/Remove` throw while an iteration scope is open. During a tick systems use `ctx.Commands` (an `EntityCommandBuffer`): ops are appended to a typed op list plus a byte arena for component payloads (components are unmanaged, so the arena is a plain `byte[]` with alignment handled by `MemoryMarshal`). Provisional ids use generation 0 and the op index as `Index`, resolved through a map during playback. Playback happens once, **after the last system of the tick**, ordered by (system order index, sequence). A single sync point is simpler to teach and to parallelise (each worker owns a buffer, merging is a stable sort by key), at the price that created entities become visible one tick later; at 120 Hz that is 8.3 ms and acceptable. A later change may add explicit sync points. Playback never throws for stale targets; it discards and counts (spec). `IEntityLifecycleObserver` hooks let other subsystems (scene graph) react; observers receive the playback buffer so cascades happen within the same playback and the loop runs until the queue is empty.

### D7 - System interface and scheduler

```
public interface ISimulationSystem
{
    string Name { get; }
    SystemAccess Access { get; }          // Reads, Writes, UpdateAfter, UpdateBefore
    void Update(in SystemContext context);
}
```

`SystemContext` is a readonly ref-free struct holding `World`, `FixedStepSeconds`, `Tick`, `Commands` and the validated query factory. The `SystemScheduler` builds the plan (spec: edges from component conflicts in registration order, explicit constraints, Kahn's algorithm with registration-order tie-break) and exposes `ExecutionPlan.Stages`. `ISystemExecutor` has one implementation now, `SequentialSystemExecutor`, which flattens the stages in order. The job-system change adds a parallel executor that runs a stage's systems concurrently and, within a system, chunk ranges concurrently, and must pass the same determinism tests (executor equivalence is a spec requirement: "any executor SHALL produce the same world state as the sequential one"). Declared access is validated in Debug/tests and trusted in Release.

*Alternative:* attribute-based access declaration with reflection. Rejected: the scheduler would depend on reflection order; explicit properties are testable and AOT-friendly.

### D8 - Fixed tick driver without drift

Tick counts are derived with integers, never by accumulating `TimeSpan` (1/120 s = 83 333.33 ticks of 100 ns; not representable, drift would accumulate). The driver stores `startTimestamp` and `ticksRun`, and on `Advance()`:

```
elapsed = TimeProvider.GetTimestamp() - startTimestamp          // long, timestamp units
due     = (Int128)elapsed * FixedStepHz / TimeProvider.TimestampFrequency
toRun   = min(due - ticksRun, MaxCatchUpSteps)
if (due - ticksRun > MaxCatchUpSteps) { dropped = ...; startTimestamp += dropped * frequency / 120 (carry the remainder exactly) }
```

Baseline shift on drop uses the same integer arithmetic so that the *next* step boundary is exactly one step after the last executed one (scenario "Long stall"). `MaxCatchUpSteps = 8` caps one `Advance()` at 8 x 8.33 = 66.7 ms of work so a stall cannot cause a death spiral; value is a documented constant in `SimulationParameters`. The hosted `SimulationBackgroundService` uses `TimeProvider.CreateTimer` with period 1/120 s; the OS timer is coarse (about 15.6 ms granularity on Windows by default), which is exactly why the loop catches up by counting, not by assuming one tick per wake-up. Sim time is `tick / 120.0`.

`SimulationParameters.FixedStepHz` duplicates the physics step constant in `DigitalTwin`. Modules must not reference each other's internals, so a contract test in `ride-on-ecs` (when DigitalTwin moves onto the ECS) asserts equality; until then the doc names both locations.

### D9 - Snapshot hook: canonical hash, binary save/restore, JSON projection

Three separate things, one concern each:
1. **Canonical hash** (FNV-1a 64, own 15-line implementation to avoid a dependency; `System.IO.Hashing` is a candidate for later and would need the currency check). Iterates entities by ascending slot index and components by ascending type id, so it is independent of row order. It is the determinism test oracle for this change, `scene-graph`, `job-system` and `ride-on-ecs`.
2. **Binary save/restore** (`WorldSerializer`): little-endian, `BBWS` magic, format version, tick, slot table including free-list order, component table (name, size, schema version), then entity data in slot order. Restores into a *new* `World`; validates everything before building. This makes replay, rewind-for-tests and save games possible, and a future full-state resync message.
3. **JSON projection** of replicated components (`ReplicationSchema`: field names plus `double` accessors). It is what the Angular reader consumes. The wire for production sync (binary, deltas) is decided by ADR game-0005 and its change; ECS exposes `Tick` and per-world `StructureVersion` so deltas can be added.

STJ is used only for the DTO, whose members are properties of records; component structs are never serialised by STJ.

### D10 - Diagnostics

Metrics on `BoogaBoosterTelemetry.Meter` (names in spec `simulation-diagnostics`). Per-system timing takes timestamps only when `Histogram.Enabled` is true. No per-tick `Activity`: tracing at 120 Hz would be pure overhead and drown real traces. A `simulation.snapshot` activity covers snapshot capture and save/restore. `GetSimulationDiagnostics` and `GetWorldSnapshot` are query handlers deriving from `QueryHandler<,>`, so the standard handler metrics apply. The snapshot handler asks the tick loop for a between-ticks window (a lock held by the loop for the duration of a tick; the handler waits for at most one tick).

### D11 - Benchmarks

`BenchmarkDotNet` is a **candidate** and must pass the currency check at apply time (not deprecated in the pinned version, official docs name no different default, upstream not archived and released within 12 months); version is chosen then, not here. Budgets (spec `ecs-performance-budget`):

- B1 100 k entities x 3 components `position += velocity * dt`: <= 1.0 ms mean, aspiration 0.3 ms. Derivation: the data set is 100 k x 56 B = 5.6 MB touched (read 56 B, write 24 B per entity); a modern core streams about 10 GB/s from L3/RAM, so about 0.5 ms is the memory floor and 1.0 ms leaves 2x for loop overhead. The tick budget is 8.33 ms, so one full pass at the limit is 12 %; a realistic tick has several passes plus the scene-graph update, which is the reason the limit is not looser.
- B2 10 k entities: <= 0.1 ms (linear scaling, fits cache).
- B3 create 100 k x 3 components via command buffer: <= 25 ms (250 ns per entity: arena append + archetype append + slot table write). Not per tick, load-time cost.
- B4 10 k destroy + 10 k create in one playback: <= 3 ms (150 ns per op). This is the realistic per-tick churn ceiling (guests entering/leaving).
- B5 canonical hash 100 k: <= 5 ms; B6 binary save 100 k: <= 15 ms.
- 0 B/op for B1, B2. The machine-independent version of that is a unit test.

These are *targets to be measured*, not facts; the apply run records CPU model, runtime version and results in `docs/simulation/01-ecs-foundation.md`. Timing is never asserted in unit tests (flaky across machines). Over 2x budget stops the change (storage review).

### D12 - Testing strategy (teachable, test-first)

Every task writes the failing test first and names the rule in the test name. Besides example-based tests there is a small **seeded random-operation harness** (`RandomOperationSequence` in the test project): N seeds (default 200) x M random operations (default 500) over create/destroy/add/remove/set, with a plain `Random(seed)`; on failure the assertion message prints the seed so the failing run is a one-line reproduction. Invariants checked after every operation (or every 25th for speed):
- every live id resolves; every stale id does not (generation rule);
- `EntityCount` equals live slots; no slot is both live and on the free list;
- each archetype's `Entities[row]` points back to a slot whose `Row == row` (location table consistent);
- after destroy+create the *new* id differs from every previously issued id (stable ids: no id is ever issued twice);
- the canonical hash of the world equals the hash of an independent reference model (a `Dictionary<EntityId, ...>` mirror) rebuilt after restore;
- same seed twice gives the same hash (determinism).
This hand-rolled harness is chosen over adding FsCheck (shrinking is not needed for this size, a dependency is avoided, and a failing seed is itself a minimal reproduction). `FsCheck.Xunit.v3` stays a candidate: passing the currency check at apply time is the precondition, and adopting it would be a separate decision.

### D13 - Angular: thin typed reader only

Decision: the client gets `parseWorldSnapshot` and a `readComponent` helper, nothing else. Reasons: the backend is authoritative (ADR game-0005 status Proposed, to be confirmed), so the client renders state it is given; a client ECS would duplicate component storage and the scene graph for no consumer; JS lacks structs and deterministic parallelism so SoA parity would buy nothing; and the Three.js scene graph already is the client's hierarchy (change `scene-graph`). If client-side prediction is later chosen in game-0005, that is a new change with its own design. The reader lives in `src/FourDotnet.BoogaBooster.App/src/app/world/snapshot/` as plain TypeScript (no Angular dependencies, easy to test with Vitest), with strict validation so a server/client schema drift fails loudly with a path (`entities[0].values.position`).

## Dependencies and contracts

**Requires from earlier changes:** nothing from `octopus-motion`. Reuses existing: `BoogaBoosterTelemetry` (Shared/Core), `QueryHandler<,>`/`IQueryHandler` (Shared/Core Cqrs), `DomainValidationException` (not needed in ECS internals; used by handlers), `ServiceDefaults` (meter already registered by name).

**Provides to later changes (names are intent; apply may refine spellings but keeps semantics):**
- `scene-graph`: `IEntityLifecycleObserver`, `EntityCommandBuffer`, component registration with replication schema, `ISimulationSystem`/`SystemAccess`, canonical hash, `JSON snapshot` convention, `Simulation` module project layout and test harness.
- `job-system`: `ExecutionPlan.Stages`, `ISystemExecutor`, `Query.ChunkCount/GetChunk(i)`, command-buffer merge key `(systemOrderIndex, sequence)` and per-worker buffers, executor-equivalence test fixture (`WorldEquivalenceAssert`), benchmark project to extend.
- `ride-on-ecs`: component and system conventions, `FixedTickDriver`, `SimulationParameters`, save/restore for regression "golden" worlds.
- Park changes: entity/component API, archetype storage, budgets B1-B6.
- Types in `FourDotnet.BoogaBooster.Simulation.Abstractions` (public contract): `WorldSnapshotResponse`, `SimulationDiagnosticsResponse`, `ReplicationFieldDto`. Engine types (`World`, `Query`, ...) are public in the module project, and other modules that need them may reference the module project only through an explicit exception recorded in a later ADR; the default is that gameplay modules add systems via `AddSimulationSystem<T>()` from the `Simulation.Abstractions` contract interfaces (`ISimulationSystem`, `SystemAccess`, `SystemContext`, component registration interfaces live in `.Abstractions` for this reason).
- HTTP: `GET /simulation/diagnostics`, `GET /simulation/snapshot`.
- Angular: `parseWorldSnapshot`, `readComponent`, `WorldSnapshot` types.

## Risks / open questions

- **Module and folder naming (`Simulation`, `Ecs/`)** is a proposal; confirm in review. Style-guide MCP server may prefer `Domain/`. Cheap to rename before apply.
- **Where the engine types live.** Putting `ISimulationSystem` & friends in `.Abstractions` makes other modules depend only on that project (CLAUDE.md rule) but forces `World` access types there too (a system touches `World`/`Chunk`). Likely outcome: most of the ECS public surface sits in `.Abstractions`, module project holds implementation and hosting. Decide at apply task 1.3 with the style guide; record in the doc.
- **Archetype fragmentation** with many tag combinations (park changes). Mitigation: measure archetype count in the diagnostics; sparse-set fallback documented.
- **Memory growth** by doubling arrays for 1 M entities x big components; `MaxEntities` and a capacity option bound it. Large arrays go to the LOH (85 000 bytes): 100 k x 24 B = 2.4 MB arrays live there and are never freed in steady state; acceptable, noted for the job-system pooling task.
- **Floating-point determinism across CPU architectures** (x64 vs ARM64): .NET does not contract `a*b+c` into FMA automatically, and scalar `double` ops are IEEE, so results match; explicit `Vector<T>`/`Math.FusedMultiplyAdd` use would break parity and is forbidden by `ride-on-ecs` conventions unless both paths are tested. Only x64 is tested in CI.
- **Timer granularity** on Windows hosts can make rendering of 120 Hz bursty; the catch-up cap trades fidelity for stability. Open: whether to request high-resolution timers in the host. Not needed for correctness.
- **One-tick latency of structural changes** (D6). If gameplay needs same-tick creation, add explicit sync points in a later change.
- **Snapshot between ticks (D10)** introduces a lock between the HTTP thread and the loop; the handler may wait up to one tick (about 8 ms worst case, longer if a tick overruns). If ADR game-0005 chooses push streaming, replace with a copy-on-tick double buffer.
- **ADR drift.** The ADRs game-0001..0007 are being written in parallel; if one contradicts D3, D6 or D8, this design is updated before apply (record in `Decisions` of the apply PR).
- **Third-party currency checks** (BenchmarkDotNet, optionally FsCheck, `System.IO.Hashing`) must run at apply time; no versions are asserted here.
- **AI tooling sync.** Adding the `Simulation` row to CLAUDE.md requires mirroring `.github/copilot-instructions.md` and running the `ai-tooling-sync` check.
- **Open question for the user:** is `MaxEntities` default 2^20 sufficient for the park (target agents < 100 k)? Assumed yes.
