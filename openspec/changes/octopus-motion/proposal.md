## Why

The ride is a flat carousel: every arm turns about the vertical spindle in one horizontal plane, so the physics is planar (`PlanarVector`), every seat reads only horizontal G plus a constant 1 g, and the 3D view shows plain box arms that only yaw. The product is meant to be an *Octopus*-style ride whose four main arms rise and fall cyclically as the mill turns, which adds the vertical acceleration and the inertia swing that make the ride feel alive. This is also the first change of the game roadmap, so it must land on the current DDD architecture (no ECS yet) and leave clean contracts for the later migration.

## What Changes

- The four **main arms** gain an **elevation angle** `theta_k`, a cam-driven kinematic function of the mill angle: `theta_k = theta_park + lambda * (theta_cam(n * phi + k * Delta) - theta_park)`. Defaults (derived in a new `docs/06-octopus-arm-elevation.md`): base 15 deg, amplitude 15 deg (arms swing 0 to 30 deg), cam order `n = 1`, stagger `Delta = 180 deg` (opposite arms in unison, adjacent arms in antiphase). It is **not** a new motor or axis.
- An **engagement factor** `lambda` (smooth ramp, 6 s up / 8 s down) lets the arms rise when the ride is `Started` and be lowered to the park elevation after it has come to rest, so passengers always board and offload with the arms down. The ride only settles into `Offloading` when it is at rest **and** the arms are parked.
- The hub keeps a **vertical axis** (compensating parallelogram linkage): hub centre = `(L cos theta, L sin theta)` in the arm's vertical plane, the hub plane stays horizontal, so each hub's four gondolas share the hub's height and vertical acceleration.
- **Physics goes from 2D to 3D**: the mill's rotational dynamics use the cam-effective inertia `I_eff(phi)` (reach `L cos theta`, plus the vertical kinetic energy of the cam constraint) and a gravity cam torque `-V'(phi)`; gondolas feel the hub's vertical acceleration `z'' = L cos(theta) theta'' - L sin(theta) theta'^2` and the extra horizontal terms `r''` and `2 r' phi'`. The load-balance reading uses the actual reach. A new energy invariant (`E = 1/2 I_eff phi'^2 + V`) replaces the planar kinetic-energy-only invariant.
- **Safety envelope**: rate limit 0.85 rad/s, acceleration limit 2.5 rad/s^2, a minimum vertical specific force of 0.10 g (no weightlessness), and a clearance rule (ground, tower housing, neighbouring gondolas) validated at start-up and by property tests. A non-finite or limit-breaking elevation state trips the existing emergency-stop entry.
- **Telemetry (additive)**: mill angle, arm engagement state and four `ArmTelemetry` records (elevation, rate, height, reach) on `MillTelemetry`; `VerticalG` and `TotalG` on every `GondolaTelemetry`. Same SSE endpoint, event name and 30 Hz rate; old clients ignore the new fields, new clients default them when absent.
- **Visualization**: procedural tapered-tube arms, hinge pin with clevis, animated hydraulic cylinder and parallelogram linkage that keeps the hub vertical; client phase-locks its arm azimuth to the server mill angle; a draw-call and triangle budget. Glue named nodes are defined so the asset pipeline (roadmap change 11) can swap in glTF models.
- **Dashboard**: a new **arm-elevation panel** (PrimeNG, first use of PrimeNG in the app) in the telemetry rail, and the gondola panel shows forward, lateral **and vertical** G. **BREAKING (frontend-internal)**: the frontend `GForce.vertical` field today carries the backend `forwardG`; it is renamed `forward` and `vertical` now carries the true Z component.
- **Docs and tests**: new `docs/06-octopus-arm-elevation.md` (all default-value derivations); updates to `docs/01` to `docs/05` and `docs/appendix-parameters.md`; a deterministic 60 s golden-master test and a physics invariant suite (energy, symmetry, limits, clearance).

## Capabilities

### New Capabilities
- `arm-elevation`: the cam profile, parameters, phase staggering, engagement/parking lifecycle, rate and acceleration limits, clearance rule, and arm behaviour when stopped, braked, reversed or in emergency stop.
- `ride-3d-dynamics`: the 3D kinematics and dynamics that follow from elevation - vertical gondola acceleration, the G-force Z component, cam-effective inertia and gravity cam torque, energy and torque consequences, load-imbalance with reach, and the interaction with over-speed protection and single-rider loading.
- `arm-elevation-telemetry`: the additive wire contract for arm elevation, mill angle, engagement and vertical G on the 30 Hz SSE stream, its size budget and its backward/forward compatibility.
- `octopus-visualization`: the 3D rendering of the elevating arms (tapered tubes, hinge, hydraulic cylinder, hub-stays-vertical linkage), its performance budget, the procedural-placeholder-to-glTF seam, and the accessible arm-elevation panel.

### Modified Capabilities
- `ride-telemetry-streaming`: the "Full telemetry payload" requirement gains the new mill, arm and gondola fields.
- `ride-state-machine`: "Automatic condition-driven transitions", "Emergency stop from any active state" and "Safety constraints and motion are confined by state" now require the arms to be parked (not only the rotation at rest) before `Offloading`, and define arm motion as ride motion.
- `ride-telemetry-panels`: "Gondola telemetry panel" shows forward, lateral and vertical G instead of "vertical" (which meant forward).
- `ride-dashboard-layout`: "Central ride visualization" shows the arms rising and falling and the hub staying vertical.

## Impact

- **Backend, `DigitalTwin` module** (`src/DigitalTwin/FourDotnet.BoogaBooster.DigitalTwin/Domain/`): new `SpatialVector`, `CamProfile`, `ArmKinematics`, `OctopusKinematics`, `OctopusDynamics`, `ArmDrive`; changed `GreatMill`, `Hub`, `Gondola`, `Ride`, `RideKinematics`, `RotationalDynamics.Integrate` (extra external-torque argument, default 0), `RideParameters` (new constants only; no existing value changes).
- **Abstractions** (`.../FourDotnet.BoogaBooster.DigitalTwin.Abstractions/`): new `ArmEngagement`, `ArmTelemetry`; trailing defaulted members on `MillTelemetry` and `GondolaTelemetry`.
- **Tests**: `src/Tests/FourDotnet.BoogaBooster.DigitalTwin.Tests/` gets cam, dynamics, invariant, golden-master, state-machine and telemetry tests; one embedded golden file.
- **Frontend** (`src/FourDotnet.BoogaBooster.App/src/app/ride-dashboard/`): models, SSE mapping, state service, visualization (new arm model builder), new elevation panel, gondola panel, PrimeNG provider in `app.config.ts`.
- **Docs**: `docs/01` to `docs/05`, `docs/appendix-parameters.md`, new `docs/06`, `docs/README.md`; `CLAUDE.md` and `.github/copilot-instructions.md` (one sentence each; needs the `ai-tooling-sync` check).
- **Neighbouring changes** (in flight, not yet implemented): `passenger-safety` (adds `StressLevel` to the same telemetry records; its 15/18 rpm thresholds sit inside this change's limits) and `single-riders-queue` (changes load symmetry). Both are additive-compatible; see `design.md` "Dependencies and contracts".
- **Later roadmap changes**: `ecs-foundation`, `scene-graph` and `ride-on-ecs` migrate this model (see `docs/adr/game-0001-ecs-for-simulation-core.md`, `game-0004-scene-graph-transform-hierarchy.md`, `game-0006-deterministic-parallel-simulation.md`); `asset-pipeline` replaces the procedural meshes (`docs/adr/game-0007-asset-pipeline-blender-krita-gltf.md`).

> **Quality bar:** this change is for a workshop demo, not a production release. See `docs/demo-scope.md` for what may be deferred and what may not.
