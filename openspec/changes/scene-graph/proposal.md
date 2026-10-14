# Proposal

## Why

The Octopus ride (change `octopus-motion`) already needs a parent/child transform chain (mill -> arm -> hub -> hub arm -> gondola) with a hub that stays upright while its arm tilts, and the park changes will attach hundreds of nodes (rides, parts, props) to hierarchies. Today each part's pose is computed ad hoc in physics code and the Three.js view keeps its own, unrelated hierarchy (`ride-visualization.ts`). A single scene graph, owned by the simulation, keeps server and client hierarchies identical, gives every part a stable id, and is the foundation for the job system's parallel transform pass (ADR `docs/adr/game-0004-scene-graph-transform-hierarchy.md`, ADR `docs/adr/game-0006-deterministic-parallel-simulation.md`). It builds on the ECS from `ecs-foundation`.

## What Changes

- A scene graph in the `Simulation` module: nodes with stable `NodeId`s, unique sibling names and paths, stored as flat arrays in breadth-first (level) order so that a parent always precedes its children.
- Reparent with cycle prevention, subtree destroy, depth and node-count limits, and ids that stay stable across destroy/create (never reused).
- Double-precision local and world transforms (position, unit quaternion, uniform scale), dirty propagation, one linear update pass.
- The **compensating hub linkage** as a constraint-free orientation mode: a node whose world orientation keeps only the yaw (twist about the up axis) of its parent, so a tilting arm does not tilt the hub.
- Query API: world/local transform of a node, parent, children, find by path, plus `GET /simulation/scene/nodes/{id}`.
- ECS integration: `SceneNode`, `SceneParent`, `SceneName`, `LocalTransform`, `WorldTransform` components and a `TransformSyncSystem`; the graph is a derived cache of those components, so world save/restore and the state hash cover it for free.
- Pose snapshot: `GET /simulation/scene/structure` and `GET /simulation/scene/pose` (local TRS per node id, with a structure version).
- Angular: a `SceneMirror` that builds and updates a Three.js `Object3D` hierarchy from those DTOs, sharing node ids and names/paths with the server graph. It is not wired into `ride-visualization.ts` (that happens in `ride-on-ecs`).
- Tests: example-based plus seeded invariant tests (parent before child, no cycles, ids never reused, equivalence with a naive recursive reference), benchmarks for the update pass.

No existing endpoint, telemetry stream or component changes. `DigitalTwin` and the current 3D view are untouched.

## Capabilities

### New Capabilities
- `scene-node-hierarchy`: Node identity, naming and paths, flat ordered storage, reparenting, destruction, limits.
- `scene-transform-propagation`: Local and world transforms in double precision, dirty propagation, queries.
- `scene-upright-orientation`: The constraint-free upright (compensating) orientation mode used by the hub linkage.
- `scene-ecs-integration`: Scene components, the transform sync system, lifecycle handling and save/restore behaviour.
- `scene-pose-snapshot`: Structure and pose DTOs and their read-only endpoints.
- `scene-client-mirror`: Angular/Three.js mirror of the server graph sharing node ids and paths.

### Modified Capabilities
<!-- None. ecs-foundation capabilities are consumed, not changed. -->

## Impact

- **New code**: `src/Simulation/FourDotnet.BoogaBooster.Simulation/SceneGraph/**`, `Features/GetSceneStructure|GetScenePose|GetSceneNode`, scene DTOs in `Simulation.Abstractions`, tests under `src/Tests/FourDotnet.BoogaBooster.Simulation.Tests/SceneGraph/**`, benchmarks in the existing benchmark project, `src/FourDotnet.BoogaBooster.App/src/app/world/scene/**`, `docs/simulation/02-scene-graph.md`.
- **Edited code (small)**: `SimulationModuleExtensions` (register scene services and system), `SimulationEndpoints` (three read-only routes), `CLAUDE.md` / `.github/copilot-instructions.md` status wording.
- **Dependencies**: none new at runtime. Angular uses the already-installed `three`.
- **Depends on**: `ecs-foundation` (World, command buffer, `IEntityLifecycleObserver`, `ISimulationSystem`, snapshot hash). Consumed by `job-system` (level slices), `ride-on-ecs` (rig definition, pose export) and `octopus-motion`'s later ECS migration.

> **Quality bar:** this change is for a workshop demo, not a production release. See `docs/demo-scope.md` for what may be deferred and what may not.
