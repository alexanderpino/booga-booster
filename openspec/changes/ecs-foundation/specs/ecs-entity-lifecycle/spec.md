## ADDED Requirements

### Requirement: Entities are identified by generational ids

The simulation world SHALL identify every entity by an `EntityId` made of a slot index (zero-based) and a generation (starting at 1). Creating entities on an empty world SHALL yield indices 0, 1, 2, ... in order, all with generation 1. Destroying an entity SHALL increment its slot's generation and return the slot to a LIFO free list, so the most recently freed slot is reused first. A handle whose generation no longer matches its slot SHALL be reported as not alive and SHALL never resolve to the entity that now occupies the slot. The default `EntityId` (generation 0) SHALL never be alive.

#### Scenario: Sequential creation

- **WHEN** three entities are created on a new world
- **THEN** they have indices 0, 1 and 2, each with generation 1
- **AND** the world's entity count is 3

#### Scenario: Slot reuse bumps the generation

- **WHEN** the entity at index 1 (generation 1) is destroyed and a new entity is created
- **THEN** the new entity has index 1 and generation 2
- **AND** the old handle `(1, 1)` reports `IsAlive == false`
- **AND** reading a component through the old handle fails with a stale-entity error

#### Scenario: LIFO reuse order is deterministic

- **WHEN** entities at indices 0, 1, 2 are destroyed in that order and three entities are created
- **THEN** the created entities receive indices 2, 1, 0 in that order

#### Scenario: Generation exhaustion retires the slot

- **WHEN** a slot's generation has reached `int.MaxValue` and its entity is destroyed
- **THEN** the slot is retired and never reused
- **AND** the next created entity receives a fresh index

#### Scenario: World capacity is bounded

- **WHEN** the number of live entities would exceed `MaxEntities` (default 1 048 576)
- **THEN** creation fails with a capacity-exceeded error and the world is unchanged

### Requirement: Components are registered explicitly with stable names

Component types SHALL be registered on a `ComponentRegistry` before a world is created. A component type SHALL be an unmanaged value type (no managed references). Each registration SHALL carry a unique name matching `^[a-z][A-Za-z0-9]*$` and a schema version, and SHALL receive a `ComponentTypeId` equal to its registration order (0, 1, 2, ...). Tag components (no data) SHALL be registered separately and SHALL occupy no per-entity storage. The registry SHALL be frozen when the first world is built from it; registering afterwards, registering the same type or name twice, or registering more than 256 types SHALL fail.

#### Scenario: Ids follow registration order

- **WHEN** `position`, `velocity`, `mass` are registered in that order
- **THEN** their `ComponentTypeId` values are 0, 1, 2

#### Scenario: Duplicate name rejected

- **WHEN** a second component is registered under the name `position`
- **THEN** registration fails and the registry is unchanged

#### Scenario: Registry frozen after world creation

- **WHEN** a world has been created from a registry
- **AND** a new component type is registered on that registry
- **THEN** registration fails with an invalid-operation error

#### Scenario: Managed field rejected at compile time

- **WHEN** a struct containing a `string` field is passed to `Register<T>`
- **THEN** the call does not compile because `T` is constrained to `unmanaged`

### Requirement: Components can be added, set, read and removed on live entities

For a live entity the world SHALL support adding a component with a value, overwriting the value of a component it has, reading it by reference, and removing it. Adding a component the entity already has SHALL overwrite its value. Removing a component the entity does not have SHALL be a no-op. Values of all other components of the entity SHALL be preserved across add and remove (the entity may move between archetypes). Operations on a stale handle SHALL fail with a stale-entity error.

#### Scenario: Add preserves existing values

- **WHEN** an entity has `position = (1, 2, 3)` and a `velocity` component is added
- **THEN** reading `position` still returns `(1, 2, 3)`
- **AND** `velocity` holds the supplied value

#### Scenario: Remove of a missing component is a no-op

- **WHEN** a component the entity lacks is removed
- **THEN** no error occurs and the entity's other components are unchanged

#### Scenario: Swap-remove keeps other entities intact

- **WHEN** the first of three entities in one archetype is destroyed
- **THEN** the other two still resolve through their handles with their original component values

### Requirement: Structural changes are blocked during iteration and deferred through a command buffer

While systems execute (a world "iteration scope" is open), immediate structural operations (create, destroy, add, remove) SHALL fail with an invalid-operation error; setting an existing component's value in place SHALL remain allowed. Systems SHALL record structural changes in an `EntityCommandBuffer` instead. A command buffer SHALL support create (returning a provisional `EntityId` usable in later commands of the same buffer), destroy, add, set and remove. Buffers SHALL be played back after all systems of a tick finished, ordered by (system execution order index, recorded sequence number). A command targeting an entity that is no longer alive at playback SHALL be discarded and counted, never thrown.

#### Scenario: Immediate create during iteration fails

- **WHEN** a system calls `world.CreateEntity()` while the tick is executing
- **THEN** an invalid-operation error is thrown

#### Scenario: Provisional id resolved within a buffer

- **WHEN** a buffer records `create` (provisional id P) followed by `add position` on P
- **AND** the buffer is played back
- **THEN** one new entity exists with that `position`

#### Scenario: Deterministic playback order across buffers

- **WHEN** system 0 records `create A` and system 1 records `create B`
- **AND** playback runs
- **THEN** A receives a lower slot index than B regardless of which system finished first

#### Scenario: Stale target discarded

- **WHEN** two buffers both destroy entity E and a third sets a component on E afterwards
- **THEN** the first destroy succeeds, the second destroy and the set are discarded
- **AND** the discarded-command counter increases by 2

### Requirement: Entity lifecycle observers

The world SHALL notify registered `IEntityLifecycleObserver` instances synchronously when an entity is created and immediately before it is destroyed, in registration order. Observers called during command-buffer playback SHALL receive the playback buffer so that follow-up destroys they record are executed within the same playback.

#### Scenario: Observer cascades a destroy

- **WHEN** an observer, on `OnEntityDestroyed(parent)`, records a destroy of a child entity
- **AND** the parent is destroyed through a command buffer
- **THEN** after playback both parent and child are not alive
