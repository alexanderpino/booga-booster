# Spec Delta

## Purpose

Defines how any catalog ride is operated: the generic controller interface, reuse of the existing seven-state ride lifecycle, the open/closed/broken status, automatic dispatch, timed rides without physics, the digital-twin Octopus as one adapter, and the breakdown hook.

## ADDED Requirements

### Requirement: The ride lifecycle of ride-state-machine is reused unchanged

Every placed ride SHALL have exactly one lifecycle state from Idle, Loading, Safe, Started, Stopping, Offloading and Emergency Stop, with the same legal operator transitions, guards, automatic transitions and motion confinement as the published `ride-state-machine` capability. A new ride SHALL be Idle. A transition not in that table SHALL be refused and leave the state unchanged. The same transition table SHALL be verified against both the generic lifecycle and the DigitalTwin ride by one shared data file.

#### Scenario: Undefined transition

- **WHEN** a ride is Idle and a transition to Started is requested
- **THEN** it is refused as an invalid transition and the state stays Idle

#### Scenario: Shared table

- **WHEN** the shared transition table is run against the generic lifecycle and against the DigitalTwin ride
- **THEN** both accept and refuse exactly the same pairs

#### Scenario: Emergency stop from Idle

- **WHEN** an emergency stop is requested on an Idle ride
- **THEN** it is refused and the state stays Idle

### Requirement: Operating status overlay

Each ride SHALL also have an operating status of Closed, Open or Broken, independent of the lifecycle state. A new ride SHALL be Closed. Only an Open ride SHALL start new cycles. Opening SHALL be refused with `RideBroken` when Broken, `QueueEntryNotConnected` or `RideExitNotConnected` when the corresponding attach point is not connected to a path in the park grid, and `BehaviourUnavailable` when the ride's behaviour is not provided yet (coasters until `coaster-editor`). Closing SHALL let the cycle in progress finish and then keep the ride Idle.

#### Scenario: Open without a queue path

- **WHEN** an Open request is made for a ride whose queue entry is not connected
- **THEN** it is refused with `QueueEntryNotConnected` and the status stays Closed

#### Scenario: Open a connected ride

- **WHEN** both attach points are connected and Open is requested
- **THEN** the status becomes Open and the ride starts loading

#### Scenario: Close during a run

- **WHEN** a ride is Started and Close is requested
- **THEN** the status becomes Closed, the run completes through Stopping and Offloading, and the ride stays Idle afterwards

#### Scenario: Coaster before its editor exists

- **WHEN** Open is requested for a placed `family-coaster`
- **THEN** it is refused with `BehaviourUnavailable`

### Requirement: Automatic dispatch of a scripted ride

For a ride with behaviour `scripted-cycle` that is Open and Idle, the system SHALL enter Loading. While Loading, it SHALL request Safe when either the ride is full and the minimum load time has elapsed, or the maximum load time has elapsed, and in the second case only if at least one rider is aboard or empty cycles are enabled (default enabled until `guests-and-needs`). When the ride is Closed, Safe SHALL be requested as soon as the minimum load time has elapsed so the cycle can end. After Safe it SHALL request Started. Started SHALL last the ride time, then Stopping SHALL be requested; Stopping ends when the motion is at rest; Offloading lasts the unload time and then all riders are gone and the ride is Idle.

#### Scenario: Empty carousel cycle

- **WHEN** an Open `starlight-carousel` (load 15/45 s, ride 75 s, ramp down 6 s, unload 15 s) enters Loading at tick 0 with empty cycles enabled
- **THEN** it requests Safe at tick 5400 (45 s), Started within 2 ticks later, Stopping 9000 ticks (75 s) after Started, Offloading 720 ticks (6 s) after Stopping, and Idle 1800 ticks (15 s) after Offloading

#### Scenario: Full ride leaves early

- **WHEN** 24 riders are aboard the carousel 1800 ticks (15 s) after Loading began
- **THEN** it requests Safe at tick 1800

#### Scenario: Not full and no riders, empty cycles disabled

- **WHEN** empty cycles are disabled and no rider boards
- **THEN** the ride remains in Loading past 45 s

#### Scenario: One rider at the maximum wait

- **WHEN** one rider is aboard and 45 s have elapsed
- **THEN** the ride requests Safe

### Requirement: Occupancy

Each ride SHALL have a capacity from its definition and an occupied count. A boarding operation SHALL accept at most the free capacity while the lifecycle state is Loading and the status is Open, and SHALL return the number accepted. At the end of the unload time the occupied count SHALL become zero. For scripted rides the guard `safe to start` SHALL hold when the state is Loading and at least one rider is aboard or empty cycles are enabled.

#### Scenario: Partial boarding

- **WHEN** 20 riders are aboard a carousel of capacity 24 and 10 more try to board
- **THEN** 4 are accepted and 6 are refused

#### Scenario: Boarding while running

- **WHEN** the ride is Started and a rider tries to board
- **THEN** none is accepted

### Requirement: Motion of a scripted ride

A scripted ride with motion kind spin SHALL rotate its `joint_motion` scene node about its configured axis at a rate that ramps linearly from 0 to the level's target rate over the ramp-up time while Started, and ramps linearly from its current rate to 0 over the ramp-down time while Stopping. The target rate SHALL be `maxRate * level / levelCount`. A swing ride SHALL oscillate with an angle of `envelope * profile(phase)`, where the envelope ramps like the rate, the amplitude at the top level is the configured amplitude scaled by `level / levelCount`, and the profile is a smooth periodic function of period `periodSeconds` computed with basic arithmetic only. The ride is at rest exactly when the envelope or rate is 0. A ramp time of 0 SHALL apply the target at once.

#### Scenario: Spin ramp

- **WHEN** a carousel at level 2 (target 0.4 rad/s, ramp-up 5 s) has been Started for 300 ticks
- **THEN** its spin rate is 0.2 rad/s

#### Scenario: Spin at full rate

- **WHEN** it has been Started for 7 s
- **THEN** its rate is 0.4 rad/s and over the next 120 ticks the angle advances by 0.4 rad

#### Scenario: Ramp down to rest

- **WHEN** Stopping begins at rate 0.4 with ramp-down 6 s
- **THEN** after 360 ticks the rate is 0.2 rad/s and after 720 ticks it is 0 and the ride is at rest

#### Scenario: Deterministic angle

- **WHEN** the same ride is simulated for 1000 ticks twice
- **THEN** the joint angles are bit-identical

### Requirement: Operating level

A ride SHALL have an operating level from 1 to its level count, starting at the definition's default. Changing it SHALL be refused with `LevelOutOfRange` outside that range and with `LevelLockedWhileRunning` while the lifecycle state is Started, Stopping or Emergency Stop. A change SHALL update the ride's ratings at once.

#### Scenario: Out of range

- **WHEN** level 4 is requested for a ride with 3 levels
- **THEN** it is refused with `LevelOutOfRange`

#### Scenario: While running

- **WHEN** a level change is requested while Started
- **THEN** it is refused with `LevelLockedWhileRunning`

### Requirement: Emergency stop

An emergency stop request SHALL follow the ride-state-machine rules (accepted from Loading, Safe, Started and Stopping). The motion SHALL ramp to rest within 2 seconds (240 ticks) regardless of the normal ramp-down time. After the ride reaches Offloading and then Idle, its status SHALL be Closed.

#### Scenario: Stop a running carousel

- **WHEN** an emergency stop is requested while Started at rate 0.4
- **THEN** the rate reaches 0 within 240 ticks, the state becomes Offloading, then Idle, and the status is Closed

### Requirement: Breakdown hook

An Open ride with a breakdown section SHALL have a reliability from 0 to 10000, starting at 10000. Every 600 ticks (5 s) while Open, the system SHALL subtract the definition's `baseUnreliability` (floored at 0), then draw an integer `d` from 0 to 9999 from a per-ride deterministic stream, and the ride SHALL break down when `d < (10000 - reliability) / 4` (truncating). A breakdown SHALL set the status to Broken, request an emergency stop if the lifecycle state is Loading, Safe, Started or Stopping, and publish `ride-broke-down`. Repair SHALL restore reliability to 10000, set the status to Closed and publish `ride-repaired`; repair of a ride that is not Broken SHALL be refused with `NotBroken`. Draws SHALL depend only on the world seed, the ride and the check number.

#### Scenario: First check

- **WHEN** a ride with `baseUnreliability` 20 makes its first check and the draw is 4
- **THEN** reliability becomes 9980, the threshold is 5, and the ride breaks down

#### Scenario: Draw at the threshold

- **WHEN** the same check draws 5
- **THEN** the ride does not break down

#### Scenario: Break while running

- **WHEN** a ride breaks down while Started
- **THEN** its status is Broken and an emergency stop brings it to rest and to Idle

#### Scenario: Repair

- **WHEN** a Broken ride is repaired
- **THEN** its reliability is 10000, its status is Closed and `ride-repaired` is published

#### Scenario: Reproducible

- **WHEN** two worlds with the same seed run the same commands for 100000 ticks
- **THEN** the same breakdowns occur at the same ticks

### Requirement: Generic controller surface

The system SHALL expose a controller interface that can open, close, emergency stop, set the level of, repair and read the state of any ride by its instance id, and SHALL expose it over HTTP as `GET /rides`, `GET /rides/{id}`, `POST /rides/{id}/open`, `/close`, `/emergency-stop`, `/level` and `/repair`, where the id is the park placement serial. An unknown id SHALL return 404; a refusal SHALL return 409 with its reason code; malformed input 400. These routes SHALL be separate from the existing `/ride/*` routes.

#### Scenario: Unknown ride

- **WHEN** `GET /rides/999` is requested and no such ride exists
- **THEN** the response is 404

#### Scenario: Refusal

- **WHEN** `POST /rides/3/repair` is sent for a ride that is not Broken
- **THEN** the response is 409 with `NotBroken`

#### Scenario: List

- **WHEN** `GET /rides` is requested with a carousel and an octopus placed
- **THEN** both are listed with id, definition id, status, state, ratings and reliability

### Requirement: The digital-twin Octopus is one adapter

The `octopus` entry SHALL use behaviour `digital-twin`: its lifecycle, physics, loading and telemetry remain those of the DigitalTwin module and its `/ride/*` routes and dashboard SHALL behave exactly as before whether or not the Octopus is placed. Placing it SHALL attach the ride's scene root to the placement node; removing it SHALL detach it and park it at the world origin, and removal SHALL be refused as busy unless the lifecycle state is Idle. The controller SHALL read state, occupancy and safety through the ride adapter port, forward open, close and emergency stop as status changes and lifecycle requests, and take rating measurements from the adapter. Automatic dispatch of the twin SHALL be off by default and configurable.

#### Scenario: Dashboard unchanged

- **WHEN** the Octopus is not placed in any park
- **THEN** the existing `/ride/*` routes and the dashboard work as before

#### Scenario: Placement attaches the rig

- **WHEN** the Octopus is placed at anchor `(20, 20)` with rotation 0
- **THEN** the ride's scene root is a child of the placement node at world position `(50.0, 2.0, 50.0)`

#### Scenario: Removal while running

- **WHEN** the Octopus is Started and its removal is requested
- **THEN** it is refused with `ObjectBusy`

#### Scenario: One Octopus

- **WHEN** an Octopus exists and a second is placed
- **THEN** the placement is refused with `MaxPerParkReached`

#### Scenario: Measured ratings

- **WHEN** the Octopus completes a run
- **THEN** its ratings are recomputed from the adapter's measured maxima and marked measured
