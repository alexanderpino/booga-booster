## ADDED Requirements

### Requirement: Systems declare their read and write component sets

Each system registered in a job graph SHALL declare the set of component types it reads and the set it writes, and a name. Registration order SHALL define the reference (serial) semantics of the frame: the outcome of a frame SHALL equal the outcome of running the systems one after another in registration order. Registering two systems with the same name SHALL be rejected.

#### Scenario: Duplicate system name

- **WHEN** two systems named `Movement` are registered
- **THEN** registration throws an error naming `Movement`

#### Scenario: Declared sets are exposed

- **WHEN** a graph is built and described
- **THEN** each node reports its name, its read set and its write set

### Requirement: Conflicts create ordering edges

The builder SHALL add an edge from an earlier system to a later system when they conflict: the earlier writes a component the later reads, the earlier reads a component the later writes, or both write the same component. Two systems that only read the same components, or that touch disjoint components, SHALL NOT be ordered against each other.

#### Scenario: Write then read is ordered

- **WHEN** system `Integrate` (writes `Position`) is registered before system `Render` (reads `Position`)
- **THEN** the graph has an edge `Integrate -> Render`

#### Scenario: Read then write is ordered

- **WHEN** system `Sense` (reads `Position`) is registered before system `Integrate` (writes `Position`)
- **THEN** the graph has an edge `Sense -> Integrate`

#### Scenario: Write-write is ordered

- **WHEN** two systems both write `Velocity`
- **THEN** the later one depends on the earlier one

#### Scenario: Read-read is not ordered

- **WHEN** two systems only read `Position`
- **THEN** there is no edge between them and they MAY run at the same time

#### Scenario: Disjoint systems run in parallel

- **WHEN** system A writes `Position` and system B writes `Color`, with no other overlap
- **THEN** the graph has no edge between A and B
- **AND** the graph reports them in the same dependency level

### Requirement: Explicit ordering and barriers

The builder SHALL accept an explicit "run after" relation between two systems, and a barrier node that depends on everything registered before it and that everything registered after it depends on. An explicit relation that would point to a later-registered system SHALL be rejected because it would contradict registration order.

#### Scenario: Explicit ordering without a data conflict

- **WHEN** system B is declared to run after system A and they share no components
- **THEN** the graph has an edge `A -> B`

#### Scenario: Barrier separates phases

- **WHEN** a barrier is registered between systems A and B
- **THEN** B starts only after A and every system registered before the barrier have finished

#### Scenario: Backward reference rejected

- **WHEN** system A (registered first) is declared to run after system B (registered later)
- **THEN** building throws an error naming both systems

### Requirement: The graph is built once, validated, and reusable

Building SHALL produce an immutable graph that can be run every frame without being rebuilt. Building SHALL validate the graph and fail if it contains a cycle or refers to an unknown system.

#### Scenario: Unknown system reference

- **WHEN** a run-after relation names a system that was never registered
- **THEN** building throws an error naming the missing system

#### Scenario: Reuse across frames

- **WHEN** one built graph is run for 1,000 frames
- **THEN** each frame runs every system exactly once

### Requirement: Undeclared access is detected in checked mode

When access checking is enabled, a system that reads or writes a component type outside its declared sets SHALL fail with an error naming the system and the component type. When checking is disabled, no check cost SHALL be paid per access.

#### Scenario: Undeclared write detected

- **WHEN** a system declared as reading `Position` writes `Position` while checking is on
- **THEN** an error names that system and `Position`

#### Scenario: Declared access passes

- **WHEN** a system accesses only its declared components while checking is on
- **THEN** no error is raised

### Requirement: Systems may parallelise internally

A system SHALL be able to schedule a parallel-for over its entities as part of its node. The graph node SHALL be complete only when the system's own nested jobs have completed, and dependents SHALL wait for that.

#### Scenario: Nested parallel-for

- **WHEN** system A runs a parallel-for over 10,000 entities and system B depends on A
- **THEN** B starts only after all 10,000 entities were processed

### Requirement: A failed system fails its dependents only

When a system throws, systems that depend on it, directly or transitively, SHALL NOT run, systems with no dependency on it SHALL still run, and the frame SHALL surface the exception.

#### Scenario: Independent system still runs

- **WHEN** system A throws, system B depends on A, and system C is independent of both
- **THEN** B does not run, C runs, and the frame reports A's exception

### Requirement: Graph description is available for tooling

The graph SHALL expose a description with each node's name, read and write sets, dependency list and dependency level, as plain data, so a dashboard or a test can render it.

#### Scenario: Describe a three-system graph

- **WHEN** a graph of `Integrate -> Collide` plus an independent `Audio` is described
- **THEN** the description lists three nodes, the one edge, and gives `Integrate` and `Audio` level 0 and `Collide` level 1
