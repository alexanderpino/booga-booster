# Spec Delta

## Purpose

Defines the park rating: the simplified formula that turns guest and ride metrics into one number from 0 to 999, how it is smoothed over time, the bands it is reported in, and the breakdown exposed so players and students can see why it moved. The rating drives how many guests arrive.

## ADDED Requirements

### Requirement: The rating is the sum of nine integer terms

The target rating SHALL be the sum of nine terms (listed in the first scenario) clamped to [0, 999]. All divisions SHALL be integer divisions truncating toward zero on non-negative operands, with shares expressed in permille.

#### Scenario: Term definitions

- **WHEN** the terms are evaluated
- **THEN** base is 99
- **AND** happy guests is 350 times the happy share
- **AND** ride uptime is 150 times the uptime share
- **AND** ride quality is 200 times (1 - distance) as defined in the quality requirement
- **AND** variety is 150 times min(8, open rides) / 8
- **AND** crowd is 50 times min(2 000, guests) / 2 000
- **AND** litter is minus 200 times min(100, old litter) / 100
- **AND** lost guests is minus min(150, 5 times max(0, lost - 10))
- **AND** queues is minus 100 times min(420, max(0, average queue seconds - 180)) / 420

#### Scenario: Perfect park

- **WHEN** happy share is 1 000 permille, uptime 1 000, quality distance 0, 8 or more open rides, 2 000 or more guests, no litter, no lost guests and no long queues
- **THEN** the target is 999

#### Scenario: Typical park

- **WHEN** 500 guests, happy share 700 permille, uptime 950, average excitement 620 and intensity 520 (hundredths), 6 open rides, 30 old litter, 12 lost guests, average queue 240 s
- **THEN** terms are 99, 245, 142, 192, 112, 12, -60, -10, -14 and the target is 718

#### Scenario: Floor

- **WHEN** every positive term is 0 except the base and penalties exceed 99
- **THEN** the target is 0

### Requirement: Ride quality is the distance from an ideal mix

Quality distance in permille SHALL be `(|E - 600| * 1 000 / 600 + |I - 500| * 1 000 / 500) / 2` capped at 1 000, where E and I are the average excitement and intensity in hundredths over open, rated rides; the term SHALL be 200 times (1 000 - distance) / 1 000. With no open rated rides the term SHALL be 0.

#### Scenario: Distance example

- **WHEN** E is 620 and I is 520
- **THEN** the distance is 36 permille and the term is 192

#### Scenario: Extreme rides

- **WHEN** E is 1 400 and I is 1 300
- **THEN** the distance is capped at 1 000 and the term is 0

#### Scenario: No rides

- **WHEN** there is no open rated ride
- **THEN** the term is 0

### Requirement: Small samples use a neutral happy share

When fewer than 10 guests are in the park, the happy share SHALL be taken as 500 permille so a few guests cannot swing the rating.

#### Scenario: Eight guests

- **WHEN** 8 guests are present, all happy
- **THEN** the happy term is 175

### Requirement: The rating moves toward the target in bounded steps

Every 12 simulated seconds the rating SHALL move toward the target by at most 25 points: `new = old + clamp(target - old, -25, +25)`. The difficulty offset SHALL be added to the target before this step (then clamped to [0, 999]). The first computation after a new scenario starts from the scenario's initial rating (default 500).

#### Scenario: Rising

- **WHEN** the rating is 500 and the target is 718
- **THEN** the rating after one step is 525 and after nine steps 718

#### Scenario: Falling

- **WHEN** the rating is 600 and the target is 590
- **THEN** the rating after one step is 590

#### Scenario: Offset

- **WHEN** difficulty offset is -50 and the formula gives 718
- **THEN** the target used is 668

### Requirement: Lost guests are counted over the last minute

The lost-guest input SHALL be the number of guests that left as lost during the last five rating periods (60 simulated seconds), computed from the difference of the simulation's cumulative lost counter.

#### Scenario: Window

- **WHEN** the cumulative lost counter reads 0, 4, 9, 9, 15, 20, 20 at successive periods
- **THEN** the input at the last period is 20 - 4 = 16 (the sum over the last five periods)

### Requirement: Bands and breakdown are reported

The rating SHALL be reported with a band: Poor below 300, Fair 300 to 549, Good 550 to 799, Excellent 800 and above, and a breakdown listing every term with its input and value, the target and the current rating.

#### Scenario: Band edges

- **WHEN** ratings 299, 300, 549, 550, 799 and 800 are classified
- **THEN** the bands are Poor, Fair, Fair, Good, Good and Excellent

#### Scenario: Breakdown sums

- **WHEN** the breakdown is read
- **THEN** the terms sum to the target before clamping and offset

### Requirement: The rating is written to the simulation

Each new rating SHALL be sent to the simulation as one logged command, applied at the next tick boundary, only when the value changed.

#### Scenario: Unchanged

- **WHEN** the step leaves the rating unchanged
- **THEN** no command is sent
