# Spec Delta

## Purpose

Defines the arrival process that creates guests at the park entrance from park rating, entrance fee, weather, capacity, awards and marketing, and the migration from the `Queue` module's arrival planner. It is the only source of new guests in park mode.

## ADDED Requirements

### Requirement: Guest mode selects who creates guests

The configuration `Simulation:Guests:Mode` SHALL accept `LegacyQueue` (default) and `Park`. In `LegacyQueue` the simulation SHALL create no guests and the `Queue` module behaves exactly as before. In `Park` the simulation SHALL own arrivals and the `Queue` background filler SHALL be disabled.

#### Scenario: Default mode leaves the queue module untouched

- **WHEN** the host starts with no `Simulation:Guests:Mode` value
- **THEN** the simulation creates no guests and the Queue filler fills queues as in the published `weather-driven-queue-fill` behaviour

#### Scenario: Park mode disables the filler

- **WHEN** the host starts with `Simulation:Guests:Mode=Park`
- **THEN** the Queue filler enqueues nothing and guests appear only through the simulation's arrival process

#### Scenario: Unknown mode value fails at start-up

- **WHEN** the host starts with `Simulation:Guests:Mode=Banana`
- **THEN** options validation fails with a message naming the key and the accepted values

### Requirement: Arrival rate is the product of independent factors

The expected arrival rate in guests per second SHALL be `MaxSpawnRate` (default 3.0) multiplied by a rating factor, a fee factor, a weather factor, a capacity factor, an award factor and added marketing demand, each defined below, and SHALL never exceed 60 guests per second.

#### Scenario: Neutral park

- **WHEN** rating is 999, entrance fee does not exceed value for money, `NiceWeather` is 1, guests in park are below suggested capacity, no award and no campaign
- **THEN** the expected arrival rate is 3.0 guests per second

#### Scenario: Factors multiply

- **WHEN** the rating factor is 0.5, the weather factor 0.5 and all other factors 1
- **THEN** the expected arrival rate is 0.75 guests per second

### Requirement: Rating factor is linear between a floor rating and the maximum

The rating factor SHALL be 0.02 at rating 200 or below and 1.0 at rating 999, rising linearly in between.

#### Scenario: Rating bounds

- **WHEN** the rating is 200, 599.5 and 999
- **THEN** the rating factor is 0.02, 0.51 and 1.0 respectively (tolerance 0.005)

#### Scenario: Rating below the floor

- **WHEN** the rating is 0
- **THEN** the rating factor is 0.02

### Requirement: An entrance fee above value for money suppresses arrivals

Let V be the park's value for money in cents (supplied by the park metrics) and F the entrance fee in cents. The fee factor SHALL be 1.0 when F is at most V, otherwise `(V / F)` squared, never below 0.05. When V is zero and F is positive the factor SHALL be 0.05.

#### Scenario: Fee within value

- **WHEN** V is 1 200 and F is 1 000
- **THEN** the fee factor is 1.0

#### Scenario: Fee double the value

- **WHEN** V is 500 and F is 1 000
- **THEN** the fee factor is 0.25

#### Scenario: Free entry

- **WHEN** F is 0
- **THEN** the fee factor is 1.0

### Requirement: Weather scales arrivals with the existing nice-weather mapping

The weather factor SHALL be `Ceiling * NiceWeather^Exponent` clamped to [0, Ceiling], with `Ceiling` default 1.0 and `Exponent` default 1.0 (same shape and defaults as the Queue module), and exactly 0 when `NiceWeather` is 0. `NiceWeather` SHALL be read from the weather condition provider once per simulated second and held until the next read; before the first successful read the neutral default 1.0 SHALL apply.

#### Scenario: Worst weather stops arrivals

- **WHEN** `NiceWeather` is 0 for 60 simulated seconds
- **THEN** no guest arrives during those 60 seconds

#### Scenario: Monotonic in nice weather

- **WHEN** the arrival rate is evaluated at `NiceWeather` 0.2, 0.5 and 0.9 with all else equal
- **THEN** the three rates are strictly increasing

#### Scenario: Weather provider unavailable

- **WHEN** the weather provider throws on a read
- **THEN** the previous value stays in force, the failure is counted in a metric and the tick continues

### Requirement: Crowding beyond suggested capacity suppresses arrivals

The park's suggested guest capacity C SHALL be the sum of the capacity contribution of every open, working ride and facility. The capacity factor SHALL be 1.0 while guests in the park G are at most C and otherwise `max(0.1, C / G)`.

#### Scenario: Under capacity

- **WHEN** C is 800 and G is 790
- **THEN** the capacity factor is 1.0

#### Scenario: Over capacity

- **WHEN** C is 400 and G is 800
- **THEN** the capacity factor is 0.5

#### Scenario: No attractions

- **WHEN** C is 0 and G is 10
- **THEN** the capacity factor is 0.1

### Requirement: Awards and campaigns adjust arrivals through singleton inputs

An award modifier in permille (range -250 to +250) SHALL multiply the rate by `1 + modifier / 1000`. Marketing demand in guests per second (non-negative) SHALL be added after multiplication and before the 60 per second cap. Both SHALL be read from singleton inputs that default to 0 when no economy is present.

#### Scenario: Positive award

- **WHEN** the base rate is 2.0 and the award modifier is +250
- **THEN** the rate is 2.5

#### Scenario: Campaign demand is additive

- **WHEN** the multiplied rate is 1.0 and marketing demand is 0.4
- **THEN** the rate is 1.4

### Requirement: Arrivals are drawn per tick from a seeded stream

Each tick the spawner SHALL compare a draw from the counter-based stream `(world seed, spawner stream id, tick)` against `rate / (3 * 120)` (scaled to 32 bits; 3 is the mean party size) and, on success, create one party. Party size SHALL be drawn uniformly from 1 to 5. The party SHALL appear at an entrance node chosen by a second draw. Each member SHALL pay the entrance fee from its wallet on arrival; a member whose cash is below the fee SHALL be turned away without entering and counted.

#### Scenario: Long-run rate matches the expectation

- **WHEN** the rate is held at 1.2 guests per second for 3 600 simulated seconds with seed 42
- **THEN** the number of guests created (including turned away) is within 9 % of the expected 4 320 (three standard deviations of the party process) and is identical on every run with the same seed

#### Scenario: Turned away at the gate

- **WHEN** a guest arrives with cash 700 cents and the fee is 1 000 cents
- **THEN** the guest is not created as a park resident, the turned-away counter increases by one and no income is recorded

#### Scenario: Fee is paid on entry

- **WHEN** a guest with cash 4 000 cents enters and the fee is 1 000 cents
- **THEN** the guest's cash is 3 000 cents and an entrance-income record of 1 000 cents is produced for the tick

### Requirement: Migration from the queue arrival planner is staged and reversible

The change SHALL provide, in this order: the mode switch (default `LegacyQueue`), the filler switch in the Queue module, the arrival process, the `IRideQueueService` adapter, and a documented rollback (set the mode back to `LegacyQueue`). The `weather-updated` subscription of the Queue module and the `group-queued` event SHALL keep working in both modes.

#### Scenario: Weather subscription in park mode

- **WHEN** a `weather-updated` event is published while the mode is `Park`
- **THEN** the Queue module's subscriber still acknowledges it and its stored nice-weather value updates

#### Scenario: Group-queued is published in park mode

- **WHEN** a party of three joins a ride queue in park mode
- **THEN** exactly one `group-queued` integration event is published with the ride id, a group id and people count 3

#### Scenario: Rollback

- **WHEN** the mode is changed from `Park` back to `LegacyQueue` and the host restarts
- **THEN** the simulation creates no guests and the Queue filler resumes as before
