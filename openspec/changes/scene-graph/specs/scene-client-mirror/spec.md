## ADDED Requirements

### Requirement: The client mirrors the server graph as a Three.js hierarchy

The Angular app SHALL provide a `SceneMirror` that, given a structure snapshot, builds one Three.js `Object3D` per server node under a single root group, with `Object3D.name` equal to the node name and `userData.nodeId` equal to the server `NodeId`, and parent/child relations equal to the server's. It SHALL keep a map from node id to `Object3D` and support lookup by id and by path. Applying a structure whose `structureVersion` equals the current one SHALL be a no-op. Applying a newer structure SHALL create missing nodes, remove nodes that no longer exist (detaching them from their parent), reparent moved nodes, and rename changed nodes, without recreating unchanged `Object3D` instances.

#### Scenario: Build

- **WHEN** the mill/arm/hub structure is applied
- **THEN** `findByPath("greatMill/arm0/hub0")` returns an `Object3D` whose `userData.nodeId` is 3 and whose parent is the object for node 2

#### Scenario: Unchanged instances survive

- **WHEN** a structure with one extra node is applied after the first
- **THEN** the objects of nodes 1, 2, 3 are the same instances as before

#### Scenario: Remove and reparent

- **WHEN** a newer structure omits node 3 and moves node 4 from parent 2 to parent 1
- **THEN** node 3's object is no longer in the scene, the id map has no entry for 3, and node 4's object has parent node 1's object

### Requirement: Poses are applied as local transforms

Applying a pose snapshot SHALL set, for each listed node id, the `Object3D` local position, quaternion (x, y, z, w) and uniform scale from the eight numbers. Coordinates SHALL NOT be converted: server and client share a right-handed, +Y-up frame. A pose whose `structureVersion` differs from the mirror's SHALL NOT be applied and the call SHALL report `stale-structure` so the caller can fetch a new structure. Node ids unknown to the mirror SHALL be skipped and counted.

#### Scenario: Local pose

- **WHEN** the hub's pose (6, 0, 0, 0, 0, 0, 1, 1) is applied
- **THEN** the hub object's local position is (6, 0, 0) and its quaternion is identity

#### Scenario: Stale structure

- **WHEN** a pose with structureVersion 5 is applied to a mirror at version 4
- **THEN** nothing changes and the result is `stale-structure`

#### Scenario: World position equals the server's

- **WHEN** the mill/arm/hub poses of the pitched-arm scenario (mill yaw 90 degrees, arm pitch 30 degrees, hub at local (6, 0, 0) whose shipped rotation is the effective rotation of an upright hub) are applied and the scene's world matrices are updated
- **THEN** the hub's world position is (0, 3, -5.196152) within 1e-5 (single-precision tolerance)
- **AND** the hub's world up axis is (0, 1, 0) within 1e-6

### Requirement: Validation of snapshots

The mirror's snapshot reader SHALL validate shapes: `nodeIds.length * 8 == transforms.length`, ids are positive safe integers, names match the server's name rule, parents appear before their children, no duplicate ids. Violations SHALL throw an error naming the path of the offending element.

#### Scenario: Length mismatch

- **WHEN** `transforms` has 23 numbers for 3 ids
- **THEN** applying throws with a message containing `transforms`

#### Scenario: Child before parent

- **WHEN** a structure lists node 3 before its parent 2
- **THEN** applying throws naming `nodes[`
