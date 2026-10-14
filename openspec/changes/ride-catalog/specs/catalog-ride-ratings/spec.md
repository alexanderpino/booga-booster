# Spec Delta

## Purpose

Defines how a ride's excitement, intensity and nausea are obtained (from data plus an operating level, or from measured simulation), how too much intensity reduces excitement, how a ride value for pricing is derived, and what the catalog publishes about each ride. All numbers are original tuning values.

## ADDED Requirements

### Requirement: Rating units and integer arithmetic

Ratings SHALL be integers in hundredths (350 means 3.50). Every rating computation SHALL use integer arithmetic with truncating division of non-negative values, so the same inputs give the same outputs on every machine. Measured G inputs SHALL be rounded to integer hundredths of g (round half to even) before use. A computed rating SHALL be clamped to 0 through 2000.

#### Scenario: Rounding of G inputs

- **WHEN** a measured peak total G is 1.905 g
- **THEN** the input used is 190 hundredths (half to even)

#### Scenario: Clamp

- **WHEN** a computed intensity would be 2350
- **THEN** the published intensity is 2000

### Requirement: Fixed-base model with an operating level

Under the fixed-base model the published ratings SHALL be `base + (level - 1) * step` per rating, where `level` is the ride's current operating level from 1 to `levels.count`. A ride with one level, or a zero step, SHALL publish its base ratings at every level.

#### Scenario: Swing at level 3

- **WHEN** `corsair-swing` (base 170/230/160, step 25/40/30) runs at level 3
- **THEN** its raw ratings are 220, 310 and 220

#### Scenario: Level change recomputes at once

- **WHEN** the operator changes `whirlwind-twirl` from level 2 to level 3
- **THEN** its published ratings become base 140/160/210 plus 2 steps of 30/25/40, which is 200, 210 and 290

### Requirement: Simulation-derived model

Under the simulation-derived model the ratings SHALL be computed once per completed run (at the transition from Started to Stopping) from the maxima measured by the run: `gPeak` (largest total G over all seats and ticks) and the vertical swing `vSwing` (largest vertical G minus smallest vertical G), both in hundredths of g after rounding. With `excess = max(0, gPeak - 100)` and `v = min(vSwing, 150)`, the raw ratings SHALL be: excitement = `E0 + excess * 60 / 100 + v * 40 / 100`; intensity = `I0 + excess * 150 / 100 + v * 30 / 100`; nausea = `N0 + v * 120 / 100 + excess * 20 / 100`, where `E0`, `I0`, `N0` are the definition's base values. Before the first completed run the ratings SHALL be the definition's provisional baseline and be marked provisional.

#### Scenario: Reference run

- **WHEN** the octopus (base 250/220/180) completes a run with gPeak 190 and vSwing 100
- **THEN** its raw ratings are excitement 344, intensity 385 and nausea 318
- **AND** the ratings are marked measured

#### Scenario: Gentle run

- **WHEN** a run has gPeak 100 and vSwing 0
- **THEN** the raw ratings equal the base values 250, 220 and 180

#### Scenario: Swing is capped

- **WHEN** a run has vSwing 400
- **THEN** the computation uses v = 150

#### Scenario: Before any run

- **WHEN** an octopus has not completed a run
- **THEN** its ratings are the provisional baseline and are marked provisional

### Requirement: Intensity penalty on excitement

After the model, for each bound in 900, 1100 and 1300 that the raw intensity reaches or exceeds, the excitement SHALL be multiplied by 3/4 using truncating integer division. Intensity and nausea SHALL be unchanged by the penalty.

#### Scenario: Two bounds crossed

- **WHEN** raw ratings are excitement 400, intensity 1150 and nausea 300
- **THEN** the bounds 900 and 1100 are crossed, so excitement becomes 300 and then 225
- **AND** intensity stays 1150 and nausea stays 300

#### Scenario: Below all bounds

- **WHEN** raw intensity is 899
- **THEN** excitement is unchanged

#### Scenario: All three bounds

- **WHEN** raw ratings are excitement 400 and intensity 1300
- **THEN** the published excitement is 168

### Requirement: Delegated and absent models

A coaster SHALL use the delegated model: its published ratings are the provisional baseline until the `coaster-editor` change supplies measured values through the rating contract, and then they SHALL be marked measured. Stalls, facilities, path additions and staff buildings SHALL publish no ratings.

#### Scenario: Coaster baseline

- **WHEN** a `family-coaster` is placed
- **THEN** its ratings are 200, 180 and 100 marked provisional

#### Scenario: Stall has no ratings

- **WHEN** the stats of a `frost-cart` are requested
- **THEN** no ratings are present

### Requirement: Ride value

The ride value in cents SHALL be `(excitement * wE + intensity * wI + nausea * wN) / 100` with truncating division, using the penalised excitement and the definition's value weights. Age and duplicate-ride factors are out of scope and belong to `park-economy`.

#### Scenario: Octopus value

- **WHEN** ratings are 344, 385, 318 and the weights are 40, 20, 10
- **THEN** the value is 246 cents

#### Scenario: Zero ratings

- **WHEN** all ratings are 0
- **THEN** the value is 0

### Requirement: Derived throughput

For rides with timed cycles the catalog SHALL publish `throughputPerHourAtFullLoad = capacity * 3600 / cycleSeconds` with truncating division, where `cycleSeconds = loadSecondsMin + rideSeconds + rampDownSeconds + unloadSeconds`.

#### Scenario: Carousel throughput

- **WHEN** `starlight-carousel` is read (capacity 24, load min 15, ride 75, ramp down 6, unload 15)
- **THEN** its throughput at full load is 778 guests per hour

### Requirement: Published ride statistics

For every placed ride the system SHALL publish: excitement, intensity, nausea, rating source (fixed, provisional or measured), reliability (0 to 10000), ticket price (cents), value (cents), operating status, lifecycle state, operating level, queue length and queue time. Queue length and queue time SHALL be 0 until `guests-and-needs` supplies them. The ticket price SHALL default to the definition's `defaultPriceCents` and be settable only through the economy contract (a settable hook that this change exposes and `park-economy` uses).

#### Scenario: Statistics of a new carousel

- **WHEN** a carousel was just placed
- **THEN** it publishes ratings 105/25/30 at default level 2 (base 95/20/25 plus one step 10/5/5), source fixed, reliability 10000, price 200, status Closed, state Idle, queue length 0
