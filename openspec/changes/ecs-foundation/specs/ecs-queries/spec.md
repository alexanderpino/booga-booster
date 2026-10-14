## ADDED Requirements

### Requirement: Queries select entities by required and excluded components

A query SHALL be described by a set of required components (each marked read or write), a set of components that must be present without being accessed (`With`), and a set of components that must be absent (`Without`). It SHALL match exactly the entities whose archetype contains all required and `With` components and none of the `Without` components. Tag components SHALL be usable in `With` and `Without`.

#### Scenario: Required components

- **WHEN** 10 entities have `position` + `velocity`, 5 have only `position`, and the query requires `position` and `velocity`
- **THEN** the query visits exactly the 10

#### Scenario: Without filter

- **WHEN** the same data is queried for `position` with `Without<velocity>`
- **THEN** exactly the 5 position-only entities are visited

#### Scenario: Empty world

- **WHEN** a query runs on a world with no matching entities
- **THEN** zero chunks are produced and no error occurs

### Requirement: Queries iterate in contiguous chunks without allocating

A query SHALL yield its matching rows as chunks. A chunk SHALL expose, for each required component, a span over contiguous values of one archetype, and the matching `EntityId` span. A chunk SHALL contain at most `ChunkRows` rows (default 1 024). Iterating a query on an existing world SHALL allocate zero managed bytes in steady state (after the query has been created and archetypes exist).

#### Scenario: Chunk size cap

- **WHEN** 2 500 entities share one archetype and a query over it is iterated
- **THEN** three chunks are produced with 1 024, 1 024 and 452 rows

#### Scenario: Writes through spans are visible

- **WHEN** a system writes `position` through the write span of a chunk
- **THEN** reading `position` by handle afterwards returns the written value

#### Scenario: No allocation

- **WHEN** a query over 100 000 entities is iterated twice (warm-up, then measured)
- **THEN** `GC.GetAllocatedBytesForCurrentThread` is unchanged across the second iteration

### Requirement: Iteration order is deterministic

For a given sequence of world operations, iteration order SHALL be identical across runs, processes and machines: archetypes are visited in the order they were first created, and rows within an archetype in storage order (creation order, with destroy performed as swap-remove: the last row fills the freed row). The order is deterministic but NOT stable across structural changes; consumers needing a stable order SHALL sort by `EntityId.Index`.

#### Scenario: Creation order

- **WHEN** entities E0, E1, E2 with the same components are created in that order
- **THEN** a query visits E0, E1, E2

#### Scenario: Swap-remove order

- **WHEN** E0 is then destroyed
- **THEN** a query visits E2, E1

#### Scenario: Archetype creation order

- **WHEN** an entity with `{position}` is created, then one with `{position, velocity}`
- **THEN** the query for `position` visits the first entity before the second

### Requirement: Chunk partitioning for later parallel execution

A query SHALL expose its chunks as an indexable, immutable list (`ChunkCount` and `GetChunk(i)`) that is identical in content and order to sequential iteration, so that a job system can hand disjoint chunk ranges to workers.

#### Scenario: Index access equals sequential access

- **WHEN** a query is enumerated sequentially and also read by chunk index 0..ChunkCount-1
- **THEN** both visit the same entities in the same order
