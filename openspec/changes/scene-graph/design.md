## Context

`ecs-foundation` provides the entity-component-system core inside the `Simulation` module. `octopus-motion` (apply order 1) makes the four main arms rise and fall and keeps each hub's axis vertical through a compensating linkage, but it computes poses inside DigitalTwin physics and the Three.js view (`ride-visualization.ts`) builds its own boxes with Y-rotation only. As more rides and park objects arrive, hand-rolled pose maths and two unrelated hierarchies (server physics, client scene) would diverge. ADR `docs/adr/game-0004-scene-graph-transform-hierarchy.md` sets the direction: a transform hierarchy as flat arrays with parent indices, updated in topological order. This change implements it in the simulation, integrates it with the ECS, exports a pose snapshot, and adds the client mirror that shares node ids and names.

Conventions fixed here and reused by later changes: right-handed coordinates, +Y up (identical to Three.js, so no conversion on the client), quaternions as (x, y, z, w), angles in radians, doubles on the server, floats only inside Three.js.

## Goals / Non-Goals

**Goals:**
- Flat-array scene graph with parent-before-child order, dirty propagation and stable ids.
- Closed-form compensating hub orientation (no solver).
- Fits ECS: structure and transforms are components; the graph is a rebuildable cache.
- A client `Object3D` hierarchy that shares node ids and paths with the server.
- Update pass that is trivially parallel per level for the job system, with bitwise-identical results.
- Strong invariant tests (order, no cycles, ids never reused, equivalence with a naive reference).

**Non-Goals:**
- No physics, no animation curves, no cam profile (octopus-motion); no rig definition for the Octopus (ride-on-ecs).
- No constraint solver, no inverse kinematics, no non-uniform scale, no shear.
- No wiring into `ride-visualization.ts` (ride-on-ecs).
- No binary/delta pose wire format and no streaming transport (ADR game-0005 and its change); only read-only JSON endpoints.
- No dynamic structure edits from inside systems through a dedicated command API: structure edits made via component values are supported (D6); a richer spawn API arrives with `guests-and-needs` if needed.
- No frustum culling, bounds or spatial index (park-world-grid).

## Decisions

### D1 - Placement: same `Simulation` module, folder `SceneGraph/`

The scene graph is engine-level and depends on ECS types; a separate module would only add a project pair and an artificial boundary. It lives in `Simulation/FourDotnet.BoogaBooster.Simulation/SceneGraph/`, tests in `Tests/.../SceneGraph/`. Pure math types (`Vector3D`, `QuaternionD`, `Transform3D`) are `readonly record struct`s with properties (no public fields) in `SceneGraph/Math/`. Alternative `System.Numerics` was rejected: it is single-precision for `Quaternion`/`Vector3`; there is no double-precision equivalent in the BCL.

### D2 - Two representations with one truth: components own, arrays cache

ECS components (`SceneNode`, `SceneParent`, `SceneName`, `LocalTransform`, `WorldTransform`) are the source of truth. The `SceneGraph` object holds flat arrays (parent index, local, world, dirty flags, names, entity ids) in level order as a **cache** that the `TransformSyncSystem` keeps consistent. Why not keep transforms only in the graph arrays: ECS archetype rows move on structural change, and the whole ECS persistence story (state hash, save/restore, replication) works on components; if the graph were authoritative it would need its own serialisation and could silently diverge from the hash. Cost: transforms exist twice (64 B local + 64 B world in components, same again in arrays) and a compare/copy pass per tick; budgeted by benchmarks S1-S3 below.

*Alternative considered:* make the graph arrays the only storage and expose components as views. Rejected for the reasons above and because it would couple ECS queries to graph internals.

### D3 - Flat arrays in breadth-first (level) order, stable slots

Dense arrays (`parentIndex[]`, `local[]`, `world[]`, `flags[]`, `slotOfDense[]`) are ordered breadth first. A node keeps a **slot** (index into a slot table) for its whole life; `denseOfSlot[]` maps slot to current dense index, and a `Dictionary<long,int>` maps `NodeId` to slot (never iterated, so hash ordering cannot leak into results). Structural changes mark the order dirty and the next update rebuilds the dense order in O(n) with a stable BFS (roots by creation sequence, children by sibling sequence), compacting destroyed slots.

Why BFS instead of DFS pre-order: both give parent-before-child. DFS makes each subtree contiguous (cheap "recompute subtree" ranges) but then the nodes at one depth are scattered. BFS makes each depth a contiguous slice, which is exactly the unit a parallel update needs (all nodes of a level are independent given the previous level; `job-system` can `parallel-for` over a level slice, ADR game-0006). Dirty propagation does not need subtree ranges because it is a single forward pass with `dirty[i] |= dirty[parent[i]]`. Level starts are kept in a small `int[] levelStart`.

Update pass:

```
for i in 0 .. count-1:                       // parents are always at smaller indices
    p = parentIndex[i]
    if (!dirty[i] && (p < 0 || !recomputed[p])) continue
    world[i] = Compose(p < 0 ? Identity : world[p], local[i], mode[i])
    recomputed[i] = true
```

`recomputed` is a transient flag array (also exposed as the count and as the list used for the write-back to `WorldTransform`).

### D4 - Math: position + unit quaternion + uniform scale, not matrices

World = `P.pos + P.rot * (P.scale * L.pos)`, `rot = normalize(P.rot * L.rot)`, `scale = P.scale * L.scale`. Uniform scale means no shear, so TRS composition is closed and cheap (about 50 flops) and quaternions need no re-orthogonalisation. World transforms are *recomputed from locals every time they are dirty*, never accumulated, so floating-point error does not grow with time ("pure function", spec). Quaternions are normalised once per composition. Doubles on the server give about 1e-9 m resolution at 1e7 m (spec scenario); the client receives **effective local** transforms (D8), whose magnitudes are small, so single-precision Three.js keeps sub-millimetre accuracy regardless of world size.

Determinism: the JIT does not contract `a * b + c` into FMA implicitly, and `Math.Sqrt`, `Math.Sin`, `Math.Cos` are called only by builders/tests, not by the update pass. Quaternion composition uses only `*`, `+`, `-`, `Math.Sqrt` (normalise); `Math.Sqrt` is IEEE-exact. Tests compare world transforms bitwise between orderings.

### D5 - Compensating hub linkage: `UprightYaw` mode via swing-twist

Real Octopus rides keep gondola hubs level with a parallelogram linkage. A physical solver is unnecessary: the linkage's *effect* is that the hub orientation ignores the arm's tilt but follows the mill's yaw, while the hub **position** still follows the arm tip. In `UprightYaw`:

`W.rot = normalize(twistY(P.rot) * L.rot)`, position and scale as in `Inherit`.

`twistY(q) = normalize((0, q.y, 0, q.w))` is the twist component of the swing-twist decomposition about +Y. Proof sketch for the rig: `P.rot = qY(phi) * qZ(theta)` (mill yaw then arm pitch about its hinge axis Z) gives `q.y = cos(theta/2) sin(phi/2)`, `q.w = cos(theta/2) cos(phi/2)`; the common factor `cos(theta/2)` cancels in the normalisation, so `twistY = qY(phi)`: the hub's tilt is exactly removed for any pitch. When `q.y` and `q.w` are both about 0 (the parent is flipped 180 degrees about a horizontal axis) the twist is undefined; we define it as identity (deterministic, tested). For pitch about a different horizontal axis the result is the same, because only the Y component of the rotation survives.

*Alternatives:* (a) a general `LookAtUp` constraint solver - rejected, iterative and order-dependent, against ADR game-0006; (b) a per-node "world rotation lock" absolute orientation - rejected, loses the mill-yaw coupling the hub needs; (c) store the compensation as a counter-rotation recorded by gameplay code (`local.rot = inverse(arm tilt)`) - rejected, every ride author would re-implement and get it wrong. The mode costs one extra branch in the compose step.

The rig assumption "arm pitches about its local Z with the mill's hinge axis horizontal" is an `octopus-motion` convention; see Risks.

### D6 - Structure edits: immediate between ticks, by component value during ticks

- Between ticks (setup, tests, loading): `SceneBuilder`/`SceneGraph.Create/Reparent/Rename/Destroy` operate immediately (and throw during an iteration scope).
- During a tick: systems change `SceneParent`, `SceneName`, `SceneNode.Mode`, `LocalTransform` values with ordinary in-place sets, and spawn/destroy node entities with the ECS command buffer (new entities carry `SceneNode{NodeId=0}`). The `TransformSyncSystem` reconciles at the end of its run: ids assigned in ascending entity index order (so ids are predictable and independent of archetype layout), parent/name/mode changes validated (cycle, duplicate sibling name, missing parent, limits), invalid changes **reverted in the component** and counted, never thrown (one bad system must not stop the tick).

This keeps a single mechanism (components) for editing, saving and replicating structure.

### D7 - Lifecycle: observer cascade

`SceneGraph` implements `IEntityLifecycleObserver` from `ecs-foundation`. On `OnEntityDestroyed(e)` for a node entity it removes the node and records destroy commands for the entities of all descendants into the playback buffer (Design D6 of ecs-foundation guarantees the cascade finishes in the same playback). Because subtree membership is defined by the graph cache, the cache is reconciled at the start of each sync, so even a missed notification is repaired on the next tick (belt and braces; a test removes the observer and checks the repair).

### D8 - Pose export: effective local transforms and a structure version

Two documents, split by change frequency: **structure** (ids, parent ids, names, paths; changes rarely) and **pose** (local TRS per id, every snapshot). `structureVersion` ties them: a pose for a different version is rejected by the client, which refetches structure. Poses are `nodeIds[]` plus a flat `double[]` with stride 8 (typed-array friendly), in dense order, which also means parents precede children on the wire (the client builds incrementally).

The client has no notion of orientation modes; to keep it that way the server ships the **effective local rotation** for upright nodes, `conjugate(P.rot) * W.rot`, so Three.js composing plain parent-child matrices reproduces the server world transform. Position and scale are unchanged. This is the single place where server-only semantics are folded into data.

Sending local rather than world transforms: matches Three.js's own composition (`matrixWorld`), keeps magnitudes small for float32, and stays correct if the client reparents for effects. Cost: the client must receive the full structure; fine at this scale, deltas come with ADR game-0005.

### D9 - Client mirror: `SceneMirror`, plain TypeScript class

Location `src/FourDotnet.BoogaBooster.App/src/app/world/scene/`: `scene-snapshot.models.ts`, `scene-snapshot.reader.ts` (validation, same style as the ECS snapshot reader), `scene-mirror.ts`. The mirror owns a `Group` root, a `Map<number, Object3D>` and a `Map<string, number>` path index. `Object3D.name` is the node name, `userData.nodeId` the server id, so debugging in the browser tools shows both. Updates never recreate unchanged objects (stable references matter for materials/meshes later attached by ride-on-ecs and asset-pipeline: attaching a glTF mesh to the node object keeps working across structure updates). It does not touch `ride-visualization.ts`; adoption is a task of `ride-on-ecs`. Three.js is already a dependency; the pinned version (0.169 at time of writing) is read from `package.json` at apply time, no new library is introduced, so no currency check is needed beyond confirming the `Object3D`, `Quaternion`, `Group` APIs used are not deprecated in the pinned version.

### D10 - Tests and invariants (teachable)

Example tests quote the numbers in the specs (the 30 degree pitch, 90 degree yaw case gives hub at (0, 3, -5.196152422706632); written once, reused in server tests and client tests so both sides agree on one fixture: `scene-fixtures.json`, hand-copied into the tests with a comment pointing to the spec scenario).

Seeded random-operation tests (plain `Random(seed)`, 200 seeds; same approach and justification as `ecs-foundation` D12; FsCheck stays a candidate behind a currency check):

- `Order_invariant`: after any random sequence of create/destroy/reparent/rename, `parentIndex[i] < i` for all i, level starts partition the array, siblings are in sibling-sequence order.
- `No_cycles`: walking up from every node reaches a root in at most `MaxDepth` steps; no rejected operation changed the graph (hash of structure before == after).
- `Ids_never_reused`: the set of all ids issued across a run has no duplicates; surviving nodes keep their ids and names.
- `Matches_naive_reference`: world transforms equal a recursive reference within 1e-9 for random trees with random locals and random modes.
- `Pure_function`: building the same final structure by two different operation sequences gives bitwise-identical worlds.
- `Rebuild_equals_live`: `Rebuild(world)` after random ops equals the live graph.
- `Level_order_independent`: computing each level in reverse index order equals sequential.

### D11 - Performance budgets (targets, measured at apply time)

Added to the benchmark project from `ecs-foundation`: S1 update of 10 000 nodes, all dirty, depth up to 5: <= 0.6 ms (about 60 ns per node; one compose is about 50 flops plus 128 B of array traffic, so 0.3 ms is the floor and 0.6 ms leaves 2x); S2 sync with nothing changed, 10 000 nodes (compare-only pass over components): <= 0.25 ms; S3 sync with 1 % locals changed: <= 0.3 ms; S4 pose projection of 10 000 nodes to the DTO arrays: <= 1.5 ms; S5 `Rebuild` of 10 000 nodes: <= 5 ms. Per tick, S2 + S1-style work for a ride-sized graph (about 100 nodes) is negligible; the budgets matter for the park (thousands of attached nodes). Timing is not asserted in unit tests; results over 2x a budget stop the change. Allocation: the update pass and the compare pass allocate nothing (unit test asserting allocated bytes, as in ecs-foundation).

## Dependencies and contracts

**Requires from `ecs-foundation` (intent, names may differ slightly at apply):** `World`, `ComponentRegistry.Register<T>(name, schemaVersion, replication)`, `EntityCommandBuffer`, `IEntityLifecycleObserver` (with the playback buffer), `ISimulationSystem`/`SystemAccess`/`SystemContext`, `Query`/`Chunk`, canonical hash and `WorldSerializer` (for the restore test), `BoogaBoosterTelemetry` metrics pattern, test harness `RandomOperationSequence` style, benchmark project, `SimulationModuleExtensions`/`SimulationEndpoints`.

**Requires from `octopus-motion`:** nothing at code level. Convention-level: right-handed +Y-up frame, arm length 6 m, hub arm length 2 m, main-arm pitch about the arm's hinge axis; see Risks.

**Provides to later changes:**
- `job-system`: `SceneGraph.LevelStarts` / `LevelCount` and `ComputeLevel(level, from, to)` so workers can update slices of one level; invariant `Level_order_independent` as the equivalence test.
- `ride-on-ecs`: `SceneBuilder` (fluent rig definition: `Root("greatMill").Child("arm0", Transform3D...).Child("hub0", mode: UprightYaw)`), components `LocalTransform`, `WorldTransform`, `SceneNode`, `SceneParent`, `SceneName`, query API `TryGetWorldPosition(NodeId)`, pose DTOs, `SceneMirror` and its reader to replace ad hoc `ride-visualization.ts` pose code, the numeric fixtures.
- `asset-pipeline`: stable node names/paths as attachment points for glTF meshes (a glTF node named like a server node binds to it), `SceneMirror.findByPath`.
- Park changes: attach/detach by changing `SceneParent`; `MaxNodes` option.
- HTTP: `GET /simulation/scene/structure`, `GET /simulation/scene/pose`, `GET /simulation/scene/nodes/{id}`.
- Abstractions DTOs: `SceneStructureResponse`, `SceneNodeDto`, `ScenePoseResponse`, `SceneNodeResponse`.
- Metrics: `boogabooster.scene.*` (spec `scene-pose-snapshot`).

## Risks / open questions

- **Rig axis convention vs `octopus-motion`.** This design assumes the main arm pitches about its local Z (horizontal hinge, arm length along local +X) and the mill rotates about +Y. If `octopus-motion` chooses a different hinge axis or a Z-up frame, `UprightYaw` still holds for any horizontal hinge axis (only the Y component of the parent's rotation survives) but the fixtures change; conflict must be flagged in the apply PR and resolved by changing the fixtures, not the maths. A Z-up frame would be a conflict with Three.js and is not assumed anywhere.
- **Upright mode on a node whose parent *yaws and rolls in non-trivial combination*** (twist is only the Y component): intended; documented in the doc with the 180-degree-flip edge case.
- **Effective-rotation round trip.** The server sends `conjugate(P.rot) * W.rot`; the client composes in float32. Accumulated float32 error over a 5-level chain is below 1e-5 in the fixtures; a deeper chain could drift visibly. `MaxDepth` 64 is a limit, not a promise of client accuracy; open question whether the client should receive worlds for deep rigs. Not needed for the planned rigs (depth <= 5).
- **Cost of two copies of transforms** (components + arrays) at 10 k nodes is about 2.5 MB; acceptable. If the park needs 100 k+ nodes the compare pass becomes the bottleneck and a change-version mechanism in the ECS (not present in `ecs-foundation`) would be needed.
- **System ordering.** Gameplay systems that write `LocalTransform` must be registered before `scene.transform-sync`, otherwise their changes apply one tick late; systems that read `WorldTransform` after registration of the sync see this tick's value. Documented in the doc and tested with a scheduler plan assertion.
- **Reverting invalid component values in place** is surprising for system authors (their write is undone). The counter `boogabooster.scene.rejected{reason}` and the doc section "Why did my parent change not stick?" mitigate; strictness (throwing) was rejected because one buggy system would stop the simulation.
- **Names in components** are fixed 31-byte ASCII (`[InlineArray]`), which excludes non-ASCII names. Acceptable for rig part names; a limitation to state in the doc.
- **Hash ordering of the id dictionary.** `Dictionary<long,int>` is used only for lookup; a test iterates the graph through dense arrays only. A code-review checklist item in the doc prohibits iterating the dictionary.
- **ADR drift.** ADR game-0004 is written in parallel; if it mandates DFS order or world-space snapshots, update D3/D8 before apply and record the change in the PR.
- **Handedness and Z-fighting of conventions between Blender (Z-up) and Three.js** is the asset pipeline's concern (glTF is Y-up by specification), not this change's.
- **Open question for the user:** are node names allowed to be user-visible (park objects named by players)? Assumed no: names are rig identifiers; display names live elsewhere.
- **Currency checks:** no new third-party library is introduced here; the benchmark library and any property-testing library fall under `ecs-foundation`'s checks.
