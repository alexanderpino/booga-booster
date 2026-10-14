## ADDED Requirements

### Requirement: Throughput meets a measured budget

The benchmark project SHALL measure these scenarios on one thread, Release build, with allocation diagnostics: (B1) integrate `position += velocity * dt` over N = 100 000 entities carrying `position` (3 doubles), `velocity` (3 doubles) and `mass` (1 double); (B2) the same over N = 10 000; (B3) create 100 000 entities with those 3 components through the command buffer, including playback; (B4) 10 000 destroys plus 10 000 creates through the command buffer in one playback; (B5) canonical hash of 100 000 entities; (B6) binary save of 100 000 entities. The budgets are: B1 mean at most 1.0 ms (aspiration 0.3 ms); B2 at most 0.1 ms; B3 at most 25 ms; B4 at most 3 ms; B5 at most 5 ms; B6 at most 15 ms; and 0 bytes allocated per operation for B1 and B2. Results SHALL be recorded, with CPU model and runtime version, in `docs/simulation/01-ecs-foundation.md`. A result worse than twice its budget SHALL stop the change for a storage-design review instead of being accepted.

#### Scenario: B1 within budget

- **WHEN** B1 is run on the reference machine
- **THEN** the mean is at most 1.0 ms and allocated bytes per operation is 0

#### Scenario: Over budget

- **WHEN** B1 measures 2.5 ms
- **THEN** the apply run records the result and stops with an open question instead of marking the benchmark task done

### Requirement: Allocation freedom is asserted by a unit test, timing is not

Zero-allocation steady-state iteration SHALL be asserted by an ordinary unit test (allocated bytes before and after), because it is machine-independent. Wall-clock budgets SHALL NOT be asserted in unit tests or CI gates.

#### Scenario: Unit test guard

- **WHEN** the unit-test suite runs on any machine
- **THEN** the allocation test passes and no test fails because a machine is slow
