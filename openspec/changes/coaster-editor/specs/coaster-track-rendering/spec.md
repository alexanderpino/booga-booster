# Spec Delta

## Purpose

Defines how the browser turns a coaster design into Three.js geometry: composition of the track from catalogue geometry and placed frames, procedural rails, ties and supports, the ghost preview of the next piece, incremental updates and the mesh budgets that keep many coasters cheap to draw.

## ADDED Requirements

### Requirement: The client composes the track from catalogue geometry (Tier 1)

The client SHALL build the track sample table by placing each piece's local sample table with the piece's placed frame (a lattice translation and a rotation by a multiple of 90 degrees about the vertical axis) and concatenating the pieces. It SHALL NOT re-evaluate curve formulas. Positions SHALL be kept in double precision until upload to the GPU. A shared fixture of the starter coaster (sample count, total length and the position of every 50th sample, produced by the server tests) SHALL be reproduced by the client within 1e-6 m.

#### Scenario: Starter matches the server fixture

- **WHEN** the client composes the starter coaster
- **THEN** its total length is 313.27 m within 0.01 m and every fixture sample position matches within 1e-6 m

#### Scenario: Rotation exactness

- **WHEN** the same design is placed with heading 1 instead of 0
- **THEN** every composed position equals the rotated heading 0 position exactly

### Requirement: Rails, ties and spine are generated procedurally (Tier 1)

For every piece the client SHALL generate two rails as tubes of radius 0.04 m with 6 radial segments, centred 0.5 m to each side of the centreline along the rider's lateral vector, one centre spine beam 0.25 m wide and 0.3 m high below the centreline, and ties 1.2 m x 0.06 m x 0.12 m every 0.5 m of arc length, oriented by the rider frame. Mesh sample spacing along the rails SHALL be 0.5 m (every second compiled sample). Station platforms SHALL add a deck 12.0 m long and 3.0 m wide beside the track.

#### Scenario: Straight piece

- **WHEN** the 4.0 m `straight` piece is meshed
- **THEN** each rail has 9 ring positions, 8 tie placements are generated and rail ring centres are exactly 1.0 m apart

#### Scenario: Banked piece (Tier 2)

- **WHEN** a piece with bank class +1 is meshed
- **THEN** the two rails are on a line tilted 45 degrees from horizontal within 1e-6 radians

### Requirement: Supports are generated from track and terrain (Tier 1)

The client SHALL place a support column of radius 0.15 m every 4.0 m of arc length from the track underside down to the terrain height at that point, where the underside is at least 1.0 m above the terrain, and SHALL add cross braces between consecutive columns taller than 6.0 m. It SHALL NOT place a column over a path tile. A column SHALL end on the terrain with a 0.6 m square footing.

#### Scenario: Column height

- **WHEN** the underside of the track is 9.5 m above the terrain at a support point
- **THEN** the column is 9.5 m tall within 1 mm

#### Scenario: Low track

- **WHEN** the underside is 0.6 m above the terrain
- **THEN** no column is generated at that point

#### Scenario: Over a path

- **WHEN** the terrain tile below a support point is a path
- **THEN** no column is generated there

### Requirement: Meshes stay within a fixed budget (Tier 1)

A coaster of 300 m of track SHALL be drawn with at most 6 draw calls and at most 50,000 triangles, using one merged mesh for rails and spine, instanced meshes for ties, columns and braces. Building the mesh of a 100-piece design SHALL take at most 40 ms in the reference test environment; the figure is recorded, not asserted in unit tests.

#### Scenario: Count check

- **WHEN** the starter coaster is meshed
- **THEN** the triangle count is at most 50,000 and the draw-call count at most 6

### Requirement: Appending a piece updates only that piece (Tier 1)

When a design response adds one piece, the client SHALL generate and add the geometry of that piece only, leaving all earlier objects unchanged (same object references). Removing the last piece SHALL remove and dispose only its objects. A full rebuild SHALL happen only when the catalogue version, the terrain or a non-trailing piece changes. Appending SHALL take at most 5 ms for a piece in the reference test environment.

#### Scenario: References are stable

- **WHEN** a piece is appended to a meshed design
- **THEN** the objects of the earlier pieces keep their identity

### Requirement: The ghost shows the next piece at once (Tier 1)

Selecting a palette entry SHALL show a ghost of that piece at the end of the track in the same animation frame, built from the cached catalogue geometry without a server request. The ghost SHALL be drawn at 45 % opacity. When the preview answer arrives the ghost SHALL show valid or invalid by an outline colour and by an icon and text label; invalid SHALL state the first error. The ghost SHALL be removed on append, cancel or when the palette is empty.

#### Scenario: No request for geometry

- **WHEN** a palette entry is selected with the catalogue already cached
- **THEN** a ghost mesh exists in the scene before any network request completes

#### Scenario: Not colour only

- **WHEN** the preview is invalid
- **THEN** the ghost has an error icon and a text label in addition to the colour

### Requirement: Resources are released (Tier 1)

Every geometry, material and texture created for a coaster SHALL be disposed when its objects are removed or the editor is destroyed. A test SHALL count creations and disposals through a spy and require equality after a build, a series of appends, removals and a teardown.

#### Scenario: No leak

- **WHEN** a design is meshed, 20 pieces are appended and removed and the view is destroyed
- **THEN** disposal calls equal creation calls for geometries and materials

### Requirement: Test-run markers are placed on the track (Tier 1)

For each marker of a test result the client SHALL place a visible 3D marker at its arc length, with a text label in a DOM overlay that is readable by assistive technology and linked to the marker list, and SHALL highlight the pieces involved. Warn markers and hard or stall markers SHALL differ by shape and label, not only by colour.

#### Scenario: Stall marker

- **WHEN** the result contains a stall at arc length 150.0 m
- **THEN** a marker is placed at the composed position for 150.0 m and its label names the piece

### Requirement: Distant coasters are simplified (Tier 3)

Coasters farther than 150 m from the camera SHALL be drawn as one ribbon mesh with no ties or columns, swapping back without a visible pop when the camera approaches.

#### Scenario: Far coaster

- **WHEN** the camera is 300 m from a coaster
- **THEN** the coaster uses 1 draw call
