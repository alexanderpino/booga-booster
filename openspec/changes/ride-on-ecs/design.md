## Context

Today the ride is a DDD aggregate tree: `Ride` owns the lifecycle state machine and `GreatMill`, which owns four `Hub`s, which own `Gondola`s, which own `Seat`s (`src/DigitalTwin/FourDotnet.BoogaBooster.DigitalTwin/Domain/`). One hosted service, `RideSimulationService`, calls `IRideStore.Advance(1/120 s)` on a `PeriodicTimer`; command handlers call the same `IRideStore`; `RideTelemetryStream` samples it at 30 Hz for the SSE endpoint. `IRideStore` is therefore already the seam between the application and the simulation, and every method returns a `RideTelemetry` snapshot of the resulting state. (`Application/IRideStore.cs`, `Application/RideSimulationService.cs`.)

The legacy step order is fixed by `Ride.Advance` and `GreatMill.AdvancePhysics` and must be reproduced exactly for bit-identical results:

1. Passenger behaviour by lifecycle state (`Loading`/`Safe`: restraints secure; `Offloading`: passengers leave).
2. Automatic demotion `Safe -> Loading` if no longer safe.
3. While `Started`, `Stopping` or `EmergencyStop`: mill drive torque (signed by direction) then integrate the mill; then for each hub: drive torque, integrate; then for each gondola of that hub: sense G-forces from the new mill and hub angles and the **old** gondola yaw, then update the gondola pendulum (semi-implicit Euler).
4. Stopping or emergency-stopping ride at complete rest: engage gondola brakes, release restraints, enter `Offloading`.
5. `Offloading` with an empty ride becomes `Idle`.
6. Simulated time advances by `dt.TotalSeconds`.

`octopus-motion` (change 1, applied before this change) adds the cam-driven elevation of the four main arms, the hub position `(L cos θ, L sin θ)`, the vertical component of gondola acceleration and G-force, and the reach-dependent inertia; it updates the same legacy path. This design treats its pure functions (intent: a cam-profile function, kinematics helpers, new `RideParameters` constants) as the shared source of truth the ECS systems call. Exact names are reconciled at apply time from the archived `octopus-motion` change.

`ecs-foundation` provides the Simulation module (entities, structure-of-arrays components, queries, a system abstraction, deferred structural changes). `scene-graph` provides parent-indexed local and world transforms and their propagation system. `job-system` provides `IJobScheduler`, `JobGraphBuilder` with declared read and write sets, `InlineJobScheduler`, `IJobTimelineProvider` and the shared metrics. This change consumes all three through the `Simulation.Abstractions` project only, as the modular-monolith rule requires.

Motivation and scope: see `proposal.md`.

## Goals / Non-Goals

**Goals:**
- Run the Octopus on the World with systems scheduled by the job system, with no change visible to the HTTP surface, the SSE contract, the lifecycle rules or the 20 published specs.
- Prove equivalence mechanically (golden master, contract tests, shadow mode) before the default flips.
- Keep every commit building green and keep the legacy engine as the default until the proof is complete.
- Make the new architecture visible: a documented component and system table, a declared-access graph, and a dashboard view of system and job timings.
- Measure honestly, including the case where the new engine is not faster.

**Non-Goals:**
- No new ride behaviour and no change to any physical constant.
- No removal of the legacy aggregates in this change (the removal plan is a document and a follow-up change).
- No change to `Queue`, `Weather` or `Controller`.
- No change to the SSE payload, the Angular dashboard, or the Three.js view.
- No live switching of engines on a running ride, no state transfer between engines.
- No multi-ride product feature: spawning several rides into one world exists for tests and benchmarks only.

## Decisions

### D1. The ECS ride lives inside the DigitalTwin module and uses Simulation.Abstractions

New code goes in `DigitalTwin/FourDotnet.BoogaBooster.DigitalTwin/Ecs/` with subfolders `Components/`, `Systems/`, `Acl/` (the `EcsRideStore`, telemetry mapper, sampler replay, shadow store) and `Observability/` (timings collector). Ride-specific components and systems stay with the ride's bounded context rather than in `World`, which stays generic. This requires `Simulation.Abstractions` to offer a way to register component types and systems and to create a world (intent: `IWorldBuilder` / `IWorldFactory`, `ISystem`, the scheduler contracts). If `ecs-foundation` only exposes that in the implementation project, the apply agent stops and raises it (see Open questions); it does not add a reference to the module project. The style guide's module anatomy lists `Domain/Application/Features/Endpoints/`; an extra `Ecs/` folder is an adapter layer and the style-guide consultation in task 1.3 confirms or renames it.

### D2. The anti-corruption layer is an `IRideStore` implementation

`EcsRideStore : IRideStore` (and `IRideTelemetryProvider`) translates each store method into world operations and returns `RideTelemetry` built from components. Because every consumer already depends on `IRideStore`, handlers, endpoints, `RideSimulationService`, `RideLoadingCoordinator` and `RideTelemetryStream` need no change. Engine selection is a DI decision in `AddDigitalTwinModule()`: `Legacy` registers `RideStore`, `Ecs` registers `EcsRideStore`, `Shadow` registers `ShadowRideStore` (D12). The legacy `RideStore` is not edited except for the behaviour-preserving extractions in D3.

*Alternative rejected:* a new port in front of the aggregates only (`IRidePhysics`), keeping `Ride` as the owner of lifecycle and boarding with ECS used only for the physics step. It is a smaller change but leaves two sources of truth for state (aggregate and components), needs a copy-in/copy-out every tick, and does not demonstrate the migration the roadmap asks for.

### D3. Share pure code; never duplicate physics

Bit-identical results come from calling the same functions in the same order, not from re-deriving them. Behaviour-preserving extractions, each its own commit with the existing tests as proof:

- **Lifecycle policy**: the transition table, guards, safety evaluation and automatic-transition rules move out of `Ride` into a pure `RideLifecyclePolicy` working on a small `RideSafetyFacts` value (counts of occupied and unsecured seats, brake state, rest flag, load balance flags, and so on). `Ride` calls it; the ECS lifecycle systems build the facts from components and call the same policy. This is what makes "state machine unchanged" true by construction.
- **Loads and inertia helpers**: summation of gondola, passenger and hub loads in the legacy fixed order, as static functions over spans.
- **Already pure and shared as they are**: `RotationalDynamics`, `RideKinematics`, `RideParameters`, `MotorDirectionExtensions`, the octopus-motion cam profile. `internal` visibility is enough because `Ecs/` is in the same assembly.

The ECS systems are thin loops over component arrays that call these functions. The Gondola's `CentreOfMass`, `UpdateGForces` and pendulum update are likewise extracted into pure static functions on a struct of inputs. After the extraction, `Gondola.AdvancePhysics` calls them, so any later correction is made once.

### D4. Components

Plain `struct` components with property-style data per the repo's no-public-fields rule where the foundation's storage allows it (intent: `ecs-foundation` fixes the component declaration form; if it requires fields, the rule is applied through the foundation's own convention and flagged in the PR). Separate component types per body class, because the job system orders systems by conflicting component **types**, not by entities (job-system D6): if the mill, hub and gondola angle all lived in one `Spin` type, three systems that could overlap would be serialised.

| Component | Key fields | On entity |
|-----------|-----------|-----------|
| `RideIdentity` | `RideIndex` | ride |
| `RideLifecycle` | `State`, `SimulationTimeSeconds`, `EnginesBraked` | ride |
| `MillSpin` | `Angle`, `Omega` | ride (the mill spindle) |
| `HubSpin` | `Angle`, `Omega` | hub |
| `GondolaSpin` | `Angle` (yaw), `Omega` | gondola |
| `MotorDrive` | `PowerFraction`, `DirectionSign`, `Kind` (`Mill` or `Hub`) | ride, hub |
| `DriveTorque` | `Torque`, `Coulomb` (derived each tick) | ride, hub |
| `ArmElevation` | `ArmIndex`, `PhaseRadians`, `ElevationRadians`, `RateRadiansPerSecond` | main arm |
| `HubPlacement` | `RadialReach`, `Height`, `VerticalVelocity`, `VerticalAcceleration`, `TiltRadians` (always 0) | hub |
| `GondolaState` | `HubIndex`, `Index`, `Brake` | gondola |
| `GForce` | `Lateral`, `Forward`, `Vertical` | gondola |
| `Seat` | `Position`, `Restraint`, `CloseTimerSeconds`, `MeasuredKg` | seat |
| `PassengerRef` | `HasPassenger`, `PassengerId`, `WeightKg` | seat |
| `LoadSummary` | per-hub and ride loads, imbalance | ride, hub |
| `Link` | `Parent` entity, `Ordinal` | arm, hub, gondola, seat |
| `LocalTransform` (scene-graph) | rotation, translation | all of the above except links |
| `TelemetryFrame` | pre-allocated arrays for the projection | ride |

`PassengerRef` is always present on all 32 seats with a `HasPassenger` flag, so no structural change (add or remove component) happens inside a tick. `LoadSummary` is recomputed only where seats change (command application, pre-physics lifecycle), in the legacy summation order. Task 4.1 enumerates every legacy field that physics or telemetry reads and completes this table in `docs/simulation/ride-on-ecs.md`; that table is the reviewable deliverable.

### D5. Systems, declared access and graph

The tick graph has ten systems, registered in this order, which is the reference (serial) semantics. Order mirrors the legacy step order in Context, so a serial run is the legacy tick:

| # | System | Reads | Writes | Parallelism inside |
|---|--------|-------|--------|--------------------|
| 1 | `PrePhysicsLifecycleSystem` | `RideLifecycle`, `Seat`, `PassengerRef`, `GondolaState`, `MillSpin`, `HubSpin`, `LoadSummary` | `RideLifecycle`, `Seat`, `PassengerRef`, `LoadSummary` | none (32 seats) |
| 2 | `MotorTorqueSystem` | `MotorDrive`, `MillSpin`, `HubSpin`, `RideLifecycle`, `LoadSummary` | `DriveTorque` | none (5 motors) |
| 3 | `RotationalDynamicsSystem` | `DriveTorque`, `LoadSummary`, `HubPlacement` | `MillSpin`, `HubSpin` | hubs (4) |
| 4 | `CamElevationSystem` | `MillSpin`, `ArmElevation` (phase) | `ArmElevation` | arms (4) |
| 5 | `ArmKinematicsSystem` | `ArmElevation`, `MillSpin` | `HubPlacement` | hubs (4) |
| 6 | `GForceSystem` | `MillSpin`, `HubSpin`, `GondolaSpin`, `GondolaState`, `Seat`, `PassengerRef`, `HubPlacement` | `GForce` | gondolas (16) |
| 7 | `GondolaPendulumSystem` | `MillSpin`, `HubSpin`, `GondolaState`, `Seat`, `PassengerRef`, `HubPlacement` | `GondolaSpin` | gondolas (16) |
| 8 | `TransformWriteSystem` | `MillSpin`, `HubSpin`, `GondolaSpin`, `ArmElevation`, `HubPlacement` | `LocalTransform` | none |
| 9 | `PostPhysicsLifecycleSystem` | `MillSpin`, `HubSpin`, `Seat`, `PassengerRef` | `RideLifecycle`, `GondolaState`, `Seat` | none |
| 10 | `TelemetryProjectionSystem` | all of the above | `TelemetryFrame` | none |

Operator commands run through an eleventh, separate `CommandApplicationSystem` in a one-system **command graph** executed on the same scheduler (D6). It writes `MotorDrive`, `Seat`, `PassengerRef`, `GondolaState`, `RideLifecycle`, `LoadSummary`.

Notes that matter for correctness:
- G-force is sensed **before** the pendulum update, as in the legacy `Gondola.AdvancePhysics`, because the legacy engine computes G from the old yaw. This is not a modelling statement, it is the parity requirement; the spec scenario "Dependent systems are ordered" pins it. If `octopus-motion` changed that order, the step-order table produced in task 2.1 wins and the spec scenario is updated through a delta in this change before its tasks are implemented.
- Legacy interleaves hub 0's gondolas before hub 1's integration; the ECS runs all hubs, then all gondolas. This is equal because no hub integration reads a gondola's yaw (only loads, not orientation) and no gondola reads another hub. The golden master verifies the claim (spec `ride-engine-equivalence`); the table is not trusted on its own.
- Honest parallelism analysis: the data dependencies form a nearly linear chain (mill, elevation, kinematics, G-force, pendulum, projection). The graph overlaps little; the parallel speed-up for one ride comes from the parallel-for inside systems 3 to 7 (4 hubs, 16 gondolas), which is small work. The real parallel win appears when many rides share a world (D10). The graph is still the right abstraction: later guest and economy systems have broad independent fan-out.

### D6. Commands, concurrency and consistent reads

A single gate (a lock) serialises "tick" and "command" on the world, mirroring the existing `RideStore` lock. A command takes the gate, runs the command graph (the application system plus a forced telemetry projection), builds the `RideTelemetry` from the frame and returns it. A tick takes the gate, runs the tick graph, publishes the new frame. Reads (`GetTelemetry`, the SSE sampler) never take the gate: the projection writes a complete `TelemetryFrame` into one of two buffers and publishes it with an atomic reference swap, so a reader gets the last complete frame and the 30 Hz stream cannot block the 120 Hz tick (the `ride-telemetry-streaming` rule that sampling "SHALL NOT drive or block the physics loop").

The projection runs at the telemetry cadence (every 4th tick) and on command. `RideTelemetry` (the immutable DTO, with its allocations) is built from the published frame on demand and cached per frame version, so at most one DTO is built per frame, no matter how many readers there are, and the tick itself allocates nothing.

Commands reuse the same validation as legacy: argument checks and domain rules that throw `DomainValidationException` live in the shared policy or in small static validators extracted alongside D3, so the error type and message are identical. Domain rules that need the aggregate's state (for example "seat is occupied") are expressed as validators over components.

`RideSimulationService.Tick` calls `IRideStore.Advance` synchronously on a hosted-service thread; the ECS store blocks that thread while the scheduler runs the frame. That ties up one ThreadPool thread for under a millisecond 120 times a second, which is acceptable for a single ride; a dedicated thread is noted in Risks.

### D7. Telemetry mapping

A `RideTelemetryMapper` builds `RideTelemetry`, `MillTelemetry`, `HubTelemetry`, `GondolaTelemetry`, `SeatTelemetry` from a `TelemetryFrame`, including every field `octopus-motion` added. It is the only place that knows the wire shape. A unit test serialises one legacy and one ECS snapshot for the same state and compares property names, order and value kinds, which is the `Same frame shape` scenario.

### D8. Feature flag semantics

`DigitalTwinModuleOptions` gains a nested `Simulation` section with `Engine` (`RideEngineKind`: `Legacy`, `Ecs`, `Shadow`) and `ShadowCompareEveryTicks` (default 4, range 1 to 120). Validation through `IValidateOptions` at start-up; unknown values fail the host with a message naming the setting and the valid values. The default is `Legacy` in every commit until task 14.1, which is a single, separate commit that flips the default to `Ecs` after the equivalence suite and the shadow soak are green. Rollback after the flip is a configuration change to `Legacy`; no data migration exists because the ride is in-memory and ephemeral.

While the ECS engine is incomplete, selecting `Ecs` or `Shadow` fails at start-up with "not yet supported" so a half-built engine can never be reached by configuration. The guard is removed in the commit that makes the contract test suite pass on the ECS engine (task 7.4).

### D9. Strangler-fig sequence

The order of work follows the strangler pattern: first make the legacy path observable and seamed, then grow the new path next to it behind the flag, then prove equivalence, then flip the default, then (in a later change) remove the old path. The twelve phases below map onto task groups 2 to 14 in `tasks.md`; every task group ends in a commit that builds and passes the full DigitalTwin test project:

1. Characterise and seam the legacy path (tests and behaviour-preserving extractions only; default and behaviour unchanged).
2. Engine option and contract-test harness (still only `Legacy` selectable).
3. ECS world skeleton and ACL, rest-state telemetry only (guarded).
4. Lifecycle, commands, boarding in ECS (guarded).
5. Physics systems one by one, each with an equivalence check against legacy for exactly the fields it computes.
6. Telemetry projection, mapper, guard removed after the contract suite passes on `Ecs`.
7. Job-graph scheduling of the systems and worker-count independence.
8. Full golden master.
9. Shadow mode.
10. Timings endpoint, then the frontend view.
11. Performance comparison and default job configuration.
12. Default flip, documentation, removal plan.

Every intermediate state is releasable because the default is `Legacy` and the unfinished engine is unreachable.

### D10. Golden-master equivalence

The suite lives in `src/Tests/FourDotnet.BoogaBooster.DigitalTwin.Tests/Equivalence/`:

- A `RideScenario` is a seeded list of `(tick, command)` pairs; a runner executes it against any `IRideStore` factory and yields one `RideTelemetry` per tick (the store returns it from `Advance`). Scenarios: S1 board 32 passengers, start, 80% mill and 60% hub power for 60 s, stop to offload; S2 reverse the mill at 20 s; S3 emergency stop at 15 s; S4 engine brake at 25 s and gondola-brake toggles; S5 three loaded hubs with heavy passengers (imbalance flags); S6 rejected commands. A `ScriptedSampler` supplies fixed weights and delays, so no real randomness enters.
- A `TelemetryComparer` walks both snapshots by reflection over the public properties of the telemetry records (so a new field cannot be missed), applies the tier table, and returns the first difference with scenario, tick, simulated time, path (for example `Gondolas[7].LateralG`), both values with their bit patterns and the tier.
- **Tiers.** Tier A (bit-identical via `BitConverter.DoubleToInt64Bits`): simulation time, mill and hub angle and speed, power, arm elevations and each gondola's angle, speed and G-forces, because the ECS calls the same functions in the same order. Tier B (`abs(a - b) <= 1e-9 * max(1, abs(a), abs(b))`): loads, imbalance, consumed power, any value built by a reduction. Tier C (exact): enumerations, booleans, integers, strings, list lengths. Default for an unlisted `double` is Tier B. The tolerance of 1e-9 relative sits about seven orders of magnitude above double-precision rounding (about 2.2e-16 per operation), which is generous for sums of tens of terms over 10,800 steps, and about six orders below the smallest physical effect the dashboard shows; a larger gap is a bug, not rounding. Tier B values are promoted to Tier A, with a recorded reason in `docs/simulation/ride-on-ecs-equivalence.md`, wherever the golden run shows them bit-identical and the ECS summation order is provably the legacy order; the aim is an empty Tier B list.
- **Self-tests.** Legacy against a second legacy instance passes everything; changing the ECS mill Coulomb friction by 1 percent fails S1 within 10 simulated seconds. Without these a comparer that compares nothing would be green.
- **Worker-count independence.** The ECS output of every scenario is compared across worker counts 1, 2, 4, processor count and `Inline` (bit-identical), and a 16-ride world is hashed with 1 and 4 workers.
- Floating-point on different machines: the comparison is between two engines in one process on one machine, so it is immune to cross-machine JIT differences. No floating-point values are checked into the repository.

### D11. Random draws in Shadow and in tests

The legacy `RideStore` pulls from `IRideEventSampler` (weights, restraint-close delays) when boarding. In Shadow, a `RecordingSampler` wraps the real sampler and logs each draw; a `ReplaySampler` gives the ECS engine the same values in the same order and throws if it asks for a draw that was not recorded. In the equivalence tests both engines use the same `ScriptedSampler` instead.

### D12. Shadow mode

`ShadowRideStore : IRideStore` forwards every call to the legacy store first, returns its result, and then repeats the call on the ECS store inside a try/catch. A call the legacy store rejects is not repeated. ECS exceptions are logged (deduplicated per message per minute) and counted on `boogabooster.ride.engine.shadow.errors`. Every `ShadowCompareEveryTicks` ticks it compares snapshots with the equivalence comparer and counts differences on `boogabooster.ride.engine.divergence`, tagged `group` with the bounded set `time`, `lifecycle`, `mill`, `hub`, `gondola`, `seat`, `other`. The first divergence per group is logged with the path and values. Shadow mode costs the sum of both engines per tick and is for validation environments, not for production.

### D13. Engine timings: collector, endpoint, view

- **Collector.** `EngineTimingsCollector` (singleton) is fed by `EcsRideStore.Advance`: the tick duration from `Stopwatch` and the per-system spans from `IJobTimelineProvider.GetLatest()` of the frame that just ran. It keeps fixed 120-element rolling windows (ring arrays, no per-tick allocation) for the tick and each system and computes average, median, p99, max and the over-budget count on request from a copy of the window, so statistics are exact for the window.
- **Endpoint.** `GET /ride/engine/timings` is a new route in the `DigitalTwinEndpoints` `/ride` group, dispatching `GetEngineTimingsQuery` to a handler (`ExecuteAsync`), returning an `EngineTimingsResponse` DTO in `DigitalTwin.Abstractions/DataTransferObjects/EngineTimings/` (use the `dto-organization` skill) that embeds the `JobFrameTimeline` DTO from the job system. It is a separate endpoint on purpose: the SSE `RideTelemetry` contract stays byte-for-byte unchanged. Under `Legacy` it returns `available: false`.
- **View.** A lazy-loaded route `/engine` renders `EngineTimingsPageComponent` (standalone, zoneless, signals, `OnPush`). The first real PrimeNG use in the app: `Card`, `Tag` (engine and over-budget status with text and icon), `Table` (systems), `ProgressBar` or `MeterGroup` (budget; the component is chosen by consulting the `primeng` MCP server first), `Message` (empty state). The worker lanes have no PrimeNG equivalent (PrimeNG's `Timeline` is an event list, not a Gantt chart), so they are a small SVG component with `role="img"`, an accessible name, and an adjacent table of per-worker busy time and job count; that table is the text equivalent required by the accessibility spec. Data comes from a service that polls the endpoint once per second with an `interval`-driven signal that stops on destroy, following the existing queue and weather polling pattern. Text colour never carries meaning alone (over-budget has a label and an icon). Vitest specs plus an `*.a11y.spec.ts` with axe-core. PrimeNG needs a theme provider (`providePrimeNG` with a `@primeuix/themes` preset) in `app.config.ts`; the task verifies that adding it does not alter the existing hand-rolled panels (existing specs and the a11y suite are the proof) and scopes styles via the CSS layer option if it does.

### D14. Performance comparison and acceptance

`src/Tests/FourDotnet.BoogaBooster.DigitalTwin.Benchmarks` (BenchmarkDotNet, a candidate dependency subject to the currency check) measures tick time for: legacy; ECS in `Inline` mode; ECS with 1, 2, 4 workers; and a scale-out world of 1, 16, 64 and 256 rides. It also measures allocation per tick and the cost of building a `RideTelemetry` from the frame. As in job-system, hardware-dependent limits are relative to a baseline measured on the same machine first, and they are targets to calibrate after that baseline, not facts verified today. Results go to `docs/simulation/ride-on-ecs-performance.md`.

| # | Measure | Acceptance | Why |
|---|---------|-----------|-----|
| P1 | One ride, ECS `Inline`, median tick | At most 2.0 times legacy, and at most 100 microseconds | 21 rotating bodies is tiny work; indirection through component arrays is expected to cost something. 100 microseconds is 1.2 percent of the 8.33 ms budget, so even 2x legacy is invisible to the loop |
| P2 | One ride, default job configuration, p99 tick | At most 1.0 ms | The hard requirement is the deadline: 12 percent of the budget leaves room for GC pauses and OS jitter on a `PeriodicTimer` loop |
| P3 | Allocation per steady-state tick (ECS, projection included, DTO excluded) | 0 bytes, and no more than legacy | Allocation is deterministic and checkable in xUnit; telemetry DTO building is on demand |
| P4 | `RideTelemetry` build from a frame | At most 1.5 times legacy `ToTelemetry` | It runs at 30 Hz, not 120 Hz, so a small penalty is fine |
| P5 | 64 rides in one world, 4 workers versus 1 worker | Speed-up at least 2.4 times, identical state hashes | Same Amdahl derivation as job-system B6 (about 15 percent serial fraction gives a 2.76 bound at 4 workers) |
| P6 | 64 rides, 4 workers, tick time | At most 4 ms (half the budget) | Shows headroom for the park-scale changes that follow |

An expected, honest outcome: for a single ride the parallel mode may be slower than `Inline` because scheduling overhead exceeds 21 bodies of work. The acceptance rule is therefore a decision rule, not a hope: task 13.3 picks the default job mode and worker count with the lowest p99 for one ride, records it, and sets it in `Simulation:Jobs` defaults for the ride's world. Parallelism is demonstrated by P5 and P6 and by the shared world used by later changes. If P1 or P2 fail, profile and fix (typical causes: boxing, per-tick LINQ, lock contention) before relaxing a number, and record any relaxation with the measured value in the performance doc and the PR description.

### D15. Removal plan for the legacy path

Documented now in `docs/simulation/ride-on-ecs-removal-plan.md`; executed by a follow-up change `remove-legacy-ride-aggregates`, not here.

Gates before removal: (1) the equivalence suite is green in CI on every run since the default flip; (2) at least one full release, and at least 7 days in a shared environment in `Shadow` mode with the divergence counter at 0 before the flip; (3) the default has been `Ecs` for one release with `Legacy` unused in any deployed configuration; (4) the performance comparison meets P1 and P2.

Steps: delete `RideStore`, `Ride`, `GreatMill`, `Hub`, `Gondola`, `Seat` and their aggregate tests (domain-level tests on the aggregates are ported to ECS system tests first); delete `ShadowRideStore`, `RecordingSampler`/`ReplaySampler` and the divergence metrics; remove the `Engine` flag and the Legacy and Shadow branches in DI; keep and relocate the shared pure code (`RideLifecyclePolicy`, `RotationalDynamics`, `RideKinematics`, `RideParameters`, extracted gondola functions, the cam profile) and the contract and golden-master tests re-based on stored scenario digests or removed with the engine they compared. The doc lists each file and the order, so the follow-up change can be proposed without rediscovery.

## Dependencies and contracts

**Requires from earlier changes (intent; reconcile names at apply time):**
- From `octopus-motion`: the cam-profile function `f(mainAngle, armIndex)` and its parameters in `RideParameters`; hub position, vertical acceleration and G-force formulas as pure functions; updated `MillTelemetry`/`GondolaTelemetry` fields; updated legacy step order if it changed.
- From `ecs-foundation`: `Simulation.Abstractions` entry points to create a world, register component types and systems, query components, and read/write through a per-system context; module registration conventions; `World.Tests`.
- From `scene-graph`: `LocalTransform`, parent links, and a propagation system that can be added to the graph.
- From `job-system`: `IJobScheduler` (threaded and inline), `JobGraphBuilder` with `SystemAccess`, `JobGraph.Run`, `IJobTimelineProvider` with `GetLatest()`/`GetRecent`, the `Simulation:Jobs` options, and the metrics on the shared meter.

**Provides to later changes (intent):**
- `FourDotnet.BoogaBooster.DigitalTwin.Abstractions`: `EngineTimingsResponse` and related DTOs; nothing else public changes.
- `DigitalTwin`: `RideWorldFactory.Spawn(world, rideIndex)` (spawn one Octopus into any world; used by tests, benchmarks and potentially the park world), the component set and system set as a reusable template, the equivalence harness (`RideScenario`, `TelemetryComparer`) as the pattern for later strangler migrations, `DigitalTwin:Simulation:*` options, metrics `boogabooster.ride.engine.divergence` and `boogabooster.ride.engine.shadow.errors`, and `GET /ride/engine/timings`.
- Frontend: the `/engine` route, `EngineTimingsService` and the first PrimeNG theme provider, reusable by `park-economy` and `coaster-editor` dashboards.

## Risks / Trade-offs

- **Hidden numeric drift** (a different operation order gives different low-order bits) → Mitigation: shared pure functions, Tier A on the principal state, a comparer that reports the first differing tick and field, and a mutation self-test. A drift found means fixing the order, not widening the tolerance.
- **Reductions parallelised by accident** → The legacy sums are sequential in a fixed order. Mitigation: ECS sums (loads, imbalance) are sequential loops in the same order inside single systems; parallel-for is used only for per-entity maps; Tier B exists as a documented safety net, and the target is an empty Tier B list.
- **Declared sets wrong or too coarse** → Per-type conflict detection serialises more than strictly needed, and a wrong declaration races. Mitigation: access checking on in all tests; separate component types per body class (D4); the thread-count independence tests catch races.
- **Octopus-motion not yet written when this is written** → All its interfaces are stated as intent. Mitigation: task 1.4 reads the archived change and writes a reconciliation table; a changed step order updates the spec through a delta before implementation.
- **A half-finished engine reachable by configuration** → Mitigation: start-up guard (D8) removed only when the contract suite passes on `Ecs`.
- **Shadow cost and sampler coupling** → Shadow doubles CPU and relies on the recorded-sampler discipline. Mitigation: validation environments only; replay throws on an unrecorded draw so desync is loud.
- **Parallel mode slower for one ride** → Expected (D14). Mitigation: decision rule and Inline default if measurement says so; the architecture value is shown by the scale-out benchmarks.
- **Blocking a ThreadPool thread 120 times per second** → Acceptable at under 1 ms for one ride. Mitigation: measure in P2; if it hurts under load, run the tick loop on a dedicated thread (a follow-up, not designed here).
- **PrimeNG theme provider may restyle the app** → Mitigation: existing Vitest and a11y suites as the check, CSS layer scoping if needed (D13).
- **Third-party candidates** → BenchmarkDotNet and PrimeNG components are used as candidates: at apply time each must pass the currency check (not deprecated in the pinned version, official docs name no different default, upstream not archived and released within 12 months); PrimeNG is already pinned by the repo, so only (a) and (b) apply for it, using the `primeng` MCP server. No version is asserted here because none was verified.
- **Coverage drop** → New ECS code must hold the 80 percent module floor; the group verifies it explicitly.

## Migration Plan

Additive until the last step. Phase 1 (this change, every commit): `Legacy` is the default, nothing observable changes, the ECS path is unreachable or opt-in for tests. Phase 2 (task 14.1, its own commit): the default becomes `Ecs` once the golden master, contract suite, the shadow soak and P1 to P2 are green. Rollback at any time: set `DigitalTwin:Simulation:Engine` to `Legacy` and restart; no persisted state exists. Phase 3 (a later change, plan in D15): remove the legacy path after the gates.

## Open Questions

- Does `ecs-foundation` expose component and system registration through `Simulation.Abstractions`? If not, the apply agent must stop and ask rather than reference the module project. (Would change task structure, so confirm before apply.)
- Should the engine timings view eventually live under the main dashboard as a collapsible panel? Deferred: a separate route avoids changing the `ride-dashboard-layout` spec, and moving it later is a frontend-only change.
- Should `Shadow` support a sampling mode that compares every tick for a short window after a command? Deferred: the default of every fourth tick is enough for a soak.
