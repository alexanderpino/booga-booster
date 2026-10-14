## ADDED Requirements

### Requirement: The simulation engine is chosen by configuration

The DigitalTwin module SHALL read the setting `DigitalTwin:Simulation:Engine` with the values `Legacy`, `Ecs` and `Shadow` (case-insensitive) and SHALL run the ride on the chosen engine. The default SHALL be `Legacy` until the change's final cut-over step makes it `Ecs`. An unknown value SHALL stop the host at start-up with an error that names the setting and lists the valid values.

#### Scenario: Default engine

- **WHEN** the setting is absent and the module starts
- **THEN** the ride runs on the engine named by the documented default

#### Scenario: Explicit engine

- **WHEN** the setting is `Ecs`
- **THEN** commands and the fixed-step tick are served by the ECS-backed ride

#### Scenario: Case-insensitive value

- **WHEN** the setting is `ecs`
- **THEN** it selects the same engine as `Ecs`

#### Scenario: Unknown value

- **WHEN** the setting is `Turbo`
- **THEN** the host fails to start with an error naming `DigitalTwin:Simulation:Engine` and the values `Legacy`, `Ecs` and `Shadow`

### Requirement: Engine changes take effect only at start-up

The engine SHALL be fixed for the lifetime of the process. Changing the setting while the host runs SHALL NOT switch engines, and no ride state SHALL be transferred between engines.

#### Scenario: Setting changed at runtime

- **WHEN** the configuration value changes from `Legacy` to `Ecs` while the host is running
- **THEN** the running ride keeps using the engine it started with

### Requirement: External behaviour is identical in every mode

The HTTP endpoints, request and response shapes, status codes, the SSE telemetry stream and its 30 Hz cadence, the lifecycle state machine and the command semantics SHALL be the same whichever engine is selected. No endpoint SHALL be added to, removed from, or changed in the existing `/ride` route group by the engine choice.

#### Scenario: Same routes

- **WHEN** the route table is listed for each of the three engines
- **THEN** the sets of existing `/ride` routes and HTTP methods are identical

#### Scenario: Same rejection

- **WHEN** an operator requests a lifecycle transition that the guard forbids, on each engine
- **THEN** each engine responds with the same status code and the same error message

#### Scenario: Same frame shape

- **WHEN** one telemetry frame is serialised to JSON on each engine for the same ride state
- **THEN** the JSON has the same property names, order and value kinds

### Requirement: Shadow mode answers from the legacy engine and compares

In `Shadow` mode the legacy engine SHALL be authoritative: every response and every telemetry frame SHALL come from it. The ECS engine SHALL receive the same commands and ticks. Every `DigitalTwin:Simulation:ShadowCompareEveryTicks` ticks (default 4, valid 1 to 120) the module SHALL compare the two snapshots with the tolerance tiers of `ride-engine-equivalence` and SHALL increment a divergence counter, tagged with the field group, for each group that differs.

#### Scenario: Identical engines report no divergence

- **WHEN** a scripted scenario runs in `Shadow` mode and both engines agree
- **THEN** the divergence counter stays at 0

#### Scenario: Divergence is counted

- **WHEN** the ECS engine is deliberately made to return a different mill speed in a test
- **THEN** the divergence counter for the group `mill` increases
- **AND** the response to the operator still carries the legacy value

#### Scenario: Invalid compare interval

- **WHEN** `ShadowCompareEveryTicks` is 0 or 121
- **THEN** the host fails to start with an error naming the setting

### Requirement: Shadow mode isolates the ECS engine from the operator

An exception or fault inside the ECS engine in `Shadow` mode SHALL be logged and counted and SHALL NOT change any response, SHALL NOT stop the tick, and SHALL NOT stop the legacy engine. A command rejected by the legacy engine SHALL be rejected to the operator and SHALL NOT be applied to the ECS engine.

#### Scenario: ECS engine throws

- **WHEN** the ECS engine throws during a tick in `Shadow` mode
- **THEN** the legacy tick completes normally
- **AND** an error counter increases and an error is logged once per distinct message per minute

#### Scenario: Rejected command is not mirrored

- **WHEN** the legacy engine rejects a command with a domain validation error
- **THEN** the same error reaches the caller
- **AND** the ECS engine's state is unchanged

### Requirement: Random draws are shared between engines

In `Shadow` mode, and in the equivalence test harness, both engines SHALL see the same sequence of random values from the injected sampler. A value drawn for the legacy engine SHALL be replayed for the ECS engine; the ECS engine SHALL NOT draw from the real sampler itself.

#### Scenario: Same boarding delay

- **WHEN** a passenger is boarded in `Shadow` mode with an unspecified weight
- **THEN** both engines seat a passenger of the same weight
- **AND** both use the same restraint-close delay

#### Scenario: Sampler drained in order

- **WHEN** 10 passengers are boarded
- **THEN** the real sampler is called once per draw and the ECS engine consumes the recorded values in the same order
