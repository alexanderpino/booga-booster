# Spec Delta

## Purpose

Defines how guests find their way along the footpath graph: integer flow fields per destination, deterministic tie-breaking, queue-path access rules, a per-tick work budget and cache invalidation when paths change. It must scale to thousands of guests without per-guest searches.

## ADDED Requirements

### Requirement: Routes follow precomputed distance fields per destination

For each destination class the system SHALL maintain one distance field over all path nodes: one per ride entrance, and one multi-source field each for nearest food, nearest drink, nearest toilet, nearest first aid, nearest information kiosk and the park exit. A guest SHALL move by repeatedly stepping to the neighbour with the smallest field distance. Distances SHALL be unsigned integers in millimetres, with no floating point in path-finding.

#### Scenario: Guest follows the shortest route

- **WHEN** a path graph has two routes from node A to the ride entrance, 24 m and 30 m long, and a guest at A heads for that ride
- **THEN** the guest walks the 24 m route and the field distance at A is 24 000

#### Scenario: Nearest facility is by walking distance, not straight line

- **WHEN** toilet T1 is 10 m away in a straight line but 80 m by path and toilet T2 is 30 m away by path
- **THEN** the nearest-toilet field at the guest's node is 30 000 and the guest heads for T2

#### Scenario: Unreachable destination

- **WHEN** a ride entrance is not connected to the guest's node
- **THEN** the field distance there is reported as unreachable, the ride is excluded from that guest's candidates and the guest does not start walking toward it

### Requirement: Ties are broken deterministically

Where two neighbours have equal field distance, the guest SHALL step to the neighbour with the lower node index. While building a field, equal-distance nodes SHALL be processed in ascending node index. The result SHALL not depend on thread count, insertion order or hash codes.

#### Scenario: Symmetric junction

- **WHEN** two neighbours N5 and N9 both have field distance 12 000
- **THEN** every guest at the junction steps to N5, on every run and at every worker count

#### Scenario: Equal fields built twice

- **WHEN** the same graph and destination are built on two separate runs
- **THEN** the distance and next-node arrays are element-for-element equal

### Requirement: Queue paths are entered only by guests heading for that ride

Edges flagged as belonging to a ride's queue path SHALL be excluded from every field except that ride's own field. Ride exit paths SHALL be traversable only outward. A wandering guest SHALL never step onto a queue path.

#### Scenario: Wanderer ignores queue path

- **WHEN** a wandering guest stands at the node where a queue path begins
- **THEN** it never steps onto the queue path edges

#### Scenario: Ride-bound guest enters the queue path

- **WHEN** a guest has chosen ride R and reaches the queue path start
- **THEN** it enters R's queue path and no other ride's field includes that path

### Requirement: Field construction is incremental under a per-tick budget

Field builds SHALL be resumable and SHALL consume at most `FlowFieldExpansionsPerTick` node expansions per tick in total (default 16 000), shared among active builds in creation order. A guest whose destination field is not yet complete SHALL keep wandering; it SHALL NOT wait on a build or trigger a per-guest search. The budget SHALL be a documented constant.

#### Scenario: A large graph is built over several ticks

- **WHEN** a field is requested on a 20 000-node graph with a budget of 16 000 and no other build is active
- **THEN** the field completes after at most 2 ticks and no tick performs more than 16 000 expansions for path-finding

#### Scenario: Budget is shared deterministically

- **WHEN** three builds are active with a budget of 16 000
- **THEN** build 1 receives 5 334 expansions and builds 2 and 3 receive 5 333 each on each tick, regardless of worker count

#### Scenario: No guest triggers a search

- **WHEN** 5 000 guests re-plan in the same tick
- **THEN** zero additional node expansions happen beyond the budgeted field builds

### Requirement: Fields are cached by graph version and replaced atomically

A field SHALL be keyed by destination and graph version. When the graph version changes, new builds SHALL start for fields in use and the previous field SHALL stay in force until the replacement completes, then be swapped in at a tick boundary. Fields for destinations that no longer exist (a removed ride) SHALL be released within one tick.

#### Scenario: Path edit

- **WHEN** a path tile is removed and the graph version increments
- **THEN** guests keep using the previous field until the new one completes, and a guest whose next edge no longer exists stops at its current node, re-chooses a goal and never walks the removed tile after the tick that applied the edit

#### Scenario: Ride removed

- **WHEN** a ride is demolished
- **THEN** its field is released within one tick and guests heading to it re-choose a goal

#### Scenario: Cache reuse

- **WHEN** 1 000 guests choose the same ride within one second and the graph version is unchanged
- **THEN** exactly one field exists for that ride and it is built at most once

### Requirement: Wandering is a deterministic random walk

A wandering guest at a node SHALL pick the next node uniformly among allowed neighbours excluding the node it came from, unless that is the only neighbour. The pick SHALL use the counter-based stream `(world seed, wander stream id, guest number, node arrivals count)`.

#### Scenario: Dead end

- **WHEN** a wandering guest reaches a dead-end node
- **THEN** it turns back to the node it came from

#### Scenario: Repeatable wandering

- **WHEN** the same scenario runs twice with the same seed
- **THEN** every guest visits the same node sequence

### Requirement: Path-finding scales to the target population

With 5 000 guests, 60 destination fields and a 20 000-node graph, steady-state path-following (stepping along fields) SHALL cost at most 0.25 ms per tick on one thread, and a complete rebuild of all fields SHALL finish within 90 ticks. The numbers are targets recorded by the benchmark; exceeding them by more than a factor of two SHALL stop the change for review.

#### Scenario: Stepping benchmark

- **WHEN** the movement benchmark runs 5 000 walking guests for 1 000 ticks
- **THEN** the mean time per tick is recorded and is at most 0.5 ms (twice the target) or the benchmark fails

#### Scenario: Rebuild benchmark

- **WHEN** the graph version changes with 60 fields in use
- **THEN** all replacements complete within 90 ticks at the default budget
