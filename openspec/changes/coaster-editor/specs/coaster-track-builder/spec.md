# Spec Delta

## Purpose

Defines how a player builds a coaster piece by piece: starting a design, a palette that only offers pieces that connect, appending and removing, previewing a piece before committing it, undo and redo, concurrency safety, and the life cycle from draft to built ride.

## ADDED Requirements

### Requirement: A design starts with an anchored station (Tier 1)

A new coaster design SHALL be created with an anchor (tile x, tile z, height step, heading) and SHALL contain exactly one piece, a `station`, placed at the anchor. A design SHALL have a name of 1 to 40 characters, a ride type id, and a train configuration (cars 1 to 8, default 4, and seats per car, default 2). An anchor outside the owned land or below the terrain SHALL be rejected with the validation code that applies.

#### Scenario: Create with defaults

- **WHEN** a design is created at tile (10, 20), height 0, heading 0 with name "First"
- **THEN** it has one piece `station`, 4 cars of 2 seats, revision 1, state Draft and no undo history

#### Scenario: Bad name

- **WHEN** a design is created with an empty name or a name of 41 characters
- **THEN** the request is rejected with a 400 naming the name rule

### Requirement: The palette offers only pieces that connect (Tier 1)

For a design, the system SHALL compute the palette as the pieces of the allowed set whose entry slope class and bank class equal the exit slope class and bank class of the last piece. Each palette entry SHALL carry its id, display name, group and a validity flag with a reason when appending it would raise an error-severity finding. The palette order SHALL be deterministic: groups in a fixed order, then smaller slope change first.

#### Scenario: After the station

- **WHEN** the last piece is `station` (exit slope class 0)
- **THEN** the palette holds `straight`, `curve:L2`, `curve:R2`, `curve:L4`, `curve:R4`, `brake:trim` and the `pitch:0>x` transitions to classes -2, -1, +1 and +2 including their lift variants, and no `slope` piece and no `pitch:+1>x` piece

#### Scenario: On a slope

- **WHEN** the last piece is `slope:+2` (exit class +2)
- **THEN** the palette holds `slope:+2`, `slope:+2+lift` and the `pitch:+2>x` transitions, and no curve and no `straight`

### Requirement: Append and remove act on the end of the track (Tier 1)

The system SHALL append a chosen palette piece after the last piece and SHALL remove the last piece except the station. Every successful append or remove SHALL raise the design revision by 1, invalidate any stored test run, and return the new design with its validation report and palette. Appending a piece that is not in the palette, or beyond 256 pieces, SHALL be rejected with 409 and an error code, leaving the design unchanged.

#### Scenario: Append

- **WHEN** `straight` is appended to a design at revision 3
- **THEN** the design has revision 4, one more piece, and its test result is cleared

#### Scenario: Remove the station

- **WHEN** removal of the last piece is requested on a design that only contains the station
- **THEN** the request is rejected and the design is unchanged

#### Scenario: Piece limit

- **WHEN** a design has 256 pieces and one more is appended
- **THEN** the request is rejected with the code `PieceLimit` and the design is unchanged

### Requirement: Preview does not change the design (Tier 1)

The system SHALL answer a preview request for a candidate piece with its placed frame, its findings if it were appended and the cost delta, without changing the revision or the history. A preview of a piece that is not in the palette SHALL be answered as not connectable with the exit and entry classes that differ.

#### Scenario: Preview a colliding piece

- **WHEN** a preview is requested for a piece whose swept volume overlaps an existing piece
- **THEN** the answer lists a `TrackCollision` error naming both piece indexes, and the design revision is unchanged

#### Scenario: Preview twice

- **WHEN** the same preview is requested twice
- **THEN** both answers are identical

### Requirement: Undo and redo restore exact designs (Tier 1)

The system SHALL keep an operation history of at most 200 entries per design. Undo SHALL restore the design that existed before the last operation and redo SHALL reapply it. A new operation SHALL clear the redo stack. After an undo, the piece list and the content hash SHALL equal those before the undone operation. The history is session state and SHALL NOT be part of the track file.

#### Scenario: Undo an append

- **WHEN** a piece is appended and then undone
- **THEN** the piece list and the content hash equal the values before the append, and redo is available

#### Scenario: New edit clears redo

- **WHEN** an undo is followed by a different append
- **THEN** redo is no longer available

#### Scenario: History depth

- **WHEN** 201 operations are performed
- **THEN** the oldest operation can no longer be undone and the 200 most recent can

### Requirement: Commands are checked against the revision (Tier 1)

Every mutating request SHALL carry the revision it was made against. If the revision differs from the current one, the system SHALL reject the request with 409 and the current revision, and SHALL NOT apply it. Two simultaneous requests on one design SHALL be applied one after the other, never interleaved.

#### Scenario: Stale client

- **WHEN** two clients both send an append with revision 5
- **THEN** exactly one succeeds and the other gets 409 with revision 6

### Requirement: A design moves from draft to built (Tier 1)

A design SHALL be in state Draft, Tested (a test run completed for the current revision with no failing verdict) or Built (placed in the park as a ride). Building SHALL require the Tested state and zero error findings, SHALL register the ride with the ride catalogue port and SHALL spawn the coaster into the simulation through the simulation gateway. Editing a Tested design SHALL return it to Draft. A Built design SHALL be immutable; changing it SHALL be done on a copy.

#### Scenario: Build needs a test

- **WHEN** build is requested for a Draft design
- **THEN** the request is rejected with `TestRunRequired`

#### Scenario: Edit after test

- **WHEN** a piece is appended to a Tested design
- **THEN** its state is Draft and its ratings are cleared

### Requirement: Editing in the middle (Tier 2)

The system SHALL allow replacing one piece by another piece with the same entry and exit slope class, bank class and the same exit frame, and SHALL allow removing a trailing run of pieces by selecting a piece index. Replacement that changes the exit frame SHALL be rejected. Each such edit SHALL be one history entry.

#### Scenario: Swap a curve radius

- **WHEN** `curve:L2` is replaced by `curve:L4`
- **THEN** the request is rejected because the exit frame differs

#### Scenario: Swap a lift for a slope

- **WHEN** `slope:+2` is replaced by `slope:+2+lift`
- **THEN** the replacement succeeds as one history entry and the exit frame is unchanged

### Requirement: Construction cost estimate (Tier 2)

The system SHALL ask the construction cost port for a cost for the piece list and for each preview, and SHALL show the total and a per-group breakdown. If the port gives no answer, the estimate SHALL be absent, not zero. Building SHALL charge through the same port and SHALL fail without registering the ride if the charge is refused.

#### Scenario: Port missing

- **WHEN** no cost provider is registered
- **THEN** the design response has no cost field and building is not blocked by cost

#### Scenario: Charge refused

- **WHEN** the port refuses the charge during build
- **THEN** no ride is registered, no entity is spawned and the design stays Tested
