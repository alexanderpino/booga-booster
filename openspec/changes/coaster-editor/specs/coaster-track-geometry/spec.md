# Spec Delta

## Purpose

Defines how a coaster design becomes a continuous curve: the analytic pieces, their continuity at joins, the arc-length parametrisation that lets physics address the track by distance, the rider frames, and the compiled track tables that physics, metrics and rendering all read.

## ADDED Requirements

### Requirement: Pieces are analytic curves, the track is their concatenation (Tier 1)

Each piece SHALL be an exact curve built from line, circular arc (with optional constant slope, a helix) or parabolic pitch primitives, evaluated in the piece's local frame and placed by a rigid motion made of a lattice translation and a rotation by a multiple of 90 degrees about the vertical axis. The track SHALL be the concatenation of the placed pieces in design order. Rotations by multiples of 90 degrees SHALL use exact cosine and sine values of 0, 1 and -1.

#### Scenario: Rotation is exact

- **WHEN** a design is rotated by 90 degrees about the anchor
- **THEN** every compiled position equals the rotated original position exactly (bit for bit), because only exact trigonometric values are used

#### Scenario: Circle point

- **WHEN** `curve:L4` is placed at the origin with heading 0 and evaluated at the point where it has turned 45 degrees
- **THEN** the position is (11.3137, 0, -4.6863) within 1e-4 m

### Requirement: Joins are continuous in position and tangent (Tier 1)

At every join between two connected pieces the end position SHALL equal the next start position within 1e-9 m and the end tangent SHALL equal the next start tangent within 1e-9. Curvature MAY jump at a join. Slope and bank classes make this hold by construction and a property test SHALL check it on random designs.

#### Scenario: Random walks

- **WHEN** 200 designs of up to 80 palette-chosen pieces are generated with fixed seeds and compiled
- **THEN** every join meets the position and tangent tolerances

#### Scenario: Starter closes

- **WHEN** the starter coaster is compiled
- **THEN** the end of the last piece equals the start of the first piece within 1e-9 m and 1e-9 in tangent

### Requirement: The track is parametrised by arc length (Tier 1)

The compiled track SHALL hold samples at increasing arc length. Each piece of analytic length L SHALL be divided into n = max(1, round(L / 0.25 m)) equal arc-length intervals, so the spacing inside a piece is L / n. Sample positions SHALL lie on the exact curve within 1e-9 m at their arc length. The total length SHALL equal the sum of the analytic piece lengths, not the polyline length.

#### Scenario: Curve sampling

- **WHEN** `curve:L4` (arc 25.1327 m) is compiled
- **THEN** it has n = 101 intervals of 0.24884 m each within 1e-5 m

#### Scenario: Arc length of a parabola

- **WHEN** `pitch:0>+1` is compiled
- **THEN** the sum of its sample intervals equals its analytic length 8.0826 m, and the polyline length of its samples differs from it by less than 1e-3 m

### Requirement: Lookup by distance is exact at samples and bounded between (Tier 1)

Evaluating the track at an arc length s SHALL return position, tangent, curvature vector, rider up vector and roll by interpolation between the two neighbouring samples. At a sample the result SHALL be the stored value. Between samples the interpolated position SHALL lie within 1.0 mm of the analytic curve for every catalogue piece. For a closed track the argument SHALL wrap modulo the total length. A lookup that starts from a stored sample index (a hint) SHALL return the same result as one without a hint.

#### Scenario: Wrap

- **WHEN** the track is evaluated at s = total length + 1.0 m
- **THEN** the result equals the evaluation at s = 1.0 m

#### Scenario: Hint equals no hint

- **WHEN** 10,000 random distances are evaluated with a correct hint, a hint one piece away and no hint
- **THEN** all three results are identical

### Requirement: Rider frames are orthonormal and respect roll (Tier 1)

At every sample the tangent T, the rider up vector U and the lateral vector L (to the rider's right, L = T x U) SHALL be unit length and mutually orthogonal within 1e-12. With bank class 0 and no pitch, U SHALL be the world up vector. Frames SHALL be propagated along the track without twisting (each frame is the previous one carried along the curve, then rolled by the piece's roll), so a flat unbanked circuit has U equal to world up everywhere.

#### Scenario: Flat circuit

- **WHEN** a flat unbanked circuit is compiled
- **THEN** every up vector is (0, 1, 0) within 1e-12

#### Scenario: Banked roll

- **WHEN** a sample has bank class +1 (Tier 2)
- **THEN** the angle between U and the world up vector, measured about T, is 45 degrees within 1e-9 radians

### Requirement: Curvature vector and potential height are tabulated (Tier 1)

For every sample the compiled track SHALL store the curvature vector dT/ds, the heading angle, the height of the centreline and the zone code (none, lift, trim brake, station platform, block brake). For a train of n cars spaced 2.0 m apart it SHALL provide the car-averaged height `Hbar_i = mean over k of h(s_i - 2.0 k)` at each sample, wrapping for closed tracks, built lazily and cached per (track, car count). The dynamics SHALL read only the height table, the zone table and the segment lengths.

#### Scenario: Single car equals centreline height

- **WHEN** the potential table is built for a 1-car train
- **THEN** `Hbar_i` equals the centreline height at every sample

#### Scenario: Cache

- **WHEN** the table for 4 cars is requested twice
- **THEN** the second call returns the same instance without recomputation

### Requirement: Compilation is deterministic and fast (Tier 1)

Compiling a design SHALL be a pure function of the piece list, anchor and catalogue, and SHALL use only IEEE-exact operations (addition, subtraction, multiplication, division, square root) in the dynamics-facing tables; trigonometric functions MAY be used only while generating piece geometry. Compiling a 200-piece design SHALL meet the performance budget of the `coaster-determinism-and-performance` capability.

#### Scenario: Repeatability

- **WHEN** a design is compiled twice in one process
- **THEN** all table values are bit-identical

#### Scenario: Prefix reuse

- **WHEN** one piece is appended to a compiled design
- **THEN** the compile of the new design reuses the tables of the first N pieces unchanged and only compiles the new piece
