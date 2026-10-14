# Spec Delta

## Purpose

Defines the connectivity and validity rules a coaster design must satisfy, how findings are reported to the player, and which findings block a test run, a build or an opening. The rules are this project's own and are deterministic functions of the design and the park state.

## ADDED Requirements

### Requirement: Findings have a code, a severity and a location (Tier 1)

The validator SHALL return a list of findings, each with a stable code, a severity (`Error` or `Warning`), the affected piece index when there is one, a message for the player and, where it helps, a suggestion. The list SHALL be ordered by piece index (findings without a piece last), then by code, so equal designs give equal reports. Validation SHALL be a pure function of the design, the catalogue and the park occupancy it is given.

#### Scenario: Deterministic report

- **WHEN** the same design is validated twice against the same park state
- **THEN** the two reports are identical, including order

#### Scenario: Finding shape

- **WHEN** a finding is serialised
- **THEN** it has code, severity, optional pieceIndex, message and optional suggestion, and the code is one of the documented codes

### Requirement: The circuit must close (Tier 1)

A design SHALL be a closed circuit: the exit frame of the last piece (position, heading, slope class, bank class) SHALL equal the entry frame of the first piece. Otherwise the validator SHALL report `OpenCircuit` as an error with a gap report: the difference in tiles east and north, in height steps, in heading quarter turns, and whether the slope or bank classes differ.

#### Scenario: Closed starter

- **WHEN** the starter coaster is validated
- **THEN** there is no `OpenCircuit` finding

#### Scenario: Gap report

- **WHEN** the last exit frame is at tile (3, -2), height 1, heading 1, while the first entry is at tile (0, 0), height 0, heading 0
- **THEN** the finding reports gap east 3 tiles, north -2 tiles, height 1 step, heading 1 quarter turn

### Requirement: Exactly one station, correctly placed (Tier 1)

A design SHALL contain exactly one `station` piece, as its first piece. The platform length SHALL be at least the train length (cars x 2.0 m), else `StationTooShort` is an error. The piece before the station (the last piece, since the circuit is closed) SHALL be flat and straight or a trim brake, else `StationApproach` is an error. If none of the 3 pieces before the station is a brake, `NoApproachBrake` SHALL be a warning.

#### Scenario: Train too long for the platform

- **WHEN** a design has 8 cars (train length 16.0 m) and the 12.0 m station
- **THEN** `StationTooShort` is reported as an error

#### Scenario: Curve into the station

- **WHEN** the last piece is `curve:L4`
- **THEN** `StationApproach` is reported as an error naming that piece

#### Scenario: No brake

- **WHEN** the 3 pieces before the station are `straight`
- **THEN** `NoApproachBrake` is reported as a warning

### Requirement: Lift and brake runs have minimum lengths (Tier 1)

Each unbroken run of lift pieces SHALL be at least the train length plus 2.0 m long, else `LiftTooShort` is an error. A design with no lift piece SHALL give the warning `NoLift` (a downhill-only layout can still run on the station drive). Each unbroken trim brake run SHALL be at least 4.0 m, which every brake piece satisfies.

#### Scenario: Short lift

- **WHEN** a 4-car train (8.0 m) has a lift run of two `slope:+2+lift` pieces (8.94 m)
- **THEN** `LiftTooShort` is reported because 8.94 m is below 10.0 m

#### Scenario: Long enough

- **WHEN** the lift run is three `slope:+2+lift` pieces (13.42 m)
- **THEN** there is no `LiftTooShort`

### Requirement: Track may not collide with terrain, paths, rides or itself (Tier 1)

Each piece SHALL have a swept envelope 2.0 m wide, from 0.5 m below to 2.5 m above the centreline, quantised onto cells of one tile by one tile by one height step. The validator SHALL report `TrackCollision` (error) when two non-adjacent pieces of the design, or a piece and any other ride or coaster, claim the same cell. It SHALL report `TerrainCollision` (error) when the envelope bottom of a piece is below the terrain height of a tile it passes over, `PathClearance` (error) when a piece over a path tile has its envelope bottom less than 3.0 m above the path surface, and `OutOfBounds` (error) when a piece leaves the owned land. Adjacent pieces share their boundary cell without conflict.

#### Scenario: Crossing at 3 steps

- **WHEN** one track crosses another at the same tile with a centreline height difference of 3 steps
- **THEN** `TrackCollision` is reported

#### Scenario: Crossing at 4 steps

- **WHEN** the height difference is 4 steps
- **THEN** no collision is reported

#### Scenario: Under a path

- **WHEN** a piece's envelope bottom is 2.0 m above a path tile
- **THEN** `PathClearance` is reported, and at 3.0 m or more it is not

#### Scenario: Terrain

- **WHEN** a piece has its envelope bottom 0.1 m below the terrain height of a tile
- **THEN** `TerrainCollision` is reported naming the piece

### Requirement: Size and height limits (Tier 1)

A design SHALL have at most 256 pieces and its envelope SHALL stay within 64 height steps above the anchor and within the park grid; otherwise `PieceLimit` or `HeightLimit` is an error. The limits SHALL be configuration values with those defaults.

#### Scenario: Height limit

- **WHEN** a piece would reach 65 height steps above the anchor
- **THEN** `HeightLimit` is reported for that piece

### Requirement: Findings gate the actions (Tier 1)

A test run SHALL be refused while any error exists. Build SHALL be refused while any error exists or while the design is not Tested. Opening SHALL be refused while the last test run has verdict Fail. Warnings SHALL never block an action; the client SHALL show them.

#### Scenario: Warning only

- **WHEN** the only findings are warnings
- **THEN** a test run is accepted

#### Scenario: Error blocks test

- **WHEN** a test run is requested for a design with an `OpenCircuit` error
- **THEN** it is refused with 409 and the report is returned

### Requirement: Imported files are checked like built designs (Tier 1)

A design loaded from a file SHALL be checked for piece ids that exist and are enabled, and for each piece connecting to the previous one. A mismatch SHALL be reported as the error `JoinMismatch` with the piece index and the two class pairs, not as an exception.

#### Scenario: Hand-edited file

- **WHEN** a file lists `slope:+2` directly after `straight`
- **THEN** loading yields a design with `JoinMismatch` at index 1, and a test run is refused

### Requirement: Structural span rule and auto-close help (Tier 2)

A run of consecutive pieces with no support (over a path or ride footprint) longer than 24.0 m SHALL give the error `SpanTooLong`. For an open circuit the system MAY offer a suggestion: the shortest sequence of up to 12 palette pieces that closes the gap, found by a deterministic search with fixed tie-breaks, or the statement that none exists within the limit.

#### Scenario: Suggestion is deterministic

- **WHEN** the close-circuit suggestion is requested twice for the same open design
- **THEN** the two suggestions are identical

#### Scenario: No suggestion

- **WHEN** no sequence of up to 12 pieces closes the gap
- **THEN** the answer says so and lists the gap report
