## ADDED Requirements

### Requirement: Simulation output is independent of worker count

For a simulation that follows the declared-access rules, the state after every frame SHALL be bit-identical for worker counts 1, 2, 4, 8 and the processor count, and for the `Inline` mode. This is the contract of ADR `docs/adr/game-0006-deterministic-parallel-simulation.md`.

#### Scenario: Golden master across worker counts

- **WHEN** the reference mini-simulation (4096 entities, 5 systems, seed 20260101) runs for 120 frames with worker counts 1, 2, 4, 8 and `Inline`
- **THEN** the per-frame state hash sequence is identical for all five runs
- **AND** it equals the checked-in golden hash sequence

#### Scenario: Differing worker count does not change simulation output

- **WHEN** worker count differs between two otherwise identical runs
- **THEN** every frame's state hash is identical

#### Scenario: Golden master detects a real change

- **WHEN** one system constant in the reference simulation is changed by one unit in the last place
- **THEN** the comparison against the golden file fails and names the first differing frame

### Requirement: Repeated runs are identical

The same input, seed and configuration SHALL give the same state hash sequence on every repetition on the same machine, regardless of thread timing.

#### Scenario: One thousand repetitions

- **WHEN** the reference mini-simulation's first 20 frames run 1,000 times with 4 workers
- **THEN** all 1,000 hash sequences are identical

### Requirement: Scheduling order is not observable in results

Perturbing the schedule SHALL NOT change results. A test hook SHALL be able to inject delays, shuffle steal-victim choice with different seeds, and vary the grain-independent timing, and the output hash SHALL stay identical.

#### Scenario: Randomised steal order

- **WHEN** the reference mini-simulation runs with 8 different steal-victim seeds on 4 workers
- **THEN** all 8 runs produce the same hash sequence

#### Scenario: Injected delays

- **WHEN** a random 0 to 200 microsecond delay is injected before 10% of jobs, chosen from a fixed seed
- **THEN** the hash sequence equals the undelayed one

### Requirement: Result-affecting state is never shared between jobs

The job system SHALL provide no API that lets two concurrently running jobs write the same output slot implicitly. Parallel results SHALL flow through per-chunk output slots combined in fixed order, or through component data that the graph's access declarations serialise.

#### Scenario: Concurrent writers are serialised by the graph

- **WHEN** two systems both declare a write to `Health`
- **THEN** they never run at the same time, in any worker configuration

#### Scenario: Reduce slots are per chunk

- **WHEN** a parallel-reduce runs with 100 chunks
- **THEN** it holds 100 independent partial slots, each written by exactly one chunk

### Requirement: Race and stress tests are bounded and repeatable

The test suite SHALL include stress tests of the deque, the counters and the scheduler whose iteration counts are fixed (not time-based), whose seeds are fixed, and whose failure message prints the seed and iteration so a failure can be replayed.

#### Scenario: Deque stress

- **WHEN** one owner pushes and pops while 3 thieves steal, over 1,000,000 items
- **THEN** every item is taken exactly once, none lost and none duplicated

#### Scenario: Failure is replayable

- **WHEN** a stress test fails
- **THEN** its message contains the seed and the iteration index
