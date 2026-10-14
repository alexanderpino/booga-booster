# Spec Delta

## Purpose

Defines how a park is saved and loaded, and how grid changes reach the browser as a binary keyframe plus sequenced deltas with loss recovery, in line with ADR game-0005.

## ADDED Requirements

### Requirement: Park save

The system SHALL save the whole park to a named file in the configured directory. The name SHALL match `^[a-z0-9][a-z0-9-]{0,63}$`. The file SHALL be a container with magic `BBPK`, a format version, the park dimensions, the terrain levels, the placement serial counter, the definition table (each id with the version it was placed with), the serialized simulation world, and a 64-bit FNV-1a checksum of everything before it. The file SHALL be written to a temporary name and atomically replace the target so a failure never leaves a half-written park.

#### Scenario: Save creates a file

- **WHEN** the park is saved as `demo-park`
- **THEN** a file `demo-park.bbpark` exists whose first four bytes are `BBPK`

#### Scenario: Unsafe name

- **WHEN** a save is requested as `../x` or `A B`
- **THEN** it is refused with a 400 and no file is written

#### Scenario: Failure keeps the old file

- **WHEN** writing fails after the temporary file was created
- **THEN** the previous `demo-park.bbpark` is unchanged and the temporary file is removed

### Requirement: Park load

Loading SHALL validate the entire file before any live state changes: magic, supported format version, size at most 64 MiB, checksum, dimensions within limits, terrain levels within 0 to 63 and satisfying the smoothness rule, and every definition id resolvable with a saved version not below the definition's minimum compatible version. A failure SHALL leave the live park untouched and report the first cause with one of `NotFound`, `TooLarge`, `BadMagic`, `UnsupportedFormatVersion`, `Corrupt`, `InvalidTerrain`, `DefinitionUnknown` (listing ids), `DefinitionIncompatible` (listing ids and versions). A valid load SHALL replace the park between ticks, clear undo and redo, and increment the sync epoch.

#### Scenario: Round trip

- **WHEN** a park with terrain edits, paths and a marker is saved and loaded into a fresh server
- **THEN** the state hash after load equals the state hash at save time

#### Scenario: Corrupted byte

- **WHEN** one byte in the middle of a saved file is changed and the file is loaded
- **THEN** loading fails with `Corrupt` and the live park is unchanged

#### Scenario: Future format

- **WHEN** a file declares a format version newer than supported
- **THEN** loading fails with `UnsupportedFormatVersion`

#### Scenario: Missing definition

- **WHEN** a file references the id `old-ride` that no provider offers
- **THEN** loading fails with `DefinitionUnknown` listing `old-ride`

#### Scenario: Definition changed incompatibly

- **WHEN** a file placed `old-ride` at version 1 and the current definition declares version 3 with minimum compatible version 2
- **THEN** loading fails with `DefinitionIncompatible` listing `old-ride` version 1

#### Scenario: Oversized file

- **WHEN** a file larger than 64 MiB is loaded
- **THEN** loading fails with `TooLarge` before the file is fully read

### Requirement: Grid keyframe

`GET /park/grid/keyframe` SHALL return a binary, little-endian snapshot with magic `BBPG`, format version, epoch, sequence number, state hash, dimensions, the vertex levels, run-length-encoded tile records (kind, path edge mask, attach bits, slope direction, occupant entity index and generation) and the list of placements (definition id, anchor, rotation, base level, serial, entity id). A client applying a keyframe SHALL be able to rebuild the full grid view with no other input.

#### Scenario: Keyframe describes the park

- **WHEN** a keyframe is requested for a park with one path tile
- **THEN** it reports the park dimensions, the path tile's kind and mask, one placement, and the current sequence and epoch

#### Scenario: Round trip

- **WHEN** a keyframe is decoded and re-encoded
- **THEN** the bytes are identical

#### Scenario: Empty park size target

- **WHEN** a keyframe is produced for an empty 128 x 128 park
- **THEN** it is at most 20 KB

### Requirement: Grid deltas

After every simulation tick that changed grid state, and no later than the next 30 Hz publish interval, the system SHALL emit one delta as a server-sent event named `grid-delta` on `GET /park/grid/stream`, coalescing all changes of that interval. A delta SHALL carry `epoch`, `seq`, `baseSeq` (the previous sequence number), `tick` and a list of operations: `vertex` (vx, vy, level), `tile` (x, y, kind, edges, attach bits, slope direction, occupant) and `object` (add with its placement fields, or remove by entity id). Sequence numbers SHALL increase by exactly one per delta within an epoch.

#### Scenario: Path placement delta

- **WHEN** one Normal path is placed next to an existing path
- **THEN** one delta is emitted with a `tile` op for the new tile, `tile` ops for the neighbour whose mask changed, and one `object` add

#### Scenario: Coalescing

- **WHEN** three commands are applied within one publish interval
- **THEN** one delta carries all their operations in application order

#### Scenario: No change, no delta

- **WHEN** a tick changes no grid state
- **THEN** no delta is emitted

### Requirement: Loss and epoch recovery

A client SHALL apply a delta only when its epoch equals the client's epoch and its `baseSeq` equals the client's last applied sequence. Otherwise it SHALL request `GET /park/grid/delta?since=<seq>`. The server SHALL keep the last 256 deltas and SHALL answer with the deltas after `seq` when it still has them, or 410 Gone when it does not or when the epoch changed, after which the client SHALL fetch a new keyframe. A client SHALL reach the server's state hash after recovery.

#### Scenario: Missed delta

- **WHEN** a client misses delta 41 and receives delta 42
- **THEN** it requests deltas since 40, applies 41 and 42, and its grid equals the server's

#### Scenario: Too old

- **WHEN** a client asks for deltas since a sequence 300 behind
- **THEN** the server answers 410 and the client fetches a keyframe

#### Scenario: Epoch change

- **WHEN** a park is loaded while a client is connected
- **THEN** the next delta carries a new epoch and the client discards its grid and fetches a keyframe

### Requirement: Wire budgets

A delta for one Normal path placed next to one neighbour SHALL be at most 1 KB of JSON. A delta for a 10 x 10 marker placement SHALL be at most 10 KB. A keyframe for a 128 x 128 park whose tiles are all paths SHALL be at most 150 KB. These are targets measured in tests and recorded in the docs; JSON field names SHALL be short and stable.

#### Scenario: Sizes recorded

- **WHEN** the size tests run
- **THEN** they print the measured bytes for the three cases and fail only when a target is exceeded by more than a factor of 2

### Requirement: Streaming is read-only and bounded

The stream and keyframe endpoints SHALL be read-only. The server SHALL cap concurrent grid streams per process at a configured number (default 32) and SHALL answer further connections with 503. A slow client SHALL not delay the simulation: its queue SHALL hold at most 64 deltas and the client SHALL be disconnected when exceeded.

#### Scenario: Slow consumer

- **WHEN** a stream consumer does not read and 65 deltas accumulate
- **THEN** that stream is closed and the simulation tick duration is unaffected
