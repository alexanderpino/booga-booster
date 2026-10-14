# Spec Delta

## Purpose

Defines the numbers a coaster is judged by: the lap over which they are measured, the felt-force (G) maths shared with the Octopus documents, how speed, drops, airtime, inversions and turns are counted, and how they map to excitement, intensity and nausea in the unit the ride catalogue uses. The mapping coefficients are this project's own tuning, not another game's.

## ADDED Requirements

### Requirement: Metrics are measured over one lap (Tier 1)

A lap SHALL start at the first step in which the train leaves the station platform with a positive speed and SHALL end at the step in which the train stops at the platform stop point. The lap duration SHALL be counted in whole steps of 1/120 s. The lap length SHALL be the track length. Average speed SHALL be lap length divided by lap duration. Maximum speed SHALL be the largest speed of the lead car in the lap.

#### Scenario: Starter lap length

- **WHEN** the starter coaster is compiled
- **THEN** its track length is 313.27 m within 0.01 m

#### Scenario: Average speed

- **WHEN** a lap of 313.27 m lasts 45.0 s
- **THEN** the average speed is 6.96 m/s within 0.01 m/s

### Requirement: Felt force is the specific force a - g (Tier 1)

The system SHALL compute the specific force f = a - g in the world frame, with a = a_t T + V^2 kappa (T the unit tangent, kappa the curvature vector, a_t the along-track acceleration of the step) and g = (0, -9.81, 0) m/s^2, and SHALL report longitudinal G = f.T / g, vertical G = f.U / g and lateral G = f.L / g with U the rider up vector and L the rider's right. Lateral G is positive toward the rider's right. Total G is |f| / g. This is the same definition as `docs/05-forces-and-g-forces.md` section 5.1, in a Y-up frame.

#### Scenario: At rest

- **WHEN** the train is at rest on flat track
- **THEN** vertical G is 1.000, lateral G and longitudinal G are 0.000, and total G is 1.000

#### Scenario: Flat curve at the matched speed

- **WHEN** a train runs on `curve:L4` (radius 16 m) at 12.528 m/s with constant speed
- **THEN** lateral G magnitude is 1.000 within 0.005, vertical G is 1.000 within 0.005, and total G is 1.414 within 0.01

#### Scenario: Free coasting has no longitudinal force

- **WHEN** a frictionless train coasts on `slope:-2`
- **THEN** longitudinal G is 0.000 within 1e-6

### Requirement: Vertical curves load and unload the rider (Tier 1)

On a pitch piece the vertical G SHALL follow from the vertical curvature: at the lowest point of `pitch:-2>+2` (vertical radius 8.0 m) a train at 8.859 m/s SHALL read vertical G 2.000, and at the apex of `pitch:+2>-2` at the same speed SHALL read 0.000.

#### Scenario: Valley bottom

- **WHEN** a train passes the vertex of `pitch:-2>+2` at 8.859 m/s
- **THEN** vertical G is 2.000 within 0.005

#### Scenario: Crest

- **WHEN** a train passes the vertex of `pitch:+2>-2` at 8.859 m/s
- **THEN** vertical G is 0.000 within 0.005

#### Scenario: Airtime speed

- **WHEN** a train passes that crest at 10.0 m/s
- **THEN** vertical G is -0.274 within 0.005

### Requirement: Reported peaks use a 0.1 s window (Tier 1)

Peak vertical, negative vertical and lateral G values SHALL be the extremes of the mean over a sliding window of 12 steps (0.1 s), so that a single-step spike at a join does not set the rating. The unwindowed values SHALL stay available in the trace. Jerk SHALL be reported as the largest absolute change of total G per second between 12-step windows.

#### Scenario: Single spike

- **WHEN** one step has a vertical G of 6.0 and its neighbours read 2.0
- **THEN** the windowed peak is 2.333 within 0.001 and the unwindowed peak is 6.0

### Requirement: Drops, airtime, inversions and turns are counted by fixed rules (Tier 1)

A drop SHALL be a fall of the lead car height of at least 3.0 m between a local maximum and the next local minimum, ignoring wiggles under 0.5 m; the highest drop is the largest such fall. Airtime is the total time with windowed vertical G below 0. An inversion is a run of the track where the rider up vector has a negative world-up component for at least 2.0 m of arc. Turns are the total absolute heading change divided by 90 degrees, rounded down.

#### Scenario: Starter drop

- **WHEN** the starter coaster completes a lap
- **THEN** it has 1 drop and the highest drop is 11.0 m within 0.1 m

#### Scenario: Short flip is not an inversion

- **WHEN** a compiled track has its up vector inverted for 1.0 m of arc
- **THEN** the inversion count is 0, and for 6.0 m of arc it is 1

#### Scenario: Turns

- **WHEN** the starter coaster is measured
- **THEN** it has 4 turns

### Requirement: Ratings are a saturating sum of metric terms (Tier 1)

Excitement, intensity and nausea SHALL each be a base value plus the sum of weight x x / (x + h) over the metric terms (maximum speed, average speed, highest drop, drop count, length, positive G above 1, negative G depth below 1, lateral G, airtime, inversions, turns), using the weights and half-saturation values of the design. Excitement SHALL be multiplied by min(1, duration / 30 s) and by 1 / (1 + 0.2 max(0, intensity - 8)). Results SHALL be reported as non-negative integers in hundredths (round half away from zero).

#### Scenario: Zero metrics

- **WHEN** all metrics are zero
- **THEN** the ratings are excitement 50, intensity 20, nausea 10

#### Scenario: Worked example

- **WHEN** maximum speed 14 m/s, average speed 8 m/s, highest drop 12 m, 2 drops, length 400 m, positive G peak 2.5, minimum vertical G 0.5, lateral G 1.5, airtime 0 s, 0 inversions, 6 turns and a duration of 60 s are rated
- **THEN** the ratings are excitement 367, intensity 405 and nausea 207

#### Scenario: Short ride

- **WHEN** the same metrics have a duration of 15 s
- **THEN** excitement is half of the value for 60 s (183 hundredths) and intensity and nausea are unchanged

#### Scenario: Too intense

- **WHEN** a ride has intensity 10.0 and unpenalised excitement 3.0
- **THEN** its excitement is 3.0 / 1.4 = 2.14

### Requirement: Ratings never decrease when a metric improves (Tier 1)

For each of the metric terms, increasing one metric with the others fixed SHALL NOT decrease excitement before the penalties, intensity or nausea. A property test SHALL check this on random metric vectors.

#### Scenario: Monotonic

- **WHEN** 1,000 seeded random metric vectors are each increased in one metric
- **THEN** none of the three raw ratings decreases

### Requirement: Stats are published in the ride catalogue unit and invalidated by edits (Tier 1)

After a Completed test run the system SHALL publish ride stats with excitement, intensity and nausea in hundredths, the metrics of this capability, the verdict, the design hash and the train configuration, through the ride catalogue port. A design with no Completed test run for its current hash SHALL be unrated: it SHALL publish no ratings, not zeros.

#### Scenario: Unrated

- **WHEN** a ride has never completed a test run
- **THEN** the published stats have no ratings and the ride cannot be opened

#### Scenario: Edit invalidates

- **WHEN** a piece is appended after a Completed test run
- **THEN** the stats for the old hash are no longer published

### Requirement: Throughput is estimated from the lap (Tier 2)

The system SHALL estimate throughput per hour as seats x 3600 / headway, with headway = max(dwell + 2.0 s, (lap duration + dwell) / trains), where dwell is the station dwell in seconds.

#### Scenario: One train

- **WHEN** 8 seats, lap 40 s, dwell 20 s and 1 train
- **THEN** headway is 60 s and throughput is 480 per hour

#### Scenario: Station limited

- **WHEN** 8 seats, lap 40 s, dwell 20 s and 4 trains
- **THEN** headway is 22 s and throughput is 1309 per hour within 1
