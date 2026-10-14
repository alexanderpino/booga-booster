## ADDED Requirements

### Requirement: A canonical state hash identifies world state

The world SHALL compute a 64-bit FNV-1a hash of its canonical state: tick, then entities in ascending slot index (index, generation, alive flag), then for each live entity its components in ascending `ComponentTypeId` as raw bytes. The hash SHALL NOT depend on archetype layout or row order. The hash is a determinism check, not a security measure.

#### Scenario: Layout independence

- **WHEN** two worlds contain the same entities and component values but were built through different sequences of add/remove that produce different row orders
- **THEN** their hashes are equal

#### Scenario: Value sensitivity

- **WHEN** one `double` component value differs by one ulp
- **THEN** the hashes differ

### Requirement: The world can be saved and restored losslessly

The serializer SHALL write a versioned binary snapshot containing the magic `BBWS`, a format version, the tick, the entity slot table (generations, alive flags, free-list order), the component table (name, size, schema version) and all live component data. Restoring SHALL build a new world with an identical state hash. After a restore, creating entities SHALL assign the same ids as the original world would have. Restoring SHALL fail without producing a world when the magic or format version is wrong, a component name is unknown, a component size or schema version differs from the registry, or the data is truncated.

#### Scenario: Round trip

- **WHEN** a world with 1 000 entities and 3 component types is saved and restored
- **THEN** the restored world has the same hash, the same entity count and the same tick

#### Scenario: Continue identically

- **WHEN** both worlds then run the same 600 ticks of the same systems
- **THEN** their hashes are equal after every 60th tick

#### Scenario: Same ids after restore

- **WHEN** an entity is created in the original and in the restored world
- **THEN** both receive the same `EntityId`

#### Scenario: Corrupt or incompatible data

- **WHEN** the first byte of the snapshot is changed, or the stream is cut in half, or a component's schema version differs
- **THEN** restore fails with a snapshot-format error and no world is returned

### Requirement: Replicated components are projected to a JSON snapshot

Components registered with a replication schema (an ordered list of field names and `double` accessors) SHALL be included in a JSON snapshot DTO: `schemaVersion`, `tick`, `timeSeconds` (`tick / 120.0`), `schema` (component name to ordered field names) and `entities` (ascending slot index, each with `index`, `generation` and `values` as component name to number array in schema order). Entities without any replicated component SHALL be omitted. A non-finite replicated value SHALL fail the projection with an error naming the component and entity rather than emit invalid JSON.

#### Scenario: Projection shape

- **WHEN** entity 0 has replicated `position = (1.5, 2, -3)` and the schema is `["x","y","z"]`
- **THEN** the DTO contains `schema.position == ["x","y","z"]` and entity 0 with `values.position == [1.5, 2, -3]`

#### Scenario: Non-replicated component hidden

- **WHEN** a component is not registered as replicated
- **THEN** it does not appear in the DTO

#### Scenario: NaN fails loudly

- **WHEN** a replicated value is NaN
- **THEN** projection throws and the snapshot endpoint answers 500 problem details

### Requirement: The snapshot is served read-only

`GET /simulation/snapshot` SHALL return the JSON projection of the current tick. The endpoint SHALL contain no business logic and SHALL dispatch a query handler. Reading SHALL NOT occur while a tick is executing: the handler SHALL take the snapshot between ticks.

#### Scenario: Between ticks

- **WHEN** the endpoint is called while the loop is mid-tick
- **THEN** the response reflects the state after the last completed tick

### Requirement: The Angular client reads snapshots through a thin typed reader

The Angular app SHALL provide a pure function `parseWorldSnapshot(raw: unknown)` returning a typed `WorldSnapshot`. It SHALL reject payloads whose `schemaVersion` is unsupported, whose `values` length differs from the schema field count, or whose fields have the wrong type, by throwing an error naming the offending path. It SHALL NOT implement entities, components or systems client-side.

#### Scenario: Valid payload

- **WHEN** a payload as in the projection example is parsed
- **THEN** `readComponent(snapshot, entity, "position")` returns `{ x: 1.5, y: 2, z: -3 }`

#### Scenario: Length mismatch

- **WHEN** an entity's `position` array has 2 numbers while the schema has 3 fields
- **THEN** parsing throws with a message containing `entities[0].values.position`

#### Scenario: Unsupported version

- **WHEN** `schemaVersion` is 99
- **THEN** parsing throws an unsupported-version error
