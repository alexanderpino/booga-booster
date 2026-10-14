## ADDED Requirements

### Requirement: The scene is described by ECS components

A scene node SHALL be an ECS entity carrying: `SceneNode` (assigned `NodeId`, orientation mode), `SceneParent` (parent `NodeId`, 0 for root), `SceneName` (fixed-size UTF-8 name, at most 31 characters), `LocalTransform` and `WorldTransform` (both a transform of eight doubles). All of them SHALL be unmanaged and registered with stable names and schema versions. The ECS components SHALL be the source of truth; the flat-array graph SHALL be a derived cache that can be rebuilt from them at any time.

#### Scenario: Create through the builder

- **WHEN** the scene builder creates `greatMill` and `arm0` (under `greatMill`) between ticks
- **THEN** two entities exist with all five components, node ids 1 and 2, and `arm0`'s `SceneParent` is 1

#### Scenario: Rebuild equals live graph

- **WHEN** the graph is rebuilt from the components of a world with 300 random nodes
- **THEN** structure, dense order, locals and worlds are identical to the live graph

### Requirement: A transform sync system keeps components and graph consistent

A system `scene.transform-sync` SHALL run once per tick and: (1) assign ids to entities that have `SceneNode.NodeId == 0`, in ascending entity index order; (2) apply changes of `SceneParent` (reparent, cycle check), `SceneName` and `SceneNode` mode to the graph; (3) copy changed `LocalTransform` values into the graph; (4) run the graph update; (5) write the new world transforms into `WorldTransform` for every recomputed node. A rejected change SHALL be reverted in the component to the last accepted value and counted. The system SHALL declare reads of `LocalTransform`, `SceneParent`, `SceneName` and writes of `SceneNode`, `SceneParent` (for reverts), `WorldTransform` so the scheduler orders gameplay writers of `LocalTransform` before it.

#### Scenario: Local change reaches the world component

- **WHEN** a system sets the arm's `LocalTransform` rotation in tick 1
- **THEN** after tick 1 the hub's `WorldTransform` reflects the new arm rotation

#### Scenario: Cycle in component data is reverted

- **WHEN** a system sets `greatMill`'s `SceneParent` to `arm0`'s id
- **THEN** after the tick `greatMill`'s `SceneParent` is 0 again and the rejection counter increased by 1

#### Scenario: Parent id of a missing node

- **WHEN** a node's `SceneParent` points to a NodeId that does not exist
- **THEN** it is reverted to its previous parent and counted

#### Scenario: Ids assigned in entity order

- **WHEN** three entities with `SceneNode.NodeId == 0` and entity indices 7, 3, 5 are synced
- **THEN** they receive node ids in the order of entity indices 3, 5, 7

### Requirement: Entity destruction removes nodes and cascades

The scene graph SHALL observe entity destruction. Destroying an entity that carries `SceneNode` SHALL remove its node and destroy the entities of all descendant nodes within the same command playback.

#### Scenario: Cascade

- **WHEN** the entity of `arm0` is destroyed through a command buffer while `hub0` is its child
- **THEN** after playback both entities are not alive and the graph has two fewer nodes

### Requirement: Save, restore and hash cover the scene

Because structure and transforms live in components, a world restored from a binary snapshot SHALL, after the graph is rebuilt, produce identical world transforms and identical query results, and the canonical state hash SHALL change when a node's parent, name, mode or local transform changes. Node-id allocation SHALL continue without reuse after a restore.

#### Scenario: Restore

- **WHEN** a world with 100 nodes is saved, restored and the graph rebuilt
- **THEN** all 100 world transforms equal the originals bitwise and the next created node gets the same id in both worlds

### Requirement: The transform pass is deterministic and level-sliceable

The update pass SHALL touch the dense arrays in index order, write only the node being computed, and read only already-computed parent entries, so that nodes of one level can be computed in any order or in parallel with identical results. The graph SHALL expose level slices for a later parallel executor.

#### Scenario: Level order independence

- **WHEN** a test computes each level's nodes in reverse index order
- **THEN** all world transforms are bitwise equal to the sequential pass
