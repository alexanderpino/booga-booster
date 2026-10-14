## ADDED Requirements

### Requirement: Structure snapshot

`GET /simulation/scene/structure` SHALL return `structureVersion` (integer, incremented by every create, destroy, reparent and rename) and `nodes`: for every node in dense order `id`, `parentId` (null for roots), `name`, `path`. The endpoint SHALL dispatch a query handler and contain no business logic.

#### Scenario: Mill, arm, hub

- **WHEN** the graph holds `greatMill` (1), `arm0` (2, parent 1), `hub0` (3, parent 2)
- **THEN** `nodes` lists ids 1, 2, 3 in that order with paths `greatMill`, `greatMill/arm0`, `greatMill/arm0/hub0`, and `parentId` null, 1, 2

#### Scenario: Version bumps

- **WHEN** a node is renamed
- **THEN** `structureVersion` is greater than before; a pure transform change leaves it unchanged

### Requirement: Pose snapshot

`GET /simulation/scene/pose` SHALL return `tick`, `structureVersion`, `nodeIds` (dense order) and `transforms`: a flat array of eight numbers per node in the same order (`px, py, pz, qx, qy, qz, qw, scale`), holding the **effective local** transforms of the last completed tick. Effective local means: the transform which, composed under plain `Inherit` rules with the parent's world transform, reproduces the node's world transform. For `Inherit` nodes it equals the local transform. For `UprightYaw` nodes position and scale equal the local ones and the rotation is `conjugate(P.rotation) * W.rotation` (the parent's world rotation undone, then the node's world rotation). Local poses are sent so that the client's own hierarchy (Three.js) composes world poses without knowing orientation modes, and so that numbers stay small in large worlds. Non-finite values SHALL fail the request with a problem-details 500 naming the node. The snapshot SHALL be taken between ticks.

#### Scenario: Shape

- **WHEN** three nodes exist
- **THEN** `nodeIds` has length 3 and `transforms` has length 24

#### Scenario: Upright node ships its effective rotation

- **WHEN** the mill is yawed +90 degrees about Y, the arm pitched +30 degrees about Z, and the hub is `UprightYaw` with identity local rotation
- **THEN** the hub's pose rotation is the inverse of the arm's world rotation times the hub's world rotation, i.e. a rotation about Z of -30 degrees (within 1e-12)
- **AND** composing the three poses under `Inherit` rules gives a hub world up axis of (0, 1, 0) within 1e-12

#### Scenario: Locals not worlds

- **WHEN** the arm's local position is (0, 0, 0) with rotation 30 degrees about Z, and the hub's local position is (6, 0, 0)
- **THEN** the pose holds (6, 0, 0) for the hub regardless of the mill's yaw

### Requirement: Single-node world position query

`GET /simulation/scene/nodes/{id}` SHALL return `id`, `name`, `path`, `parentId`, `worldPosition` (three numbers) and `isDirty` for an existing node, 404 for an unknown id, 400 for an id that is not a positive integer.

#### Scenario: Found

- **WHEN** node 3 of the pitched-arm scenario is requested
- **THEN** `worldPosition` is (0, 3, -5.196152422706632) within 1e-9

#### Scenario: Not found and malformed

- **WHEN** id 99 is requested
- **THEN** 404
- **WHEN** id `abc` is requested
- **THEN** 400

### Requirement: Scene metrics

The module SHALL publish on the shared meter: gauge `boogabooster.scene.nodes` (`{node}`), histogram `boogabooster.scene.update.duration` (`ms`), gauge `boogabooster.scene.recomputed` (nodes recomputed by the last update), counter `boogabooster.scene.rejected` (tag `reason` in `cycle`, `missing-parent`, `invalid-name`, `invalid-transform`, `duplicate-name`, `limit`).

#### Scenario: Rejection counted

- **WHEN** a cycle-creating reparent is rejected
- **THEN** `boogabooster.scene.rejected` with `reason=cycle` increases by 1
