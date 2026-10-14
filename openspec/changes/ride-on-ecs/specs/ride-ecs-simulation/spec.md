## ADDED Requirements

### Requirement: The Octopus is built from entities and components

When the ride runs on the ECS engine, its world SHALL contain one ride entity, four main-arm entities, four hub entities, sixteen gondola entities and thirty-two seat entities. Physical and operational state SHALL live in plain-data components: a transform, angular state for the mill, for each hub and for each gondola, arm elevation, motor drive, brake, gondola, seat, and a passenger reference. A passenger reference SHALL hold only an identifier and a weight, never a domain object.

#### Scenario: Entity counts at start-up

- **WHEN** the ECS ride is created
- **THEN** the world holds 1 ride, 4 main arms, 4 hubs, 16 gondolas and 32 seats
- **AND** each gondola belongs to exactly one hub and each seat to exactly one gondola

#### Scenario: Seats start empty

- **WHEN** the ECS ride is created
- **THEN** all 32 seats have no passenger reference

#### Scenario: Passenger reference carries only identity and weight

- **WHEN** a passenger of 82.5 kg is boarded
- **THEN** the seat's passenger reference holds the passenger's identifier and 82.5

#### Scenario: Several rides can share a world

- **WHEN** the ride factory spawns 3 rides into one world
- **THEN** each ride's systems act only on its own entities and the three rides evolve independently

### Requirement: Systems declare access and run as a job graph

Every system SHALL declare the component types it reads and writes. The ride's tick SHALL be a graph of ten systems in registration order: pre-physics lifecycle, motor torque, rotational dynamics, cam elevation, arm kinematics, G-force, gondola pendulum, transform write, post-physics lifecycle, telemetry projection. Operator commands SHALL run in a command-application system in its own graph. The job system SHALL derive the edges from the declarations and run one frame per fixed 1/120 s step.

#### Scenario: Lifecycle is split around the physics

- **WHEN** the graph is described
- **THEN** the pre-physics lifecycle system covers passenger behaviour and automatic demotion
- **AND** the post-physics lifecycle system covers rest detection, offloading and simulated time

#### Scenario: Graph description lists every system

- **WHEN** the ride's graph is described
- **THEN** it lists the ten tick systems with their read and write sets
- **AND** every system appears exactly once

#### Scenario: Dependent systems are ordered

- **WHEN** the graph is described
- **THEN** cam elevation precedes arm kinematics, arm kinematics precedes G-force, and G-force precedes gondola pendulum, because the legacy engine senses G-forces from the gondola angle before the pendulum step updates it

#### Scenario: Commands run through the same machinery

- **WHEN** an operator command is applied
- **THEN** it is executed by the command-application system through the job scheduler, and the call returns after that system has finished

#### Scenario: Undeclared access is caught in tests

- **WHEN** the ride runs with access checking on for a 10-second scenario
- **THEN** no system reads or writes a component outside its declaration

### Requirement: One tick equals one fixed step

Each call to advance the ride SHALL advance simulated time by exactly the fixed step of 1/120 s, SHALL run the graph once, and SHALL complete before the call returns. Simulated time SHALL be accumulated in the same way as the legacy engine so that it matches it exactly.

#### Scenario: Time after 120 ticks

- **WHEN** the ride is advanced 120 times
- **THEN** the simulation time equals the legacy engine's simulation time after 120 ticks, bit for bit

#### Scenario: Tick is synchronous

- **WHEN** an advance call returns
- **THEN** every system of that frame has finished

### Requirement: Lifecycle behaviour is unchanged

The ECS engine SHALL enforce the same lifecycle states, the same guarded operator transitions, the same automatic transitions, the same emergency-stop rule and the same safety evaluation as the `ride-state-machine` capability, by using the same pure lifecycle policy as the legacy engine.

#### Scenario: Same available transitions

- **WHEN** the ride is in each of the seven lifecycle states in turn
- **THEN** the ECS engine reports the same available transitions as the legacy engine for the same facts

#### Scenario: Automatic demotion

- **WHEN** a restraint opens while the ride is in the `Safe` state
- **THEN** the ECS engine moves the ride back to `Loading` on the next tick

#### Scenario: Stopping settles into offloading

- **WHEN** a stopping ride reaches complete rest
- **THEN** the gondola brakes engage and the ride enters `Offloading`

### Requirement: Commands apply immediately and are validated as before

An operator command SHALL be applied to the ECS world before the call returns and SHALL return a telemetry snapshot of the resulting state. A command that the legacy engine rejects with a domain validation error SHALL be rejected with the same error type and the same message. Commands and ticks SHALL never run at the same time on the same world.

#### Scenario: Power command is reflected at once

- **WHEN** the operator sets the mill power to 60% on a started ride
- **THEN** the returned snapshot shows the mill power at 60%

#### Scenario: Illegal boarding rejected identically

- **WHEN** a passenger is boarded into an occupied seat
- **THEN** the ECS engine throws the same domain validation error with the same message as the legacy engine

#### Scenario: Command during a tick

- **WHEN** a command arrives while a tick is running
- **THEN** it is applied after that tick finishes and before the next one starts

### Requirement: Arm elevation follows the cam profile

The cam elevation system SHALL compute the elevation of each of the four main arms as a pure function of the main rotation angle, the arm index and the cam parameters defined by `octopus-motion`, using that change's cam-profile function. It SHALL NOT add a motor or an independent axis for elevation.

#### Scenario: Elevation equals the profile

- **WHEN** the main angle is set to a chosen value and one tick runs
- **THEN** each arm's stored elevation equals the cam-profile function's value for that angle and arm, bit for bit

#### Scenario: Elevation does not integrate a rate

- **WHEN** the ride is stopped at rest
- **THEN** the arm elevations stay constant

#### Scenario: Phases are staggered

- **WHEN** the main angle varies over one full revolution
- **THEN** the four arms do not all reach their highest point at the same angle

### Requirement: The hub keeps a vertical axis through the elevating arm

For an arm with elevation angle θ above the horizontal and length L = 6 m, the hub position SHALL be radial reach L·cos θ and height L·sin θ, and the hub's plane SHALL stay horizontal.

#### Scenario: Arm horizontal

- **WHEN** an arm's elevation is 0 degrees
- **THEN** the hub's radial reach is 6.000 m and its height is 0.000 m

#### Scenario: Arm raised 30 degrees

- **WHEN** an arm's elevation is 30 degrees
- **THEN** the hub's radial reach is 5.196 m and its height is 3.000 m, each within 0.001 m

#### Scenario: Hub plane stays horizontal

- **WHEN** an arm's elevation is any value in the permitted range
- **THEN** the hub's tilt about the horizontal axes is 0

### Requirement: Rotational dynamics, gondola motion and G-force match the physics docs

The mill and hub dynamics SHALL use the same integrator, constants and drive/brake model as the legacy engine, and the gondola pendulum and G-force calculations SHALL use the same formulas as the legacy engine including the vertical component and reach-dependent inertia added by `octopus-motion`. No constant SHALL be duplicated: the ECS engine SHALL read them from the shared ride parameters.

#### Scenario: Spin-up from rest

- **WHEN** the mill is commanded to a power of 80% from rest on both engines and 10 s pass
- **THEN** the mill speed and angle of the ECS engine are bit-identical to the legacy engine's

#### Scenario: Reversal passes through rest

- **WHEN** the mill direction is reversed while spinning
- **THEN** the ECS mill speed decreases through zero and then becomes negative, as in the `motor-direction-control` capability

#### Scenario: Engine brake

- **WHEN** the engine brake is engaged on a spinning ride
- **THEN** the ECS mill comes to rest on the same tick as the legacy engine's mill

#### Scenario: Constants are shared

- **WHEN** a constant in the shared ride parameters is changed in a test build
- **THEN** both engines change behaviour in the same way

### Requirement: Telemetry is projected from components without blocking the tick

A projection system SHALL write the telemetry quantities into a pre-allocated frame at the telemetry cadence (every fourth tick, 30 Hz), and the engine SHALL build the `RideTelemetry` snapshot from the last completed frame on demand. A read of telemetry SHALL NOT block the tick and SHALL NOT observe a partly written frame. A command SHALL force a fresh projection so its returned snapshot reflects the command.

#### Scenario: Read during a tick

- **WHEN** a reader requests telemetry while a tick is running
- **THEN** it receives the last completed frame, never a mix of two ticks

#### Scenario: Projection cadence

- **WHEN** 120 ticks run
- **THEN** the projection system runs 30 times for the cadence-driven frames

#### Scenario: Projection allocates nothing

- **WHEN** a warmed-up projection runs 1,000 times
- **THEN** it allocates 0 bytes on the managed heap

### Requirement: Transforms are written for the scene graph

The transform-write system SHALL write each entity's local transform from its angular state and elevation, so that the scene-graph propagation yields world transforms for the mill, arms, hubs, gondolas and seats. These transforms SHALL NOT feed back into the physics.

#### Scenario: Gondola world position

- **WHEN** the mill angle is 0, the arm elevation is 0 and the hub angle is 0
- **THEN** the world position of the first gondola on hub 0 equals the legacy pivot position (hub centre plus the 2 m hub arm), within 1e-9 m

#### Scenario: Transforms do not change physics

- **WHEN** the transform-write system is disabled in a test
- **THEN** the telemetry of a 30 s scenario is unchanged
