# Spec Delta

## Purpose

Defines the park map: its size, tile and height units, coordinate systems, the vertex-based terrain with its slope rules and edit cascade, and how placed objects map to simulation entities and scene nodes.

## ADDED Requirements

### Requirement: Park dimensions and units

A park SHALL be a rectangle of `width x height` tiles. Each dimension SHALL be a multiple of 16 in the range 16 to 256; the default SHALL be 128 x 128. A tile SHALL be a square of 2.0 m edge. Terrain heights SHALL be integer levels of 0.25 m each. Any other size SHALL be rejected when a park is created or configured.

#### Scenario: Default park

- **WHEN** a park is created without explicit dimensions
- **THEN** it is 128 x 128 tiles, which is 256 m x 256 m

#### Scenario: Dimension not a multiple of 16

- **WHEN** a park of 100 x 128 tiles is requested
- **THEN** creation is rejected with a validation error naming the width

#### Scenario: Dimension out of range

- **WHEN** a park of 272 x 128 tiles is requested
- **THEN** creation is rejected with a validation error naming the width

### Requirement: Coordinate systems

Tiles SHALL be addressed by `(x, y)` with `0 <= x < width`, `0 <= y < height`; terrain vertices by `(vx, vy)` with `0 <= vx <= width`, `0 <= vy <= height`. Tile `(x, y)` SHALL have corner vertices NW `(x, y)`, NE `(x+1, y)`, SW `(x, y+1)` and SE `(x+1, y+1)`. World space SHALL be right-handed with +Y up: vertex `(vx, vy)` at level `L` is at world `(2.0 * vx, 0.25 * L, 2.0 * vy)` metres. Directions: North is -y, East is +x, South is +y, West is -x.

#### Scenario: Vertex to world

- **WHEN** vertex `(10, 4)` has level 12
- **THEN** its world position is `(20.0, 3.0, 8.0)` metres

#### Scenario: Tile centre on flat ground

- **WHEN** tile `(3, 5)` has all four corners at level 8
- **THEN** its centre is at world `(7.0, 2.0, 11.0)` metres

### Requirement: Vertex terrain with a smoothness rule

Terrain SHALL be stored as one level per vertex in the range 0 to 63. A new park SHALL start with every vertex at level 8. At all times, any two vertices that are neighbours in the eight-neighbourhood (including diagonals) SHALL differ by at most one level. No operation SHALL leave the terrain violating this rule.

#### Scenario: New park is flat

- **WHEN** a park is created
- **THEN** every vertex is at level 8 and every tile is flat

#### Scenario: Rule holds after any sequence of edits

- **WHEN** 500 random raise and lower edits from a fixed seed are applied
- **THEN** after every accepted edit no pair of eight-neighbour vertices differs by more than one level

### Requirement: Tile slope classification

Every tile SHALL be classified from its four corner levels as exactly one of: **Flat** (all four equal), **Ramp** (two adjacent corners at level `h` and the other two at `h+1`; the rise direction is the side of the higher pair), or **Rough** (any other combination). The classification SHALL be a pure function of the corner levels. Ramp grade SHALL be 0.25 m over 2.0 m, which is 12.5 percent.

#### Scenario: Flat tile

- **WHEN** NW, NE, SW and SE are all 8
- **THEN** the tile is Flat

#### Scenario: Ramp rising East

- **WHEN** NW and SW are 8 and NE and SE are 9
- **THEN** the tile is a Ramp whose rise direction is East

#### Scenario: Single raised corner

- **WHEN** only NE is 9 and the other corners are 8
- **THEN** the tile is Rough

#### Scenario: Saddle

- **WHEN** NW and SE are 9 and NE and SW are 8
- **THEN** the tile is Rough

### Requirement: Terrain editing with a smoothing cascade

The system SHALL raise or lower one vertex by one level per edit. When the edit would break the smoothness rule, the system SHALL also move the affected neighbours by the minimum amount that restores the rule, propagating outward in breadth-first order with a fixed neighbour order (N, NE, E, SE, S, SW, W, NW). A single edit SHALL change at most 4096 vertices; an edit that would exceed it SHALL be rejected as too large. A level outside 0 to 63 SHALL be rejected. An edit SHALL be all or nothing.

#### Scenario: Single step needs no cascade

- **WHEN** vertex `(10, 10)` at level 8 in a flat park is raised once
- **THEN** only that vertex changes, to level 9

#### Scenario: Second step cascades to all eight neighbours

- **WHEN** vertex `(10, 10)` at level 9 is raised again to level 10
- **THEN** its eight neighbours move from 8 to 9
- **AND** nine vertex-levels were moved in total

#### Scenario: Edge vertex has fewer neighbours

- **WHEN** vertex `(0, 0)` is raised twice
- **THEN** exactly three neighbours move from 8 to 9

#### Scenario: Level out of range

- **WHEN** a vertex at level 63 is raised
- **THEN** the edit is rejected as a level out of range and no vertex changes

#### Scenario: Cascade too large

- **WHEN** an edit would change more than 4096 vertices
- **THEN** the edit is rejected as too large and no vertex changes

### Requirement: Terrain under objects is protected

A terrain edit that would change the level of any vertex belonging to a tile that holds a ground object or a path SHALL be rejected as blocked by an object, and no vertex SHALL change. This applies to the vertices moved by the cascade as well as the vertex the player chose.

#### Scenario: Edit next to a path is allowed, under it is not

- **WHEN** a path occupies tile `(5, 5)` and the player raises vertex `(9, 9)`
- **THEN** the edit succeeds
- **AND** raising vertex `(6, 6)`, a corner of tile `(5, 5)`, is rejected as blocked by an object

#### Scenario: Cascade reaches an occupied tile

- **WHEN** a cascade started away from any object would move a corner of an occupied tile
- **THEN** the whole edit is rejected as blocked by an object and nothing moves

### Requirement: Placed objects are entities and, when articulated, scene nodes

Every placed object (path tile, ride, stall, entrance, path addition) SHALL be an entity carrying its definition id, anchor tile, rotation (0 to 3 quarter turns), base level and a placement serial. The serial SHALL start at 1, increase by one for every successful placement (redo included) and SHALL never be reused. An object whose definition is marked as a scene node SHALL also get a scene node under a root node named `park_root`, named `<definition id with dashes as underscores>_<serial>`, positioned at the centre of its footprint at base level and rotated about +Y by `rotation x 90` degrees (right-handed). Paths and path additions SHALL NOT get scene nodes.

#### Scenario: Footprint rotation

- **WHEN** an object with a 2 x 1 footprint (u in 0..1, v = 0) is placed at anchor `(5, 5)` with rotation 1
- **THEN** it occupies tiles `(5, 5)` and `(5, 4)`
- **AND** the offset rotation for one quarter turn maps `(u, v)` to `(v, -u)`

#### Scenario: Pose of a rotated object

- **WHEN** the 2 x 1 object above is on flat ground at level 8
- **THEN** its scene node origin is at world `(11.0, 2.0, 10.0)` metres with a yaw of 90 degrees

#### Scenario: Pose of a large unrotated object

- **WHEN** a 10 x 10 marker block is placed at anchor `(20, 20)`, rotation 0, level 8
- **THEN** its scene node origin is at world `(50.0, 2.0, 50.0)` metres, yaw 0

#### Scenario: Node naming

- **WHEN** the 7th successful placement is an object with definition id `marker-10x10`
- **THEN** its scene node is named `marker_10x10_7`

#### Scenario: Paths have no node

- **WHEN** a path tile is placed
- **THEN** it is an entity but no scene node is created for it

### Requirement: Park state hash covers terrain and placements

The canonical state hash of a park SHALL cover the terrain levels, the placement serial counter and every placement entity, so that identical command histories yield identical hashes and any divergence is detectable.

#### Scenario: Same history, same hash

- **WHEN** the same ordered command log is applied to two fresh parks created with the same size and seed
- **THEN** both parks have the same state hash after every command

#### Scenario: Terrain change changes the hash

- **WHEN** one vertex is raised
- **THEN** the state hash differs from before the edit
