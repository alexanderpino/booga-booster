# Spec Delta

## Purpose

Defines the commands a player uses to change the park (validate, place, batch, remove, terrain edit, undo, redo), the rules that accept or refuse them with stable reason codes, the cost preview, and the deterministic order in which they take effect.

## ADDED Requirements

### Requirement: Placeable definitions come from providers

Every placement SHALL name a definition id. The system SHALL resolve ids through registered providers and SHALL ship providers for `path`, `queue-path` and `park-entrance`, plus development-only `marker-2x2` and `marker-10x10` blocks enabled by configuration. An unknown id SHALL be refused with `DefinitionUnknown`; a provider MAY refuse a known id with `DefinitionLocked`. Two providers offering the same id SHALL be a startup error.

#### Scenario: Unknown definition

- **WHEN** a placement names `no-such-ride`
- **THEN** it is refused with reason `DefinitionUnknown`

#### Scenario: Marker blocks disabled

- **WHEN** marker blocks are disabled by configuration and `marker-10x10` is placed
- **THEN** it is refused with reason `DefinitionUnknown`

#### Scenario: Duplicate id across providers

- **WHEN** two providers both offer the id `path`
- **THEN** application start-up fails with an error naming the id

### Requirement: Validation is a side-effect-free dry run

A validate request SHALL run exactly the checks of the matching command and return: whether it is valid, all reason codes that apply (not only the first), the cost in cents, and the tiles it would occupy. It SHALL NOT change terrain, entities, serials, funds, undo history or the state hash.

#### Scenario: Validate then place agree

- **WHEN** a placement is validated as valid and then executed with no other command in between
- **THEN** the execution succeeds with the same cost

#### Scenario: Validate changes nothing

- **WHEN** a placement is validated 100 times
- **THEN** the state hash, the placement serial and the funds are unchanged

#### Scenario: All reasons reported

- **WHEN** a footprint is partly out of bounds and partly on an occupied tile
- **THEN** the response lists both `OutOfBounds` and `Occupied`

### Requirement: Placement rules

A placement SHALL be accepted only if all of the following hold, and otherwise refused with the named reason: every footprint cell is inside the map (`OutOfBounds`); the rotation is one the definition allows (`RotationInvalid`); for a ground-anchored definition, every footprint tile is Flat, except path definitions which also accept Ramp tiles (`TerrainNotFlat` or `TerrainRough`) and all footprint tiles share one level (`TerrainNotFlat`); no overlapping vertical span (`Occupied`, `ClearanceBlocked`); the per-park limit is respected (`MaxPerParkReached`); entrance-specific rules hold (`EntranceNotOnBoundary`); funds suffice (`InsufficientFunds`).

#### Scenario: Object on a slope

- **WHEN** a 2 x 2 marker is placed over a footprint containing a Ramp tile
- **THEN** it is refused with `TerrainNotFlat`

#### Scenario: Footprint straddles two levels

- **WHEN** every footprint tile is Flat but one is at level 9 and the others at level 8
- **THEN** it is refused with `TerrainNotFlat`

#### Scenario: Rotation not allowed

- **WHEN** a definition allows only rotations 0 and 2 and is placed with rotation 1
- **THEN** it is refused with `RotationInvalid`

#### Scenario: Footprint outside the map

- **WHEN** a 10 x 10 marker is anchored at `(125, 125)` on a 128 x 128 park
- **THEN** it is refused with `OutOfBounds`

### Requirement: Vertical spans, occupancy and clearance

Each placed object SHALL occupy, on each of its footprint tiles, the vertical span from its base level up to base plus its definition's clearance levels (path 12 levels). A tile SHALL hold at most 4 spans. A new span that overlaps an existing span on any shared tile SHALL be refused: with `Occupied` when the existing span's base is at or below the new base, with `ClearanceBlocked` when the existing span's base is above it. A definition anchored `free` SHALL need an explicit base level not below the highest corner of every footprint tile.

#### Scenario: Same tile twice

- **WHEN** a path exists at tile `(5, 5)` and a second path is placed there
- **THEN** it is refused with `Occupied`

#### Scenario: Overhead object above a path

- **WHEN** a path at level 8 (span 8 to 19) exists and a free-anchored test object with clearance 4 is placed over it at base 20
- **THEN** the placement is accepted

#### Scenario: Overlap with the path headroom

- **WHEN** the same free-anchored object is placed at base 19
- **THEN** it is refused with `Occupied`, since the path's base 8 is below the new base

#### Scenario: Ground object under an overhead object

- **WHEN** an overhead object spans levels 30 to 40 and a ground object with clearance 40 is placed under it at base 8
- **THEN** it is refused with `ClearanceBlocked`

#### Scenario: Span limit

- **WHEN** a tile already holds 4 spans and another object is placed there
- **THEN** it is refused with `TileSpansFull`

### Requirement: Path additions

A definition with host kind Path (bench, bin, lamp) SHALL be placeable only on a tile holding a path and SHALL occupy no span. A path tile SHALL hold at most one addition. Removing the path SHALL remove its addition.

#### Scenario: Addition on a path

- **WHEN** a path addition is placed on a path tile
- **THEN** it is accepted and the path tile reports one addition

#### Scenario: Addition without a path

- **WHEN** a path addition is placed on bare ground
- **THEN** it is refused with `HostMissing`

#### Scenario: Second addition

- **WHEN** the path tile already holds an addition
- **THEN** a second one is refused with `HostOccupied`

#### Scenario: Path removal takes the addition

- **WHEN** a path tile with an addition is removed
- **THEN** both entities are destroyed and one undo entry covers both

### Requirement: Cost, preview and funds

Each command SHALL have a cost in integer cents: a placement costs its definition's build cost; a terrain edit costs 150 cents per vertex-level moved; removal costs 0 and refunds 0. A funds hook SHALL be asked to charge before a command takes effect; the default implementation SHALL have unlimited funds. If the hook refuses, the command SHALL be refused with `InsufficientFunds` and change nothing.

#### Scenario: Path cost

- **WHEN** a Normal path (800 cents) is placed
- **THEN** the response reports cost 800 and the funds hook was charged 800

#### Scenario: Terrain cascade cost

- **WHEN** the second raise of a vertex moves nine vertex-levels
- **THEN** its cost is 1350 cents

#### Scenario: Refused by funds

- **WHEN** the funds hook refuses a charge
- **THEN** the command is refused with `InsufficientFunds` and the state hash is unchanged

### Requirement: Batches are all or nothing

A batch SHALL carry 1 to 256 placements, validated and applied in the listed order, each seeing the effects of the earlier ones. If any item is refused, no item SHALL be applied, no charge SHALL be made and the response SHALL name the failing index and reasons. A larger batch SHALL be refused with `BatchTooLarge`. A successful batch SHALL be one undo entry and one funds charge of the total.

#### Scenario: Line of paths

- **WHEN** a batch of 10 Normal paths in a row is placed
- **THEN** the total cost is 8000 cents and the ten paths are connected to each other

#### Scenario: One bad item

- **WHEN** the third of three placements is refused
- **THEN** none of the three is applied and the response names index 2

#### Scenario: Batch too large

- **WHEN** a batch of 257 placements is sent
- **THEN** it is refused with `BatchTooLarge`

### Requirement: Removal

Removal addressed to any tile of an object SHALL remove the whole object. A lifecycle hook MAY veto removal with `ObjectBusy`. Removing from an empty tile SHALL be refused with `NothingToRemove`.

#### Scenario: Remove by any cell

- **WHEN** the player removes at a tile that is one of a 10 x 10 marker's cells
- **THEN** the whole marker and its scene node are removed

#### Scenario: Busy object

- **WHEN** a hook reports the object busy
- **THEN** removal is refused with `ObjectBusy` and nothing changes

#### Scenario: Empty tile

- **WHEN** removal targets a tile with no object
- **THEN** it is refused with `NothingToRemove`

### Requirement: Undo and redo

Every accepted placement, batch, removal and terrain edit SHALL push one undo entry; the history SHALL hold at most 50 entries, dropping the oldest. Undo SHALL apply the inverse; redo SHALL re-apply. A new accepted command SHALL clear the redo history. Undo of a placement SHALL refund the amount charged; redo SHALL charge again. Re-created objects SHALL get new entity ids and new serials. If the inverse is refused (for example `InsufficientFunds` on undoing a removal) undo SHALL be refused with `UndoConflict` listing the inner reasons and leave both histories unchanged. With nothing to undo or redo the command SHALL be refused with `UndoUnavailable` or `RedoUnavailable`. Loading a park SHALL clear both histories.

#### Scenario: Undo a placement

- **WHEN** a path is placed and then undone
- **THEN** the tile is empty, 800 cents were refunded, and redo is available

#### Scenario: Redo gets a new serial

- **WHEN** the placement with serial 3 is undone and redone
- **THEN** the object exists again with serial 4 and a new entity id

#### Scenario: New command clears redo

- **WHEN** a placement is undone and then another placement is made
- **THEN** redo is unavailable

#### Scenario: History depth

- **WHEN** 51 placements are made
- **THEN** 50 undos succeed and the 51st is refused with `UndoUnavailable`

#### Scenario: Undo conflict

- **WHEN** a removal is undone while the funds hook refuses the re-placement cost
- **THEN** undo is refused with `UndoConflict` containing `InsufficientFunds` and the object stays removed

### Requirement: Deterministic command order

Accepted requests SHALL be given a strictly increasing arrival sequence and SHALL take effect at the start of the next simulation tick, in sequence order; the response SHALL be sent after that tick's commands were applied. Each applied command SHALL be recorded in a command log with tick, sequence, type and payload. Applying the same log to a fresh park of the same size SHALL reproduce the same state hash.

#### Scenario: Two commands for one tile

- **WHEN** two placements for the same free tile arrive with sequences 1 and 2 before the next tick
- **THEN** the first succeeds and the second is refused with `Occupied`

#### Scenario: Replay

- **WHEN** a recorded log of 500 mixed commands is replayed on a fresh park
- **THEN** the final state hash equals the recorded one

#### Scenario: Validate is not ordered

- **WHEN** a validate request arrives
- **THEN** it is answered against the state at the start of the current tick without entering the command log

### Requirement: Lifecycle hooks and events

Registered lifecycle hooks SHALL be called in registration order: after a placement is accepted (to create further entities), before a removal (to veto), and after a removal (to clean up). After each applied command the system SHALL publish integration events `park-object-placed`, `park-object-removed` and `park-terrain-changed` carrying the tick and the affected ids and coordinates; observers SHALL NOT be able to change the outcome.

#### Scenario: Hook creates entities

- **WHEN** a hook adds a component to a newly placed object
- **THEN** the component is present before the next system runs

#### Scenario: Event after placement

- **WHEN** a placement is applied at tick 1200
- **THEN** a `park-object-placed` event with tick 1200 and the definition id is published

### Requirement: HTTP surface and input limits

The system SHALL expose `POST /park/build/validate`, `/park/build/place`, `/park/build/batch`, `/park/build/remove`, `/park/build/terrain`, `/park/build/undo` and `/park/build/redo`, and `GET /park/connectivity`. Malformed input (coordinates not integers, unknown fields, ids not matching `^[a-z][a-z0-9-]{0,63}$`, rotation outside 0 to 3, body over 64 KiB) SHALL return 400. A rule refusal SHALL return 409 with the reason codes. Success SHALL return 200 with cost and ids. Endpoints SHALL contain no business logic.

#### Scenario: Malformed id

- **WHEN** a placement names the id `../etc`
- **THEN** the response is 400

#### Scenario: Rule refusal

- **WHEN** a placement lands on an occupied tile
- **THEN** the response is 409 and the body lists `Occupied`

#### Scenario: Oversized body

- **WHEN** a request body is 70 KiB
- **THEN** the response is 400 and nothing is parsed further
