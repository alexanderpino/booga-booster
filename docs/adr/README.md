# Architecture decision records (game roadmap)

These ADRs belong to the Octopus + park-simulation roadmap (see ../roadmap.md). The `game-` prefix avoids collision with ADR-0003/0004/0007/0009, which live in the external style-guide MCP server and not in this repo. All are Proposed; the user must accept each one.

| Id | Title | Status | One-line decision |
|---|---|---|---|
| [game-0001](game-0001-ecs-for-simulation-core.md) | ECS for the simulation core | Proposed | New Simulation module is ECS (hand-rolled minimal core behind interfaces, benchmarked against Arch/Friflo); existing modules stay DDD; only DTOs and events cross the boundary. |
| [game-0002](game-0002-data-oriented-component-storage.md) | Data-oriented component storage | Proposed | Archetype tables, SoA columns, fixed-size chunks, 64-bit generational entity ids, deferred structural changes. |
| [game-0003](game-0003-work-stealing-job-system-without-fibers.md) | Work-stealing job system without fibers | Proposed | Per-worker work-stealing deques, dependency counters, stackless coroutine continuations; no fibers (CLR/browser); frontend Workers+SharedArrayBuffer variant optional. |
| [game-0004](game-0004-scene-graph-transform-hierarchy.md) | Scene graph and transform hierarchy | Proposed | Flat arrays, parent indices, topological order, dirty flags; compensating hub linkage as an explicit constraint; feeds Three.js, does not replace it. |
| [game-0005](game-0005-simulation-authority-and-state-sync.md) | Simulation authority and state sync | Proposed | Backend-authoritative; quantised binary snapshots/deltas at 30 Hz with interest management and client interpolation. |
| [game-0006](game-0006-deterministic-parallel-simulation.md) | Deterministic parallel simulation | Proposed | No shared mutation, fixed-order reductions, seeded counter-based RNG, float rules; proven by thread-count independence and golden-master tests. |
| [game-0007](game-0007-asset-pipeline-blender-krita-gltf.md) | Asset pipeline Blender + Krita to glTF | Proposed | Blender/Krita MCP to .glb to GLTFLoader; strict node/pivot naming, texture budgets, provenance manifest, human review gate. |
