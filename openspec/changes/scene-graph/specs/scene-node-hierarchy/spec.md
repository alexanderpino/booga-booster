## ADDED Requirements

### Requirement: Nodes have stable ids, names and paths

Every scene node SHALL have a `NodeId` (64-bit integer, 1 or greater, `0` meaning "none") allocated from a monotonically increasing counter that is never reused within a world's lifetime, and a name matching `^[A-Za-z][A-Za-z0-9_-]{0,30}$`. Names SHALL be unique among siblings (root nodes are siblings of each other). A node's path SHALL be its ancestors' names and its own joined by `/`. Ids of surviving nodes SHALL NOT change when other nodes are created, destroyed or reparented.

#### Scenario: Path of a nested node

- **WHEN** root `greatMill` has child `arm0` which has child `hub0`
- **THEN** the path of `hub0` is `greatMill/arm0/hub0`
- **AND** looking up that path returns the id of `hub0`

#### Scenario: Ids are never reused

- **WHEN** nodes with ids 1, 2, 3 exist, node 2 is destroyed, and a new node is created
- **THEN** the new node has id 4
- **AND** nodes 1 and 3 keep their ids

#### Scenario: Duplicate sibling name rejected

- **WHEN** a second child named `arm0` is created under `greatMill`
- **THEN** the creation is rejected and the graph is unchanged

#### Scenario: Invalid names rejected

- **WHEN** a name is empty, starts with a digit, contains `/` or a space, or is 32 characters long
- **THEN** the creation or rename is rejected and the graph is unchanged

### Requirement: Nodes are stored in parent-before-child level order

The graph SHALL keep its nodes in dense arrays ordered breadth first: all roots (in creation order), then all nodes of depth 1 grouped by parent order and ordered among siblings by sibling sequence, then depth 2, and so on. For every node the index of its parent SHALL be smaller than its own index. The graph SHALL expose the start index of each depth level so that a level is a contiguous slice. The order SHALL be a pure function of the sequence of structural operations.

#### Scenario: Level order

- **WHEN** roots A and B exist, then A1 (child of A), B1 (child of B), and A11 (child of A1) are created in that order
- **THEN** the dense order is A, B, A1, B1, A11
- **AND** level starts are 0, 2, 4

#### Scenario: Parent precedes child after reparenting

- **WHEN** A11 is reparented under B
- **THEN** after the next update the dense order is A, B, A1, B1, A11 with `parentIndex(A11) == index(B)`
- **AND** every node's parent index is smaller than the node's index

### Requirement: Reparenting prevents cycles

Reparenting node N under node P SHALL be rejected when P is N or a descendant of N. Reparenting a node under `none` SHALL make it a root. A rejected operation SHALL leave the graph unchanged and be counted. Reparenting SHALL NOT change the node's local transform; its world transform changes accordingly at the next update.

#### Scenario: Under itself

- **WHEN** node A is reparented under A
- **THEN** it is rejected

#### Scenario: Under a descendant

- **WHEN** A has child A1 and A is reparented under A1
- **THEN** it is rejected and A is still a root

#### Scenario: Valid move

- **WHEN** A1 is reparented under B
- **THEN** B is its parent, and A no longer lists A1 as a child
- **AND** A1 is appended last among B's children

### Requirement: Destroying a node destroys its subtree

Destroying a node SHALL destroy all its descendants. Remaining nodes SHALL keep their ids and their relative order. Queries on a destroyed id SHALL report not found.

#### Scenario: Subtree destroy

- **WHEN** `arm0` (id 2) with child `hub0` (id 3) is destroyed
- **THEN** ids 2 and 3 are not found, the graph has two fewer nodes, and the structure version increased

### Requirement: Limits are enforced

The graph SHALL reject creation beyond `MaxNodes` (default 262 144) and beyond depth `MaxDepth` (default 64; roots have depth 0). Rejections SHALL leave the graph unchanged.

#### Scenario: Depth limit

- **WHEN** a chain of 65 nested nodes is created (depths 0 to 64) and a node at depth 65 is requested
- **THEN** the request is rejected

### Requirement: Structure edits are blocked during the tick

Immediate structural edits through the graph API SHALL fail with an invalid-operation error while the world's iteration scope is open. Structure changes made through component values during a tick SHALL take effect at the next transform sync.

#### Scenario: Edit in a system

- **WHEN** a system calls the immediate create API during `Update`
- **THEN** an invalid-operation error is thrown
