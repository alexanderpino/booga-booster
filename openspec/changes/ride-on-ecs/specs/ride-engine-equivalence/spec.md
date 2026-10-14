## ADDED Requirements

### Requirement: Old and new engines are compared on seeded scenarios

A golden-master suite SHALL run scripted scenarios on the legacy and the ECS engine with the same seed, the same commands at the same ticks and the same sampler values, and SHALL compare the full `RideTelemetry` snapshot after every tick. The scenarios SHALL include at least: load and run to speed and stop; reverse the mill while spinning; emergency stop while running; engine brake and gondola brake toggles; an unbalanced load; and rejected commands.

#### Scenario: Every tick is compared

- **WHEN** a scenario of 90 simulated seconds runs
- **THEN** 10,800 snapshots are compared per scenario

#### Scenario: Same seed gives the same telemetry

- **WHEN** the run-to-speed scenario runs on both engines with seed 20260101
- **THEN** every snapshot passes the three-tier comparison

#### Scenario: Rejected commands compare equal

- **WHEN** a scenario issues a forbidden transition and a boarding into an occupied seat
- **THEN** both engines raise errors of the same type with the same message

### Requirement: Three tolerance tiers are defined

The comparison SHALL classify every telemetry field into one of three tiers. Tier A is bit-identical, for quantities whose floating-point order is preserved: simulation time, mill and hub angles and speeds, motor power, arm elevations, and each gondola's angle, speed and G-forces. Tier B is a relative tolerance for summed quantities (loads, imbalance, consumed power). Tier C is exact, for enumerations, booleans, integers, strings and list lengths. An unassigned numeric field SHALL default to Tier B.

#### Scenario: Tier B formula

- **WHEN** two Tier B values are compared
- **THEN** they pass when `abs(a - b) <= 1e-9 * max(1, abs(a), abs(b))`

#### Scenario: Tier A requires equal bit patterns

- **WHEN** two values differ by one unit in the last place
- **THEN** a Tier A field fails and a Tier B field passes

#### Scenario: Tier B tolerance edge

- **WHEN** two summed values of magnitude 1000 differ by 1.5e-6
- **THEN** the comparison fails because the allowed difference is 1e-6

#### Scenario: Tier C is exact

- **WHEN** the lifecycle states differ
- **THEN** the comparison fails with the field name and tick number

#### Scenario: New telemetry field cannot be forgotten

- **WHEN** a numeric property is added to the telemetry records without a tier assignment
- **THEN** it is compared as Tier B
- **AND** a test fails until the field is listed in the tier table or deliberately left on the default

### Requirement: A failing comparison is actionable

A failure SHALL report the scenario, the tick, the simulated time, the field path (for example `Gondolas[7].LateralG`), both values with their bit patterns, and the tier. The harness SHALL stop at the first failing tick.

#### Scenario: Failure message content

- **WHEN** gondola 7's lateral G differs at tick 4,321 of the run-to-speed scenario
- **THEN** the failure names the scenario, tick 4,321, `Gondolas[7].LateralG`, both values and the tier

### Requirement: The comparison harness detects real differences

The suite SHALL include a self-test that perturbs one physical constant in one engine and expects the comparison to fail, and a self-test that proves identical inputs pass, so the suite cannot pass vacuously.

#### Scenario: Perturbed constant

- **WHEN** the mill Coulomb friction is changed by 1% in the ECS engine only
- **THEN** the run-to-speed scenario fails at or before 10 simulated seconds

#### Scenario: Same engine on both sides

- **WHEN** the legacy engine is compared with a second instance of itself
- **THEN** all scenarios pass with zero differences

### Requirement: ECS output is independent of worker count

The ride's telemetry sequence on the ECS engine SHALL be bit-identical for job-system worker counts 1, 2, 4 and the processor count, and for the inline mode, for every scenario.

#### Scenario: Worker count does not change telemetry

- **WHEN** the run-to-speed scenario runs on the ECS engine with worker counts 1, 2, 4 and the processor count and in inline mode
- **THEN** all five snapshot sequences are bit-identical

#### Scenario: Scaled world stays independent

- **WHEN** a world with 16 rides runs for 10 simulated seconds with 1 worker and with 4 workers
- **THEN** every ride's state hash is identical in both runs

### Requirement: Existing behaviour tests hold on both engines

The existing store-level behaviour tests SHALL be run unchanged in meaning against both the legacy and the ECS engine through a shared contract. Domain-aggregate tests that call the aggregates directly SHALL remain in place for the legacy engine until it is removed.

#### Scenario: Contract suite runs twice

- **WHEN** the DigitalTwin test project runs
- **THEN** each contract test executes once for the legacy engine and once for the ECS engine

#### Scenario: Published specs still hold

- **WHEN** the contract suite for `ride-state-machine`, `ride-telemetry-streaming`, `engine-braking`, `gondola-boarding`, `ride-loading` and `motor-direction-control` scenarios runs on the ECS engine
- **THEN** every scenario passes without changes to those specs
