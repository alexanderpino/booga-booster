# game-0002: Data-oriented component storage (SoA, archetypes, generational ids)

- Status: Proposed (the user must accept)
- Date: 2026-10-14
- Deciders: project owner
- Related: game-0001, game-0003, game-0006

## Context

The simulation core (game-0001) iterates the same few fields of thousands of entities every 1/120 s tick: gondola angle and angular velocity, guest position and need values, scene node transforms. Memory layout decides throughput more than algorithm choice at this scale. Entities are also created and destroyed constantly (guests arrive and leave), so ids must be safe against stale references, and iteration order must be deterministic (game-0006).

## Decision drivers

1. Cache-friendly sequential access for hot loops.
2. Cheap add/remove of entities and components.
3. Safe handles: a stale id must never silently address a recycled slot.
4. Deterministic iteration order, independent of thread count.
5. Simple enough to explain on a whiteboard.
6. Splittable into independent ranges for jobs (game-0003).

## Considered options

### Option A: Archetype tables with chunking
Entities with the same component set live in one table; columns per component (SoA), split into fixed-size chunks (for example 16 KB, a size Arch uses per its README; ours is tunable).
- Pros: dense iteration, queries match whole tables, chunks are natural job units, best locality for "entities with exactly these components".
- Cons: adding/removing a component moves the entity between tables (copy cost); more bookkeeping; fragmentation across many archetypes.

### Option B: Sparse sets per component
Each component type has a dense array plus a sparse index by entity id.
- Pros: add/remove is O(1) without moving other components; simple; easy to explain.
- Cons: multi-component queries must intersect sets and use indirection, so loops are less sequential; iteration order follows the smallest set and changes with insertion history unless sorted.

### Option C: Array of structs (class/struct per entity)
- Pros: trivial.
- Cons: pulls unused fields into cache lines; object overhead; poor SIMD; worst for the showcase.

## Decision

Use **archetype tables with SoA columns and fixed-size chunks** as the default storage, with these rules:

1. **SoA**: one `T[]` per component per chunk, components are unmanaged `struct` types (no references), which also makes snapshotting (game-0005) a memory copy.
2. **Chunking**: chunk capacity is derived from a constant byte budget divided by the archetype row size (constant in the module's constants file, no magic numbers). A chunk is the unit of work handed to a job.
3. **Entity id**: 32-bit index + 32-bit generation (`Entity` is a 64-bit struct). Destroying bumps the generation; every lookup validates it. A free list recycles indices in LIFO order, which is deterministic for a given command sequence.
4. **Structural changes are deferred**: systems record spawn/destroy/add/remove into a per-worker command buffer; buffers are applied at a sync point in fixed worker-index then record order (game-0006). No structural change inside a running query.
5. **Hot toggling** (for example "is queuing") that would cause archetype churn uses a boolean/enum field in a component, not an add/remove.
6. **Iteration order** is: archetypes by creation order, chunks by index, rows by index. Compaction after destroy moves the last row into the hole (swap-remove) and is applied at the sync point only.
7. Sparse sets remain an optional technique for rare components (for example `Selected`), not the main storage.

## Cache arguments

A 64-byte cache line holds 16 floats. Updating angle for 16 gondolas reads one or two lines in SoA; in AoS with a 64-byte object it reads 16 lines and discards most bytes. At 5,000 guests x 8 hot bytes = 40 KB, a position/velocity pass fits in L1/L2 in SoA form (typical L1 data 32-48 KB; machine-dependent, verify with the benchmark). These are reasoning figures, not measurements; the Validation section measures them.

## How determinism is preserved

- Row order is a function of the command sequence only, not of timing: deferred commands apply in a defined order, swap-remove is deterministic.
- Entity ids come from a single-threaded allocator at the sync point; workers never allocate ids.
- Jobs read/write disjoint chunk ranges; reductions follow game-0006.

## Consequences

- Good: fast linear loops, chunk = job unit, snapshots are memcpy-able, deterministic order.
- Bad: structural changes cost a row copy; more complex than sparse sets; unmanaged-only components restrict modelling (strings go to side tables).
- Neutral: component limit or archetype count must be bounded (set a cap, fail loudly).

## Validation

- Benchmark: Position+Velocity pass over 10,000 entities, SoA archetype vs a naive class-per-entity baseline: >= 5x faster (target; measured on CI machine, report actual).
- Property test (FsCheck-style or Bogus-seeded loop): random spawn/destroy/add/remove sequence with a fixed seed yields an identical row order and ids across 2 runs and across 1/4/16 workers.
- Stale handle test: destroy then recycle index; old `Entity` fails validation 100% of the time.
- Allocation test: steady-state tick allocates 0 bytes on the GC heap (verified with `GC.GetAllocatedBytesForCurrentThread`).

## References

- docs/adr/game-0001-ecs-for-simulation-core.md
- https://github.com/genaray/Arch (archetype and 16 KB chunk claim, read 2026-10-14)
- Cache-line size and L1 figures: general hardware knowledge, unverified against a specific CPU.
