# Spec Delta

## Purpose

Defines how a built coaster runs in the park: the station cycle, one train in Tier 1 and several trains with capacity, guest loading, block sections and collision avoidance in Tier 2, and the hooks through which breakdowns and crashes enter and leave the ride.

## ADDED Requirements

### Requirement: A built coaster runs a train through the station cycle (Tier 1)

A built and open coaster SHALL run one train continuously through the phases Unloading (3.0 s), Loading (at least 5.0 s), ReadyToDispatch (2.0 s), OnTrack and Arriving, counted in whole simulation steps. With no guest source configured the train SHALL still run (empty run) so the ride is visible. Phase durations are configuration values with these defaults.

#### Scenario: Empty cycle

- **WHEN** an open coaster has no guest source
- **THEN** the train leaves the station 10.0 s (1,200 steps) after it stopped, within one step

#### Scenario: Closing

- **WHEN** the ride is closed while the train is on the track
- **THEN** the train completes its lap, stops at the platform and stays there

### Requirement: Opening needs a passed test run (Tier 1)

A coaster SHALL be openable only when its design is Built and its last test run for the current hash has verdict `Pass` or `PassWithWarnings`. A Built coaster that is Closed SHALL leave its train in the station, SHALL not publish occupancy, and SHALL publish its status as Closed.

#### Scenario: Unsafe design

- **WHEN** the last test verdict is `Fail`
- **THEN** opening is refused with the verdict and the first failing finding

### Requirement: Trains have a capacity (Tier 2)

A coaster SHALL run 1 to 8 trains of 1 to 8 cars; each car SHALL have 2 seats (4 as an option); capacity SHALL be cars x seats per car. The train count SHALL NOT exceed the number of blocks. Adding or removing a train SHALL be a command applied while the ride is closed.

#### Scenario: Capacity

- **WHEN** a train has 4 cars of 2 seats
- **THEN** its capacity is 8

#### Scenario: More trains than blocks

- **WHEN** 3 trains are configured on a layout with 2 blocks
- **THEN** the command is rejected with a message naming both counts

### Requirement: The station hands guests over through a seat buffer (Tier 2)

During Unloading the guest system SHALL release seated guests and clear their seats; the train SHALL stay in Unloading until the occupied count is 0 or 8.0 s have passed. During Loading the guest system SHALL write guest ids into free seats and raise the occupied count and the payload mass in the same step; the train SHALL leave Loading when it is full, or after 30.0 s, and not before 5.0 s. The coaster SHALL never read or write guest data other than the seat buffer, the occupied count and the payload mass.

#### Scenario: Full train

- **WHEN** 8 guests are seated 2.0 s after Loading starts
- **THEN** the train enters ReadyToDispatch at 5.0 s and leaves 2.0 s later

#### Scenario: Partly full

- **WHEN** 3 guests are seated and no more arrive
- **THEN** the train enters ReadyToDispatch at 30.0 s

#### Scenario: Inconsistent buffer

- **WHEN** the occupied count says 5 but 6 seats are non-empty
- **THEN** the coaster recounts to 6, increments the `seat_repairs` counter, and proceeds

#### Scenario: Empty ride and no empty runs

- **WHEN** the empty-run option is off and no guest boards
- **THEN** the train stays in the station and does not dispatch

### Requirement: Payload mass changes only in the station (Tier 2)

The mass of a train SHALL be cars x 450 kg plus the payload mass written by the guest system, and SHALL be read at dispatch and held for the lap.

#### Scenario: Mass held

- **WHEN** the payload mass is rewritten while the train is on the track
- **THEN** the lap uses the mass read at dispatch

### Requirement: Block sections keep trains apart (Tier 2)

A layout with block brakes SHALL be divided into blocks; each block SHALL start at the exit of a block brake (or the station departure) and end at the exit of the next block brake (or the platform). A block SHALL be at least train length + 4.0 m long, else `BlockTooShort` is an error. A train SHALL leave a block brake only if the next block was free at the end of the previous step; otherwise it SHALL stop with its lead car 0.5 m before the end of the brake zone and wait. A free block releases the train with a drive to 3.0 m/s.

#### Scenario: Hold and release

- **WHEN** train A occupies block 2 and train B reaches the end of block 1
- **THEN** B stops at the hold point, and starts again within 2 steps after A has left block 2 completely

#### Scenario: Short block

- **WHEN** a block is 10.0 m long for a 4-car train (8.0 m)
- **THEN** `BlockTooShort` is reported because 10.0 m is below 12.0 m

### Requirement: Trains never overlap under any dispatch schedule (Tier 2)

For any number of trains, any dispatch schedule and any seeded loading delays, the gap between the rear of a train and the front of the train behind it SHALL stay non-negative at every step. A property test SHALL check 200 seeded schedules for 600 s each. The test run SHALL report `BlockBrakeTooShort` when the entry speed at a block brake is higher than sqrt(2 x 3.0 x L) for its brake run length L, because such a train could not be held.

#### Scenario: Property

- **WHEN** 200 seeded schedules with 4 trains on the starter layout with block brakes run for 600 s
- **THEN** the minimum gap over all steps and all schedules is not below 0

#### Scenario: Unholdable entry

- **WHEN** the test run enters a 4.0 m block brake at 6.0 m/s
- **THEN** `BlockBrakeTooShort` is reported, because 6.0 m/s needs 6.0 m and the run has 4.0 m

### Requirement: Faults stop the ride safely (Tier 2)

A fault command SHALL set the ride fault state to one of `None`, `SafetyCutOut` or `RestraintsOpen`. While a fault is set no train SHALL be dispatched, every block brake SHALL hold, the lift SHALL stop with the anti-rollback holding the train, and trains in free running SHALL continue until a brake holds them. Clearing the fault SHALL release held trains in order, nearest the station first. Each fault set or cleared SHALL emit an incident event.

#### Scenario: Cut-out mid-lap

- **WHEN** `SafetyCutOut` is set while a train is on a downhill run toward a block brake
- **THEN** the train is stopped by that brake and the ride reports `Broken`

#### Scenario: Recovery order

- **WHEN** two trains are held and the fault is cleared
- **THEN** the train nearer to the station in running order is released first

### Requirement: Crashes are detected and reported (Tier 2)

A step in which the gap between two trains is negative, or in which a train sets the `Overspeed` flag, SHALL put the involved trains into phase Crashed, stop them, emit an incident event with the kind (`Collision` or `Overspeed`), coaster, train indexes and arc length, and set the ride status to Crashed. A Crashed ride SHALL stay closed until a clear command is given.

#### Scenario: Collision

- **WHEN** block protection is switched off in a test and two trains overlap
- **THEN** both are Crashed, one `Collision` incident is emitted and the ride is Crashed

### Requirement: Coasters run in parallel without changing results (Tier 2)

Trains SHALL be stepped in parallel in fixed chunks; a train's step SHALL read only its own state, the track tables and the block occupancy as left by the previous step, and write only its own state. Block occupancy and crash detection SHALL be updated afterwards, one coaster per job, writing only that coaster's data. The world state hash after N steps SHALL be identical for every worker count.

#### Scenario: Worker counts

- **WHEN** 50 coasters with 3 trains each run 1,200 steps with 1, 2, 4 and 8 workers
- **THEN** the world state hash is identical in all four runs

#### Scenario: Occupancy lag is safe

- **WHEN** train A enters a block in step n
- **THEN** train B behind it sees that block as occupied from step n + 1 and cannot enter it in step n, because B is held in the block before until A has left that block
