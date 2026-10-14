# Spec Delta

## Purpose

Defines the guest's inner state: the five pressure needs, happiness with its target, nausea and tolerance classes, the thresholds that trigger behaviour, and how these evolve per simulated second. All values are on integer scales so results are exactly reproducible.

## ADDED Requirements

### Requirement: Needs and mood use fixed integer scales

Hunger, thirst, bladder, fatigue, nausea, happiness and happiness target SHALL be integers in [0, 10 000] (centi-points of a 0 to 100 scale). For hunger, thirst, bladder, fatigue and nausea a high value means high pressure; for happiness a high value means content. Every update SHALL clamp to the range. The scale SHALL be documented and exposed to clients as a percentage (value divided by 100).

#### Scenario: Clamping

- **WHEN** an update would raise hunger from 9 990 by 50
- **THEN** hunger is 10 000

#### Scenario: Percentage presentation

- **WHEN** happiness is 6 250
- **THEN** the inspection projection reports 62.5 percent

### Requirement: Needs are processed once per simulated second in cohorts

Each guest's needs SHALL be updated once every 120 ticks, in the tick where `(tick + entity index) mod 120` equals 0, so that about 1/120 of the guests update per tick. Each update SHALL apply one second of change. A guest created mid-run SHALL receive its first update at its own cohort tick, not immediately.

#### Scenario: Cohort spread

- **WHEN** 6 000 guests with consecutive entity indices are alive
- **THEN** in every tick between 49 and 51 guests are updated, and every guest is updated exactly once in any 120 consecutive ticks

### Requirement: Pressure needs grow at defined rates

Per update (one second) hunger SHALL rise by 8, thirst by 12, bladder by 5 and fatigue by `4 * fatigue scale / 100` (fatigue scale is a guest trait in [70, 140]), and nausea SHALL fall by 15 toward 0. The thirst rise SHALL be multiplied by a heat factor: 1.0 at or below 22 degrees Celsius, rising linearly to 2.0 at or above 30 degrees, using the latest weather temperature.

#### Scenario: Hunger from a typical start

- **WHEN** a guest starts with hunger 2 500 and eats nothing
- **THEN** hunger crosses the hungry threshold 7 000 after 563 updates (about 9.4 minutes)

#### Scenario: Heat doubles thirst growth

- **WHEN** the temperature is 30 degrees Celsius
- **THEN** thirst rises by 24 per update; at 26 degrees by 18

#### Scenario: Nausea recovers

- **WHEN** nausea is 5 000 and the guest does nothing nauseating
- **THEN** nausea is 0 after 334 updates

### Requirement: Thresholds trigger behaviour

The following thresholds SHALL be constants in the guest parameters: hungry 7 000, thirsty 7 000, needs toilet 7 500, tired 8 000, exhausted 9 000, sick 6 500, very sick 8 500. Crossing upward SHALL record the matching thought once per crossing (not once per update).

#### Scenario: One thought per crossing

- **WHEN** hunger rises from 6 992 to 7 000 and then continues to 8 000
- **THEN** exactly one "hungry" thought is recorded until hunger falls below 7 000 and crosses again

### Requirement: Happiness follows a target that events move

Each update SHALL move the happiness target toward the baseline 6 000 by up to 5, add event adjustments (unmet needs, ride outcomes, litter, queueing, shop outcomes) and clamp it. Visible happiness SHALL then move toward the target by up to 60. Unmet-need pressure SHALL subtract 10 per update from the target for each of hungry, thirsty, needs-toilet and tired, and 15 for sick. After a ride the visible happiness SHALL snap to the target.

#### Scenario: Slow convergence

- **WHEN** the target is 9 000, happiness is 6 000 and nothing else changes
- **THEN** happiness reaches 9 000 after 50 updates

#### Scenario: Hungry and thirsty guest drifts down

- **WHEN** a guest is hungry and thirsty, its target is exactly the baseline 6 000, and 10 updates pass with nothing else changing
- **THEN** the target is 5 845 (the first update subtracts 20 with no drift, each of the next nine subtracts 20 and adds 5)

#### Scenario: Snap after a ride

- **WHEN** a ride ends with target 8 200 and visible happiness 6 000
- **THEN** visible happiness is 8 200 immediately

### Requirement: Nausea tolerance classes scale nausea gain

Every guest SHALL have a nausea tolerance class (none, low, average, high). The nausea gained from a ride SHALL be `ride nausea rating (hundredths) * 10 * factor / 100` with factor 100, 60, 35 and 15 for none, low, average and high.

#### Scenario: Same ride, different tolerance

- **WHEN** a ride with nausea rating 500 (5.00) ends
- **THEN** nausea rises by 5 000 for none, 3 000 for low, 1 750 for average and 750 for high

### Requirement: Sick guests may vomit

While walking with nausea at or above 6 500, each update SHALL give the guest a chance of `(nausea - 6 500) * 300 / 3 500` permille to vomit, drawn from the guest's counter-based stream. Vomiting SHALL reduce nausea by 2 500, drop vomit litter and add a thought.

#### Scenario: Probability at maximum nausea

- **WHEN** nausea is 10 000
- **THEN** the vomit chance per update is 300 permille

#### Scenario: Below the threshold

- **WHEN** nausea is 6 499
- **THEN** the guest never vomits

### Requirement: Queue waiting erodes mood exponentially after the guest's tolerance

A queuing guest SHALL be unaffected by waiting until its queue tolerance (trait in [120, 420] s, default centre 300 s). Beyond it, each update SHALL subtract from the target `min(200, 8 * 2^((wait - tolerance) / 60))` using an integer lookup table, with no floating-point exponential in the simulation.

#### Scenario: Drain growth

- **WHEN** tolerance is 300 s and the guest has waited 300, 360 and 420 s
- **THEN** the per-update drain is 8, 16 and 32

#### Scenario: Drain cap

- **WHEN** the guest has waited 700 s past its tolerance of 300 s
- **THEN** the drain is 200

#### Scenario: No drain within tolerance

- **WHEN** the guest has waited 299 s
- **THEN** the queue term is 0
