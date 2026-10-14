# Spec Delta

## Purpose

Defines the test run: a deterministic one-lap simulation of the player's design that produces its metrics and ratings, and, when the design fails, tells the player where and why so the next edit is an informed one.

## ADDED Requirements

### Requirement: A test run simulates one full lap with a fixed load (Tier 1)

A test run SHALL simulate a single train carrying every seat filled with 75 kg, starting at rest at the station stop point and dispatched at once, at 1/120 s steps until it stops at the platform again, using the same dynamics code as the live simulation. It SHALL run synchronously to completion and SHALL NOT change any park or simulation state. It SHALL be refused with 409 and the validation report when the design has an error finding.

#### Scenario: Starter completes

- **WHEN** a test run is requested for the starter coaster
- **THEN** the outcome is `Completed`, the lap duration is between 35 s and 60 s and the maximum speed is between 11.5 m/s and 14.9 m/s

#### Scenario: Energy bound

- **WHEN** the starter coaster completes a test run
- **THEN** the maximum speed does not exceed sqrt(2 x 9.81 x 11.0 + 2.5^2) = 14.9 m/s, because the highest point is 11.0 m above the station and the lift speed is 2.5 m/s

#### Scenario: Refused

- **WHEN** the design has an `OpenCircuit` error
- **THEN** the request is refused with 409 and no simulation steps run

### Requirement: The run ends with one of a fixed set of outcomes (Tier 1)

The outcome SHALL be exactly one of `Completed`, `StalledOnClimb`, `RolledBack`, `OvershotStation`, `TimedOut` and `Faulted`. The run SHALL end at the first stall, rollback, overshoot or fault, and after 600 s of simulated time with `TimedOut`. A stall is a speed magnitude below 0.05 m/s for at least 0.5 s outside the platform; a rollback is a speed below -0.05 m/s outside a lift zone.

#### Scenario: Timeout

- **WHEN** a train never returns to the platform within 600 s
- **THEN** the outcome is `TimedOut` and the simulated time is 600 s

#### Scenario: Order of detection

- **WHEN** a train rolls back and also stalls
- **THEN** the outcome is `RolledBack` because it is detected first

### Requirement: A stall is explained with its energy deficit (Tier 1)

For `StalledOnClimb` and `RolledBack` the result SHALL give the piece index and arc length where the train stopped, the car-averaged height there, the height of the next local maximum ahead, the height deficit (the difference, at least 0.05 m), the energy deficit m g x deficit in joules, and a suggestion naming the number of height steps by which to raise the preceding lift or lower the hill, rounded up. Following a stall the player SHALL see a marker on the track at the stop point.

#### Scenario: Hill taller than the first

- **WHEN** a hill with apex 13 m follows a first hill of apex 11 m with no lift on the second (fixture `starter-tall-hill`)
- **THEN** the outcome is `StalledOnClimb` on a rising piece of the second hill, the stop height is at most 11.32 m, and the height deficit is at least 1.0 m

#### Scenario: Suggestion rounds up

- **WHEN** the deficit is 1.38 m
- **THEN** the suggestion names 2 height steps

### Requirement: G limits are checked with warn and hard levels (Tier 1)

The run SHALL compare windowed G values to limits: vertical high warn 3.5 g and hard 4.5 g, vertical low warn -1.0 g and hard -2.0 g, lateral magnitude warn 1.8 g and hard 3.0 g. Each contiguous run above a limit SHALL be one event with its kind, severity, the piece index of its peak, the arc length, the peak value and the limit. A hard event SHALL make the verdict `Fail`; warn events alone SHALL give `PassWithWarnings`. The limits are configuration values with these defaults.

#### Scenario: Tight curves

- **WHEN** the fixture `starter-tight-curves` (four `curve:L2` instead of `curve:L4`) is run
- **THEN** it completes, has at least one lateral warn event on a `curve:L2` piece, and has no hard event, because the energy bound keeps the speed under 14.9 m/s and 14.9^2 / (8 x 9.81) = 2.83 g is below 3.0 g

#### Scenario: Merge of neighbouring samples

- **WHEN** the lateral G stays above 1.8 g for 200 consecutive steps
- **THEN** one event is reported, not 200

### Requirement: Overshooting the platform is a failure (Tier 1)

If the train passes the platform stop point with speed above 0 and the station brake cannot stop it, the outcome SHALL be `OvershotStation` with the arrival speed at the platform entry, the largest speed the station can stop (sqrt(2 x 2.0 x 11.0) = 6.63 m/s), and a suggestion to add trim brake pieces before the station.

#### Scenario: Too fast to stop

- **WHEN** a layout delivers the train to the platform entry at 8 m/s
- **THEN** the outcome is `OvershotStation` and the result states 8.0 m/s and 6.63 m/s

### Requirement: Only a completed run produces ratings (Tier 1)

The result SHALL contain metrics and ratings only for outcome `Completed`; other outcomes SHALL contain the metrics measured up to the end, flagged partial, and no ratings. The verdict SHALL be `Pass`, `PassWithWarnings` or `Fail`; a non-Completed outcome is `Fail`.

#### Scenario: Partial metrics

- **WHEN** a run ends in `StalledOnClimb`
- **THEN** the result has `partial = true`, a maximum speed, and no ratings

### Requirement: The trace feeds the client chart and replay (Tier 1)

The result SHALL carry a trace sampled every 4th step (30 Hz) with time, arc length, speed, vertical G, lateral G and longitudinal G, and a list of markers (kind, severity, piece index, arc length, text). If more than 6,000 samples would result, the sampling stride SHALL be increased by whole steps so that 6,000 or fewer are kept. The arc length series SHALL let the client replay the train and the camera along the same track.

#### Scenario: Starter trace

- **WHEN** the starter lap lasts 45 s
- **THEN** the trace has 1,351 samples within 2 and the time axis is strictly increasing

#### Scenario: Long run

- **WHEN** a run lasts 600 s
- **THEN** the trace has at most 6,000 samples and keeps the first and the last step

### Requirement: Results are deterministic and tied to a design hash (Tier 1)

For the same design hash, train configuration, limits and parameters the result SHALL be byte-identical when serialised canonically. The result SHALL carry the design hash and revision; a result whose hash differs from the design's current hash SHALL be reported as stale and SHALL NOT be used for ratings or opening. At most 2 test runs SHALL execute at the same time; further requests wait.

#### Scenario: Repeat

- **WHEN** the same design is tested twice
- **THEN** the two results serialise to identical bytes

#### Scenario: Stale result

- **WHEN** a piece is appended after a test run
- **THEN** the stored result is marked stale and `GET` of the test result for the new revision answers "no test run"

### Requirement: Diagnosis is shown to the player (Tier 1)

The result SHALL carry, for every failure and every G event, a message in plain language that names the piece by its position in the design and its kind (for example "Piece 17, curve left radius 2: lateral force 2.4 g, limit 1.8 g"), and a suggestion. The client SHALL show these as a list, as markers on the 3D track, and as shaded ranges on the G chart.

#### Scenario: Message names the piece

- **WHEN** a lateral warn event occurs on piece index 13
- **THEN** its message contains "Piece 14" (one-based for the player), the piece kind and the value with one decimal

### Requirement: Failure injection for the shipped fixtures (Tier 1)

The project SHALL ship the fixtures `starter-coaster` (Completed, Pass), `starter-tall-hill` (StalledOnClimb) and `starter-tight-curves` (Completed, PassWithWarnings), each closing its circuit, so workshops can demonstrate the diagnosis without drawing a track.

#### Scenario: Fixtures behave

- **WHEN** the three fixtures are tested
- **THEN** their outcomes and verdicts are Completed/Pass, StalledOnClimb/Fail and Completed/PassWithWarnings
