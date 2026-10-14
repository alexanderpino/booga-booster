# Spec Delta

## Purpose

Defines the determinism guarantees of the guest simulation: counter-based seeded randomness, integer arithmetic, thread-count independence, golden masters and replay. They make guest bugs reproducible and let tests assert exact state.

## ADDED Requirements

### Requirement: All randomness is counter-based and seeded

Every random draw in guest systems SHALL come from a stateless counter-based generator keyed by `(world seed, stream id, entity or guest number, counter)`. Stream ids SHALL be unique constants per system. Systems SHALL NOT use `System.Random`, `Random.Shared`, `Guid.NewGuid`, wall-clock time or `Dictionary`/`HashSet` iteration order. The guest module SHALL contain no floating-point exponential or power function in simulation systems (tables are used).

#### Scenario: Same key same value

- **WHEN** the generator is called twice with the same key
- **THEN** the results are identical

#### Scenario: Different streams differ

- **WHEN** two stream ids are used with the same entity and counter
- **THEN** the results differ for at least 99.9 % of 100 000 sampled keys

#### Scenario: Forbidden APIs

- **WHEN** the architecture test scans the guest module's compiled code
- **THEN** it finds no reference to `System.Random`, `Random.Shared`, `Guid.NewGuid`, `DateTime.Now`, `DateTime.UtcNow`, `Math.Pow`, `Math.Exp`, `Interlocked` on floating-point types or `Parallel.*`

### Requirement: Guest state uses integers for everything that feeds decisions

Positions SHALL be 32-bit fixed point (16 fractional bits, metres), speeds fixed point per tick, needs and money integers, ratings integer hundredths and path distances integer millimetres. Floating point MAY be used only in projections for clients.

#### Scenario: Fixed-point position

- **WHEN** a guest of speed 1.4 m/s walks 120 ticks along a straight edge
- **THEN** its position advanced by exactly `120 * fixed(1.4 / 120)` raw units, identical on every machine

### Requirement: Results do not depend on the worker count

Running the same scenario with 1, 2, 4, 8 and 16 workers SHALL produce identical world hashes at every checkpoint. The checkpoint hash SHALL cover all guest, ride-queue, litter and metrics components.

#### Scenario: Thread-count independence

- **WHEN** the scenario "park, 600 guests, seed 42" runs 1 200 ticks (10 s) at each worker count and hashes the world after every 100th tick
- **THEN** all five hash sequences are equal

#### Scenario: Soak

- **WHEN** 100 runs execute with random worker counts
- **THEN** no hash differs from the reference

#### Scenario: Reductions are fixed-order

- **WHEN** the park metrics are computed at 1 and 16 workers
- **THEN** every metric is equal, because partial results are combined by chunk index

### Requirement: A golden master pins the behaviour

A committed golden file SHALL hold the hash sequence of the scenario "park, 600 guests, seed 42, 1 200 ticks, every 100th tick" and the final summary values (guests in park, arrivals, rides taken, purchases, litter count). The test SHALL fail on any difference; updating the file requires a deliberate commit that states why.

#### Scenario: Unchanged behaviour passes

- **WHEN** the golden test runs on an unmodified build
- **THEN** it passes

#### Scenario: Changed constant fails

- **WHEN** a guest parameter changes (for example the hunger rate)
- **THEN** the golden test fails and names the first checkpoint that differs

### Requirement: The seeded sampler is deterministic and replaceable

The guest trait sampler SHALL implement a small interface so tests can seed or fake it, mirroring the existing person-generator pattern, and the default SHALL be a pure function of seed and guest number (see guest entities).

#### Scenario: Fake sampler

- **WHEN** a test registers a sampler returning fixed traits
- **THEN** the run's results follow from those traits only

### Requirement: Runs can be replayed from an input log

All external inputs to the guest simulation (weather samples, price changes, ride status changes, graph edits, spawn mode and rate overrides) SHALL be applied at tick boundaries and written to an input log with their tick. Replaying the log with the same seed SHALL reproduce the final world hash.

#### Scenario: Replay equality

- **WHEN** a 5-minute session with three price changes and a weather change is recorded and replayed
- **THEN** the final hash equals the recorded one

#### Scenario: Weather sampled once per second

- **WHEN** the weather provider returns different values each call
- **THEN** exactly one value per simulated second is logged and used
