# Spec Delta

## Purpose

Defines the catalogue of track pieces a coaster can be built from: the integer lattice they live on, how a piece is described by an entry frame and an exit frame, the tiers that separate the demo-ready minimum from stretch pieces, and which pieces a ride type may use. The numbers are this project's own track model, not copied from any other game.

## ADDED Requirements

### Requirement: Pieces are data on an integer lattice (Tier 1)

The system SHALL describe every track piece as data: a stable string id, a tier, a shape built from a small set of curve primitives, an entry frame, an exit frame and optional drive zones. Positions SHALL be integers in tiles (horizontal) and height steps (vertical); headings SHALL be one of four directions. The tile edge and height step SHALL come from the park grid (defaults 4.0 m and 1.0 m until `park-world-grid` fixes them). A piece composed from existing primitives SHALL need no code change to be added.

#### Scenario: Default lattice

- **WHEN** the park grid supplies no values
- **THEN** one tile is 4.0 m and one height step is 1.0 m

#### Scenario: Add a piece by data only

- **WHEN** a new catalogue entry that uses only existing primitives is added to the catalogue data
- **THEN** it appears in the catalogue endpoint with its geometry and passes the catalogue invariant tests without a code change

### Requirement: A frame is position, heading, slope class and bank class (Tier 1)

A frame SHALL consist of a lattice position (tile x, tile z, height step), a heading in {0 = east, 1 = north, 2 = west, 3 = south} where heading 1 is the direction of -Z and a left turn increases the heading by 1, a slope class (integer height steps per tile, tangent = class x height step / tile edge) and a bank class (0 in Tier 1). Two pieces SHALL connect only when the exit slope class and bank class of the first equal the entry slope class and bank class of the second.

#### Scenario: Left curve exit frame

- **WHEN** a `curve:L4` piece starts at tile (0, 0), height 0, heading 0
- **THEN** its exit frame is tile (4, -4), height 0, heading 1, slope class 0

#### Scenario: Class mismatch does not connect

- **WHEN** a piece with exit slope class +2 is followed by a piece with entry slope class 0
- **THEN** the two pieces are not connectable and the second is not offered by the builder

### Requirement: Tier 1 piece set (Tier 1)

The Tier 1 catalogue SHALL contain exactly 43 piece types: `station` (3 tiles, 12.0 m), `straight` (1 tile, 4.0 m), `curve:L2`, `curve:R2` (90 degrees, radius 2 tiles = 8 m, arc 12.566 m), `curve:L4`, `curve:R4` (radius 4 tiles = 16 m, arc 25.133 m), `slope:+1`, `slope:-1` (1 tile, 4.123 m), `slope:+2`, `slope:-2` (1 tile, 4.472 m), 20 `pitch:a>b` transitions between the slope classes -2 to +2, `brake:trim` (1 tile, 4.0 m) and 12 lift variants (see lift requirement).

#### Scenario: Piece count by group

- **WHEN** the Tier 1 catalogue is listed
- **THEN** it holds 1 station, 1 straight, 4 curves, 4 slopes, 20 pitch transitions, 1 trim brake and 12 lift variants, 43 in total

#### Scenario: Piece lengths

- **WHEN** the lengths of `straight`, `curve:R4` and `slope:+2` are read
- **THEN** they are 4.0000 m, 25.1327 m and 4.4721 m within 1e-4 m

### Requirement: Pitch transitions are parabolic and sized by their slope change (Tier 1)

A `pitch:a>b` piece between slope classes a and b (a differs from b) SHALL span n tiles with n = max(2, 2 x ceil(|b - a| / 4)) and rise n x (a + b) / 2 height steps, and its vertical profile SHALL be a parabola so that the vertical curvature is constant along the piece. Because n is even the rise SHALL always be an integer.

#### Scenario: Flat to gentle

- **WHEN** `pitch:0>+1` is evaluated
- **THEN** it spans 2 tiles, rises 1 height step, and its arc length is 8.0826 m within 1e-3 m

#### Scenario: Crest

- **WHEN** `pitch:+2>-2` is evaluated
- **THEN** it spans 2 tiles with rise 0 and its apex is 1.0 m above its entry height at the middle of the piece

#### Scenario: Valley

- **WHEN** `pitch:-2>+2` is evaluated
- **THEN** it spans 2 tiles with rise 0, its lowest point is 1.0 m below its entry height, and the vertical radius at the lowest point is 8.0 m within 1e-6 m

### Requirement: Every piece respects minimum radii (Tier 1)

Every catalogue piece SHALL have a horizontal radius of at least 8.0 m and a vertical radius of at least 8.0 m at every point of its centreline. A catalogue entry that violates this SHALL fail the catalogue invariant test and SHALL NOT be loadable.

#### Scenario: Whole catalogue is checked

- **WHEN** the invariant test samples every piece at its compiled sample spacing
- **THEN** the smallest horizontal radius and the smallest vertical radius over all pieces are at least 8.0 m

#### Scenario: Too tight an entry

- **WHEN** a catalogue entry describes a curve of radius 1 tile (4 m)
- **THEN** loading the catalogue fails with an error naming the piece id and the radius

### Requirement: Drive zones are attributes of pieces (Tier 1)

A piece SHALL carry at most one drive mode: none, `lift`, `trim` brake or `station`; Tier 2 adds `block` brake and Tier 3 adds `booster` and `launch`. A `lift` variant SHALL exist for a slope or pitch piece when its entry class is at least 0, its exit class is at least 0 and they are not both 0, or when it is a crest (entry class above 0, exit class below 0). Curves, straights and the station SHALL NOT have a lift variant.

#### Scenario: Lift variants

- **WHEN** the Tier 1 lift variants are listed
- **THEN** there are 2 slope lifts (`slope:+1+lift`, `slope:+2+lift`), 6 pitch lifts between classes 0, +1 and +2, and 4 crest lifts (`pitch:+1>-1+lift`, `pitch:+1>-2+lift`, `pitch:+2>-1+lift`, `pitch:+2>-2+lift`)

#### Scenario: Lift on a curve is refused

- **WHEN** a catalogue entry combines a curve shape with the `lift` drive mode
- **THEN** loading the catalogue fails with an error naming the piece

### Requirement: Tiers gate which pieces exist (Tier 1)

Each piece SHALL declare tier 1, 2 or 3. A ride type SHALL declare the set of piece ids it allows, and the configuration value `Coaster:MaxTier` SHALL cap the tier offered. Pieces above `MaxTier` SHALL NOT appear in the palette or the catalogue endpoint and SHALL be rejected when appended or loaded from a file, with a message naming the tier.

#### Scenario: Tier cap

- **WHEN** `Coaster:MaxTier` is 1 and a design file contains a tier 2 piece
- **THEN** loading the file fails and the error names the piece id and says the tier is not enabled

#### Scenario: Default tier

- **WHEN** the configuration does not set `Coaster:MaxTier`
- **THEN** the maximum tier is 1 until Tier 2 is applied and tested, and then 2

### Requirement: Banking, steep slopes and helices (Tier 2)

Tier 2 SHALL add bank classes -1, 0 and +1 (a bank class of +1 is 45 degrees rolled to the right of the direction of travel), banked straights and banked curves whose bank side matches the turn direction, `roll:a>b` transitions that ease the roll with a smooth step over one tile, steep slope classes of +-4 and +-8, pitch transitions among all slope classes, a 180 degree banked helix, and the `block` brake. Radii limits and connection rules SHALL be unchanged.

#### Scenario: Banked curve matches its side

- **WHEN** the banked curves are listed
- **THEN** left curves have bank class -1 at entry and exit and right curves have +1, and no left curve with bank class +1 exists

#### Scenario: Steep pitch needs more tiles

- **WHEN** `pitch:0>+8` is evaluated
- **THEN** it spans 4 tiles and rises 16 height steps, and its minimum vertical radius is at least 8.0 m

### Requirement: Inversions and launches (Tier 3, stretch)

Tier 3 MAY add a vertical loop, a corkscrew, a zero-gravity roll, a booster and a launch. Each Tier 3 piece SHALL follow the same frame, join and radius rules, SHALL declare the minimum entry speed it is designed for, and a frictionless pass at that speed SHALL keep the vertical specific force between 0.3 g and 4.0 g everywhere on the piece.

#### Scenario: Loop design check

- **WHEN** a frictionless train enters a vertical loop piece at its declared minimum entry speed
- **THEN** the vertical specific force never leaves the range 0.3 g to 4.0 g on the piece

#### Scenario: Tier 3 absent by default

- **WHEN** `Coaster:MaxTier` is below 3
- **THEN** no loop, corkscrew, booster or launch piece is offered

### Requirement: The catalogue exposes local geometry (Tier 1)

The catalogue endpoint SHALL return, for every offered piece, its id, display name, group, tier, entry and exit frames, drive mode, analytic length and a local sample table (positions, tangents, rider up vectors and roll at the compiled sample spacing) for a piece that starts at the origin with heading 0, so a client can place and draw any piece with a rigid transform and without re-deriving curve maths.

#### Scenario: Local table of a straight

- **WHEN** the table of `straight` is read
- **THEN** it starts at (0, 0, 0), ends at (4.0, 0, 0) within 1e-9 m, and every tangent is (1, 0, 0)

#### Scenario: Stable ids

- **WHEN** the catalogue is read twice in the same build
- **THEN** ids, order and sample tables are identical byte for byte
