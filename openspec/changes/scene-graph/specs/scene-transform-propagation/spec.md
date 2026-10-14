## ADDED Requirements

### Requirement: Transforms are double-precision position, unit quaternion and uniform scale

A transform SHALL consist of a position (three `double`), a rotation quaternion (four `double`, order x, y, z, w) and a uniform scale (`double`, greater than 0). The coordinate system SHALL be right-handed with +Y up, identical to Three.js. A local transform SHALL be rejected when any component is NaN or infinite, the scale is not greater than 0, or the quaternion length differs from 1 by more than 1e-9 before normalisation (the graph normalises accepted quaternions). A rejected local transform SHALL leave the previous local transform in place and be counted.

#### Scenario: Invalid scale

- **WHEN** a local transform with scale 0 is set
- **THEN** it is rejected and the previous local transform remains

#### Scenario: NaN position

- **WHEN** a local transform with a NaN coordinate is set
- **THEN** it is rejected and counted

### Requirement: World transform composes parent and local transforms

For a node with parent world transform `P` and local transform `L` in `Inherit` orientation mode, the world transform SHALL be: position `P.position + P.rotation * (P.scale * L.position)`, rotation `normalize(P.rotation * L.rotation)`, scale `P.scale * L.scale`. A root's world transform SHALL equal its local transform.

#### Scenario: Translation only

- **WHEN** a parent is at world (10, 0, 0) and the child's local position is (1, 2, 3)
- **THEN** the child's world position is (11, 2, 3)

#### Scenario: Scale

- **WHEN** the parent has world scale 2 at (0, 0, 0) and the child's local position is (1, 0, 0) with local scale 3
- **THEN** the child's world position is (2, 0, 0) and its world scale is 6

#### Scenario: Yaw of 90 degrees about Y

- **WHEN** a parent at the origin is rotated +90 degrees about Y and the child's local position is (6, 0, 0)
- **THEN** the child's world position is (0, 0, -6) within 1e-12

#### Scenario: Arm pitched 30 degrees under a yawed mill

- **WHEN** the mill is rotated +90 degrees about Y, the arm below it is rotated +30 degrees about Z, and the hub's local position under the arm is (6, 0, 0)
- **THEN** the hub's world position is (0, 3, -5.196152422706632) within 1e-9

#### Scenario: Large offsets keep precision

- **WHEN** a parent is at (1e7, 0, 0) and the child's local position is (0.001, 0, 0)
- **THEN** the child's world x minus 1e7 equals 0.001 within 1e-9

### Requirement: Dirty propagation recomputes only affected nodes

Changing a node's local transform, parent, or orientation mode SHALL mark that node dirty. During an update, a node SHALL be recomputed if it is dirty or its parent was recomputed in the same update; no other node SHALL be recomputed. The update SHALL be a single linear pass over the dense arrays in order. The graph SHALL expose the number of nodes recomputed by the last update. Setting a local transform to a bitwise-identical value SHALL NOT mark the node dirty.

#### Scenario: Counts on the ride-shaped tree

- **WHEN** a tree has one mill root, 4 arms, 4 hubs (one per arm), 16 hub arms (4 per hub) and 16 gondolas (one per hub arm), 41 nodes in total, and all are clean
- **AND** the mill's local rotation changes
- **THEN** the update recomputes 41 nodes
- **WHEN** only one hub's local rotation then changes
- **THEN** the update recomputes 9 nodes (the hub, its 4 arms, its 4 gondolas)
- **WHEN** nothing changes
- **THEN** the update recomputes 0 nodes

#### Scenario: Identical value is not a change

- **WHEN** a local transform is set to exactly its current value
- **THEN** the node is not dirty and the next update recomputes 0 nodes

### Requirement: World transforms are a pure function of local transforms and structure

The world transforms after an update SHALL depend only on the current local transforms, parents and orientation modes, never on the history of updates. Two graphs built through different sequences of operations to the same structure and locals SHALL produce bitwise-identical world transforms per node.

#### Scenario: No accumulated drift

- **WHEN** the same local transforms are applied and the graph is updated 1 000 000 times alternating between two states
- **THEN** each state's world transforms are bitwise identical to their first computation

#### Scenario: Equals a naive recursive reference

- **WHEN** random trees (200 seeds, up to 300 nodes) are updated
- **THEN** every node's world transform equals the result of a straightforward recursive composition from the root within 1e-9

### Requirement: Query API

The graph SHALL provide: `TryGetWorldTransform`, `TryGetWorldPosition`, `TryGetLocalTransform`, `TryGetParent`, `GetChildren` (in sibling order), `TryFindByPath` and `TryGetPath`. World results SHALL reflect the last completed update; a node changed since then SHALL report `IsDirty == true`. Queries SHALL NOT mutate the graph and SHALL be safe to call from several threads when no update runs.

#### Scenario: World position query

- **WHEN** the hub of the pitched-arm scenario is queried by id after an update
- **THEN** `TryGetWorldPosition` returns true and (0, 3, -5.196152422706632) within 1e-9

#### Scenario: Unknown id

- **WHEN** a destroyed or never-issued id is queried
- **THEN** the query returns false

#### Scenario: Stale until update

- **WHEN** a local transform changes and no update has run
- **THEN** the world query returns the previous value and `IsDirty` is true
