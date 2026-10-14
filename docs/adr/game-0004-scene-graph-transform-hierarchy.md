# game-0004: Scene graph as flat transform arrays with parent indices

- Status: Proposed (the user must accept)
- Date: 2026-10-14
- Deciders: project owner
- Related: game-0001, game-0002, game-0007

## Context

The ride is a hierarchy: Great Mill -> 4 main arms -> 4 hubs -> hub arms -> 16 gondolas. With the Octopus change (roadmap 1) the main arms also rise and fall, while each hub keeps a vertical axis through a compensating linkage: hub position = (L cos(theta), L sin(theta)) with the hub plane staying horizontal. Parks add rides, queues and guests as further nodes. Both the backend (physics, positions) and the Three.js client (rendering) need consistent world transforms, and the backend must compute them deterministically (game-0006).

## Decision drivers

1. Cache-friendly, job-parallel transform update (game-0002/0003).
2. Express the compensating hub linkage without hacks.
3. Same mental model as Three.js and glTF so assets map (game-0007).
4. Deterministic evaluation order.
5. Cheap when little changes (most park nodes are static).

## Considered options

### Option A: Pointer tree of node objects (Three.js style)
- Pros: familiar; recursion is easy.
- Cons: pointer chasing, allocation, harder to parallelise and snapshot.

### Option B: Flat arrays with parent indices, topologically ordered (recommended)
Nodes stored in SoA arrays (local TRS, world matrix, parent index, dirty flag). Parents always have a lower index than children, so one linear pass computes all world matrices.
- Pros: linear pass, trivially chunked per depth level for jobs, snapshot-friendly, deterministic.
- Cons: re-parenting or inserting needs a re-sort (done at the sync point); invariant "parent index < child index" must be enforced.

### Option C: Use no hierarchy; compute every world pose analytically per entity
- Pros: no transform update pass.
- Cons: duplicates the kinematic chain in every system; breaks for generic rides (coaster editor, roadmap 10).

## Decision

Option B, as ECS components (`Parent`, `LocalTransform`, `WorldTransform`, `TransformDirty`) in the simulation module:

1. **Order**: nodes sorted so `parent < child`; sort key is (depth, creation order) which is deterministic. Re-sorting happens only at structural sync points (game-0002).
2. **Update**: systems write `LocalTransform` (angles, elevations). The transform system walks nodes in index order: if the node or any ancestor is dirty, `world = world[parent] * local`, set dirty for descendants. Depth-level batches are independent and run as parallel-for jobs (game-0003); within a level, nodes are independent.
3. **Dirty flags**: written by the owning system; propagation is part of the single pass, so no atomic writes across nodes. Static nodes cost one flag check.
4. **Precision**: world matrices are computed in double on the backend for ride kinematics, stored as float for rendering snapshots (float rules in game-0006).
5. **Mill -> arm -> hub -> gondola**: `Mill` rotates about Y by main angle phi. `Arm_k` is a child with a hinge rotation about its horizontal pivot by the elevation angle theta_k (cam-driven function of phi). The `Hub_k` node is a child of the arm tip but its local rotation is **solved, not authored**: after the arm pose, a `HubLinkage` system sets the hub's local rotation to the inverse of the arm's world pitch so the hub's world up-axis stays vertical. This is the compensating linkage expressed as an explicit constraint node; the hub's own motor rotation about the vertical axis is then applied as a child (`HubSpin`). `Gondola_j` are children of the hub arm tips with free spin driven by the physics. Systems order: MillMotion -> ArmElevation -> HubLinkage -> HubSpin -> Gondola dynamics -> transform pass.
6. **Relation to Three.js**: Three.js `Object3D` is a pointer tree with local `position/quaternion/scale`, `matrix`, `matrixWorld` and `matrixAutoUpdate` and `updateMatrixWorld()` recursion (known API, to be re-verified against the pinned 0.169 docs at apply time). The client keeps its own `Object3D` tree for rendering. Mapping: node names from the glTF/asset pipeline (game-0007) match backend node names; each snapshot carries the local or world transform per named node; the client sets `matrixAutoUpdate=false` for server-driven nodes and writes matrices directly, or sets local TRS and lets Three.js propagate. We do not replace Three.js's graph; we feed it.

## Consequences

- Good: one linear, parallel-friendly pass; explicit constraint for the linkage (testable: hub up-vector stays (0,1,0) within tolerance); asset naming maps 1:1.
- Bad: re-sort cost on structural changes; two scene graphs (server flat, client Object3D) must stay name-consistent; constraint solving beyond a single linkage needs more design.
- Neutral: matrix representation (4x4 vs TRS+quaternion) is an implementation detail with a benchmark spike.

## Validation

- Unit: hub world up-vector equals (0,1,0) within 1e-9 for 10,000 random main angles and all four phases.
- Unit: hub position equals (L cos(theta), L sin(theta)) formula within 1e-9.
- Property: for random trees, flat pass result equals recursive reference implementation bit-for-bit.
- Benchmark: world update of 10,000 nodes, 10% dirty < 0.2 ms single-threaded (target).
- Client: a golden-snapshot test renders the same pose via Three.js matrices and compares node world positions within 1e-4 m.

## References

- docs/05-forces-and-g-forces.md, docs/appendix-parameters.md (parameters; to be updated by the octopus-motion change)
- docs/adr/game-0002-data-oriented-component-storage.md, game-0007-asset-pipeline-blender-krita-gltf.md
- https://threejs.org/docs/#api/en/core/Object3D (not fetched in this session; unverified for 0.169)
