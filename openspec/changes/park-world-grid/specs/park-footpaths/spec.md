# Spec Delta

## Purpose

Defines footpaths and the places where objects meet them: normal and queue path tiles, automatic connectivity, attach points, the park entrance, the derived path graph, distance fields for navigation and the connectivity report.

## ADDED Requirements

### Requirement: Path tiles and their placement terrain

A path tile SHALL be either **Normal** or **Queue**. It SHALL be placeable only on a Flat tile or a Ramp tile with no other ground occupant; on a Ramp the path SHALL take the ramp's rise direction. A path SHALL need 12 levels (3.0 m) of free headroom above its base. A path SHALL NOT be placeable on a Rough tile.

#### Scenario: Path on flat ground

- **WHEN** a Normal path is placed on a Flat tile
- **THEN** the tile holds a Normal path with no slope

#### Scenario: Path on a ramp

- **WHEN** a Normal path is placed on a Ramp tile rising East
- **THEN** the path records a rise direction East

#### Scenario: Path on rough ground

- **WHEN** a path is placed on a Rough tile
- **THEN** the placement is rejected with reason `TerrainRough`

### Requirement: Automatic connectivity

Each path tile SHALL expose walk edges: all four edges for a Flat path; for a Ramp path only the two edges across the rise direction (the low end and the high end). Two adjacent path tiles SHALL be connected across their shared edge exactly when both tiles have that edge as a walk edge. Connectivity SHALL be stored as a four-bit mask with North 1, East 2, South 4, West 8 and SHALL be recomputed automatically whenever a neighbouring tile changes. Normal and Queue tiles connect to each other by the same rule.

#### Scenario: Two flat neighbours connect

- **WHEN** Normal paths exist at `(4, 4)` and `(5, 4)`, both flat
- **THEN** `(4, 4)` has East set and `(5, 4)` has West set

#### Scenario: Ramp joins a higher flat path at its high end

- **WHEN** a ramp path at `(4, 4)` rises East from level 8 to 9 and a flat path at level 9 exists at `(5, 4)`
- **THEN** `(4, 4)` has East set and `(5, 4)` has West set

#### Scenario: Ramp sides stay closed

- **WHEN** a ramp path at `(4, 4)` rises East and another path exists at `(4, 3)`
- **THEN** `(4, 4)` does not have North set

#### Scenario: Removal updates neighbours

- **WHEN** the path at `(5, 4)` is removed
- **THEN** `(4, 4)` no longer has East set
- **AND** at most the removed tile and its four neighbours were recomputed

### Requirement: Attach points

A placeable definition MAY declare attach points, each with a kind, a footprint cell and a facing. Kinds SHALL be `QueueEntry`, `RideExit`, `ShopFront` and `ParkGate`. An attach point SHALL be connected when the tile next to its cell in the facing direction holds a path of an accepted type whose walk edge faces the cell: `QueueEntry` accepts only a Queue path; `RideExit`, `ShopFront` and `ParkGate` accept only a Normal path. When connected, the path tile SHALL set an attach bit for that edge. Attach cells and facings SHALL rotate with the placement.

#### Scenario: Queue entry connects to a queue path

- **WHEN** a ride's `QueueEntry` faces East from cell `(9, 5)` and a Queue path exists at `(10, 5)`
- **THEN** the attach point is connected and the path at `(10, 5)` has its West attach bit set

#### Scenario: Wrong path type

- **WHEN** a `QueueEntry` faces a Normal path
- **THEN** the attach point is not connected

#### Scenario: Facing rotates with the object

- **WHEN** an attach point declared facing East is on an object placed with rotation 1
- **THEN** it faces North, since one quarter turn maps East to North

#### Scenario: Connection follows later path changes

- **WHEN** the queue path next to a connected entry is removed
- **THEN** the attach point becomes unconnected without any change to the object

### Requirement: The park entrance

The park SHALL allow exactly one park entrance. Its footprint SHALL be 3 tiles wide (u 0..2) and 2 tiles deep (v 0..1); the outer row is v = 0. It SHALL declare a `ParkGate` attach point at cell `(1, 1)` facing +v (South at rotation 0). It SHALL be placeable only where every outer-row cell lies on the map boundary, with the cells beyond the outer row (opposite the facing) outside the map. The cell `(1, 0)` SHALL be the guest spawn tile. A second entrance SHALL be rejected.

#### Scenario: Entrance on the north boundary

- **WHEN** the entrance is placed at anchor `(10, 0)` with rotation 0
- **THEN** it occupies x 10 to 12 and y 0 to 1
- **AND** its gate attach cell is `(11, 1)` and the tile to connect is `(11, 2)`

#### Scenario: Entrance on the west boundary

- **WHEN** the entrance is placed at anchor `(0, 2)` with rotation 1
- **THEN** it occupies x 0 to 1 and y 0 to 2
- **AND** its gate attach cell is `(1, 1)` facing East, so the tile to connect is `(2, 1)`

#### Scenario: Entrance away from the boundary

- **WHEN** the entrance is placed at anchor `(10, 5)` with rotation 0
- **THEN** the placement is rejected with reason `EntranceNotOnBoundary`

#### Scenario: Second entrance

- **WHEN** a park entrance already exists and another is placed
- **THEN** the placement is rejected with reason `MaxPerParkReached`

### Requirement: Derived path graph

The system SHALL expose a read-only path graph whose nodes are path tiles and whose edges are the connections. Neighbour enumeration SHALL use the fixed order North, East, South, West. Two views SHALL exist: **Any** (Normal and Queue tiles) and **Wander** (Normal tiles only, never entering a Queue tile). The graph SHALL carry a version number that increases on every path or attach change.

#### Scenario: Wander view excludes queue tiles

- **WHEN** a Queue tile lies between two Normal tiles on a line
- **THEN** in the Wander view the first Normal tile has no route to the third
- **AND** in the Any view it has

#### Scenario: Version changes only on path change

- **WHEN** a ride is placed away from any path
- **THEN** the path graph version is unchanged

### Requirement: Distance fields

Given a set of source tiles and a view, the system SHALL compute for every path tile the number of tile steps to the nearest source by breadth-first search. Unreachable tiles SHALL hold 65535. The result SHALL be deterministic: same map, same sources, same field. For navigation, the next step from a tile SHALL be the neighbour with the smallest distance, ties broken by the order North, East, South, West. A field SHALL be tied to the graph version it was built for and SHALL be reported stale when the version differs.

#### Scenario: Straight line

- **WHEN** five connected Normal tiles lie in a row and the first is the source
- **THEN** their distances are 0, 1, 2, 3 and 4

#### Scenario: Tie-break

- **WHEN** a tile has two neighbours at equal smallest distance, one North and one East
- **THEN** the next step is North

#### Scenario: Unreachable

- **WHEN** a Normal tile is not connected to any source
- **THEN** its distance is 65535 and it has no next step

#### Scenario: Stale field

- **WHEN** a path tile is placed after a field was built
- **THEN** the field reports stale

### Requirement: Connectivity report

The system SHALL produce a connectivity report with: whether a park entrance exists; every attach point that is not connected, with its object and kind; and every object having a `QueueEntry`, `RideExit` or `ShopFront` that cannot be reached from the entrance's gate tile through the Any view. The report SHALL be a pure read of current state.

#### Scenario: No entrance

- **WHEN** the park has no entrance
- **THEN** the report states that no entrance exists

#### Scenario: Ride with unconnected exit

- **WHEN** a ride's `QueueEntry` is connected but its `RideExit` is not
- **THEN** the report lists the exit attach point as unconnected

#### Scenario: Island ride

- **WHEN** a ride is connected to a path network that is not connected to the entrance's gate tile
- **THEN** the report lists the ride as unreachable

### Requirement: Footpath performance targets

On a 128 x 128 map, rebuilding all connectivity SHALL take at most 5 ms; computing a distance field over 4000 path tiles SHALL take at most 3 ms; computing connectivity for one placement SHALL touch at most five path tiles. These are targets recorded by benchmark and SHALL NOT be asserted as timings in unit tests.

#### Scenario: Benchmark recorded

- **WHEN** the park benchmarks are run
- **THEN** the results for full connectivity rebuild and for a 4000-tile distance field are written to `docs/park/01-park-world-grid.md` with the machine description
