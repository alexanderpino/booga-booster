## ADDED Requirements

### Requirement: Systems declare their component access

A system SHALL implement an interface exposing a unique name, the components it reads, the components it writes, and optional explicit ordering constraints (`UpdateAfter` / `UpdateBefore` other system names), and an `Update` operation receiving a context with the world, the fixed step in seconds, the tick number and its command buffer. A system SHALL only be able to obtain chunk spans for components it declared; requesting an undeclared component or requesting write access to a read-declared component SHALL fail when access validation is enabled (default on in Debug builds and in tests).

#### Scenario: Undeclared component

- **WHEN** a system declaring only `position` requests a `velocity` span
- **THEN** an invalid-operation error naming the system and the component is thrown

#### Scenario: Write to a read-declared component

- **WHEN** a system declaring `position` as read requests a write span of `position`
- **THEN** an invalid-operation error is thrown

### Requirement: Scheduler builds a deterministic execution plan

The scheduler SHALL build an execution plan from registered systems: for every pair of systems with a write/write, write/read or read/write overlap on a component, an ordering edge SHALL be added from the earlier-registered system to the later-registered one; read/read overlaps SHALL add no edge; explicit `UpdateAfter`/`UpdateBefore` constraints SHALL add edges; the plan SHALL be a topological order with ties broken by registration order. The plan SHALL also be exposed as stages: each stage a list of systems with no edge among them, in registration order. A cycle SHALL fail plan construction with an error naming the systems involved. Registering two systems with the same name SHALL fail.

#### Scenario: Read-read systems share a stage

- **WHEN** systems A (reads position) and B (reads position) are registered
- **THEN** the plan has one stage containing A then B

#### Scenario: Writer before reader by registration order

- **WHEN** A (writes position) is registered before B (reads position)
- **THEN** B runs after A and they are in different stages

#### Scenario: Explicit constraint adds an edge

- **WHEN** B declares `UpdateBefore("A")` and no component overlap exists
- **THEN** B runs before A

#### Scenario: Cycle detected

- **WHEN** A declares `UpdateAfter("B")` and B declares `UpdateAfter("A")`
- **THEN** building the plan fails with an error that names both A and B

### Requirement: Single-threaded executor runs the plan in order

The default executor SHALL run each system once per tick in plan order on the calling thread, SHALL open the world's iteration scope for the duration, and SHALL play back all command buffers after the last system in (system order index, sequence) order. Any executor SHALL produce the same world state as the sequential executor for the same inputs.

#### Scenario: One tick

- **WHEN** a tick runs with systems A then B
- **THEN** A's `Update` completes before B's starts, and command buffers are played back once, after B

#### Scenario: Exception in a system

- **WHEN** a system throws during `Update`
- **THEN** the iteration scope is closed, the command buffers of that tick are discarded, and the exception propagates to the caller of the tick

### Requirement: Systems are registered through dependency injection

`AddSimulationModule()` SHALL register the registry, world, scheduler and tick driver, and SHALL collect every `ISimulationSystem` registered in DI (via `AddSimulationSystem<T>()`) in registration order.

#### Scenario: Two systems registered

- **WHEN** two systems are added with `AddSimulationSystem<T>()`
- **THEN** the scheduler plan contains both in registration order
