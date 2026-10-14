# Design

## Context

Today the twin is planar. `Ride` owns a `GreatMill`, which owns four `Hub`s, which own four `Gondola`s (DDD aggregates, ADR-0003). `GreatMill.AdvancePhysics` integrates one angle with `RotationalDynamics.Integrate` (semi-implicit Euler, fixed 1/120 s, inertia recomputed from the load); each `Hub` integrates its own angle; each `Gondola` evaluates a horizontal field with `RideKinematics.OutwardField` (centripetal terms only, `PlanarVector`) and integrates a passive yaw pendulum. `ForwardG`/`LateralG` are horizontal; nothing is vertical. The frontend (`ride-visualization.ts`) draws arms as `BoxGeometry` that only yaw, integrates its own azimuth from the signed rpm, and maps the backend `forwardG` onto a field it calls `vertical`. The telemetry records are positional C# records streamed as JSON over SSE at 30 Hz.

Facts measured from the code that shape the design:

- Mill inertia is `4 * (800 * 6^2 / 3) + sum(AssemblyMass * 6^2)` = 251,520 kg m^2 empty and 337,920 kg m^2 fully loaded (32 x 75 kg). A hub assembly weighs 1,480 kg empty and 2,080 kg with 8 x 75 kg riders. (The comment in `RideParameters.cs` and the sanity check in `docs/appendix-parameters.md` quote older numbers; the appendix task re-baselines from code.)
- `RotationalDynamics.Integrate` clamps a coasting body to rest only when the drive torque is exactly zero; any new torque source must not break that.
- The dispatch interlock allows an imbalance up to 250 mm but does not constrain the *mass difference between adjacent pairs of arms*; that matters for vertical effects.
- `RideParameters.MaxGForce` (4.5 g) exists but nothing enforces it.
- The speed cap is 2.5 rad/s (23.9 rpm); `passenger-safety` (not yet applied) will trip at 18 rpm (1.885 rad/s) and warn at 15 rpm.
- The app pins `three` ^0.169.0, `primeng` ^22.0.0-rc.1 and `@primeuix/themes` ^3.0.0-rc.1; no component uses PrimeNG yet and no `providePrimeNG` is configured.

Numbers quoted below were computed with an offline model of the same equations (semi-implicit Euler at 1/120 s, the repo's mill and loss constants); they are expectations to be confirmed by the tests in `tasks.md`, not results of the code.

## Goals / Non-Goals

**Goals:**
- Make the four main arms rise and fall as a deterministic, kinematic function of the mill angle, and carry the consequences (vertical G, inertia, energy, imbalance) through the existing aggregate design without breaking any planar behaviour when the arms are parked.
- Keep every default derived, documented and checked by a validator and by invariants, so the numbers are teachable and cannot silently drift.
- Leave contracts (pure functions over plain data, a golden-master oracle, named render nodes) that the ECS, scene-graph, job-system and asset-pipeline changes can adopt unchanged.

**Non-Goals:**
- No new motor, axis, operator command or endpoint for the arms. The operator cannot set the elevation.
- No over-G interlock (the unenforced `MaxGForce` stays unenforced) and no change to over-speed thresholds; those belong to `passenger-safety` and later work.
- No Euler (`r * phi''`) or full Coriolis coupling of the hub frame; the existing steady-state field model is extended only by the terms that exist at constant speed (`r''`, `2 r' phi'`).
- No structural model of the tower (overturning moment, hinge loads) beyond the momentum bounds; no hydraulic circuit model (the cylinder is drawn, not simulated).
- No glTF assets and no ECS here.

## Decisions

### D1 - The elevation is a kinematic cam function of the mill angle, not a state or a motor

`theta_k = theta_park + lambda * (theta_cam,k - theta_park)` with `theta_cam,k = theta_base + A * p(n * phi + k * Delta)`, `p = cos` by default. Only `lambda` (engagement, D3) and the mill angle are state. Elevation therefore cannot drift from the angle, is path-independent (forward, reverse and coast give identical poses at identical angles) and costs four `cos`/`sin` evaluations per tick. This is how a real cam-track Octopus works.

*Alternatives rejected:* (a) an independent hydraulic axis with its own integrator and motor - adds a sixth driven axis, its own limits and an operator surface the brief excludes; (b) elevation as a function of time - breaks path independence and reversal; (c) a lookup table - hides the derivation and breaks the analytic derivatives D4 needs.

### D2 - Defaults and their derivation (full derivation goes in `docs/06-octopus-arm-elevation.md`)

| Parameter | Default | Why |
|---|---|---|
| Arm length `L`, hinge height `H` | 6.0 m, 6.0 m | Existing arm length; 6.0 m is the frontend's `towerHeight` (mirrored into `RideParameters`). |
| Park elevation | 0 deg | Arms horizontal as today, so a parked ride equals the planar ride (D3). |
| Base `theta_base` and amplitude `A` | 15 deg, 15 deg (range 0 to 30 deg) | The arms never go below horizontal (ground clearance 4.85 m, no digging). The 30 deg ceiling comes from the tower rule: nearest gondola `6 cos(theta) - 3.0 = 2.196 m` from the axis at 30 deg, limit 1.55 m, so `theta_max <= 40.7 deg`; 30 deg leaves a 0.65 m margin. Hub travel is 3.0 m. |
| Cam order `n` | 1 | The vertical acceleration scales as `L A n^2 omega^2`. With the minimum-load rule (`VerticalG >= 0.10` at the 2.5 rad/s cap) `A n^2` must stay near or below 0.26 rad for `L = 6`; `n = 1` allows the full 15 deg, `n = 2` would force `A` to about 4 deg (0.5 m of travel). The rate rule agrees: `A n omega_max = 0.655 rad/s` at `n = 1`, 1.31 at `n = 2`. |
| Stagger `Delta` | 180 deg | Opposite arms identical, adjacent arms in antiphase. Exact consequences: horizontal centre of mass stays on the axis (imbalance reading stays 0 for symmetric load), harmonics 1 to 3 of the vertical sum cancel, tower pitch/roll moments from the vertical forces cancel in pairs. `Delta = 90 deg` (a travelling wave) gives a first-order centre-of-mass offset `e = (1/2) L A sin(theta_base)` = 0.20 m, i.e. 81 % of the 250 mm interlock, and was rejected. |
| Rate limit | 0.85 rad/s | Peak cam rate at the speed cap is 0.655; the engagement ramp adds at most 0.131 (`0.5236 rad * 1.5 / 6 s`); 0.786 total, 8 % margin. |
| Acceleration limit | 2.5 rad/s^2 | Peak steady cam acceleration at the cap 1.64 (`A n^2 omega^2`), plus `A n phi''` for braking (at most about 0.22), plus the ramp (0.087): 1.95, 28 % margin. |
| Minimum vertical load | 0.10 g | Seat stays loaded at the cap; default minimum is 0.13 g. |
| Engage / park ramps | 6 s / 8 s, smooth-step | Peak ramp rate 0.131 / 0.098 rad/s; zero slope at both ends so there is no acceleration step. Parking is slower because passengers are still seated. |
| Clearance constants | underhang 1.15 m, ground 0.5 m, gondola half-length 1.0 m, tower keep-out 1.55 m (1.05 m housing radius + 0.5 m), sweep radius 3.1 m, neighbour gap 1.0 m | Taken from the frontend model dimensions (pod drop 0.55 m + half height 0.575 m, rounded up; pod length 2.0 m; hub arm 2.0 m + pod corner). Neighbour gap at equal height with defaults is 2.0 m. |

The validator (`ArmEnvelope.Validate(CamParameters)`) sweeps the angle in 1 deg steps, the engagement in 0.05 steps and the speed in 0.05 rad/s steps; the linear estimates above are for orientation only.

### D3 - Engagement factor, parking, and the hold-while-braking rule

`ArmDrive` owns progress `s` in [0, 1] and the state (`Parked`, `Engaging`, `Engaged`, `Parking`); `lambda = 3 s^2 - 2 s^3`. `Ride` drives it: entering `Started` sets the target to 1 (rate `1/6` per second); in `Stopping` and `EmergencyStop` the progress is held until the mill and hubs are at rest, then the target becomes 0 (rate `1/8` per second, so parking from a partial progress takes proportionally less); `Idle`, `Loading`, `Safe`, `Offloading` are parked by construction. `Ride.Advance` moves to `Offloading` only when rest **and** `Parked`. That keeps passengers from unboarding with raised arms and keeps the existing "Stopping/EmergencyStop -> Offloading at rest" shape, with one extra guard.

*Alternatives rejected:* (a) pure cam without parking - the ride can stop with arms at 30 deg and gondolas 7.9 m up; (b) lower the arms the moment a stop is requested - adds a `lambda'` coupling during braking and changes vertical G during an emergency; (c) a linear ramp - an acceleration impulse at both ends.

### D4 - Mill dynamics: cam-effective inertia and gravity cam torque

With `J_k = (m_arm/3 + M_k) L^2`, kinetic energy of arm `k` is `(1/2) J_k (cos^2(theta_k) phi'^2 + theta_k'^2)` (rod plus hub assembly, horizontal and vertical parts) and potential energy `g (m_arm/2 + M_k) L sin(theta_k)`. Because the cam is a scleronomic constraint, `theta_k' = (d theta_k / d phi) phi'`, so the system is one degree of freedom:

```
I_eff(phi)   = sum_k J_k (cos^2 theta_k + (dtheta_k/dphi)^2)
V(phi)       = g * sum_k (m_arm/2 + M_k) L sin theta_k
I_eff phi''  = tau_drive - tau_loss - 1/2 I_eff'(phi) phi'^2 - V'(phi)
```

`I_eff'` is analytic (needs `d theta/d phi` and `d^2 theta/d phi^2` from the cam profile's first and second derivative, which is why the profile must be `C^2`). The integrator stays semi-implicit Euler (`docs/01` 1.4): `alpha` from the current state, then `omega`, then `phi`. `RotationalDynamics.Integrate` gets one optional argument `externalTorque` (default 0, so hubs and existing tests are unchanged); the static-friction test and the coast clamp use `drive + externalTorque`, and the coast clamp (stop and stay stopped) applies when the *motor* drive is zero and `|externalTorque| <= Coulomb`. Without this, a nonzero gravity torque below static friction makes the speed chatter in sign around zero.

Offline numbers (full load): mean `I_eff` = 0.938 x planar, swing +-6.75 % twice per revolution, `V'` amplitude 5.1 kN m (empty 3.9 kN m, both under the 10 kN m Coulomb friction); under 70 % power the mean cruise speed is within 0.15 % of the planar value, 90 % of cruise is reached 2.2 s sooner, and the speed ripples +-3.6 %. Energy `E = 1/2 I_eff phi'^2 + V` drifts by 0.25 %, 0.9 % and 1.5 % of the initial kinetic energy over 60 s from 0.5, 1.8 and 2.5 rad/s with the Euler scheme; the spec tolerance is 3 %.

*Alternatives rejected:* (a) keep `alpha = tau / I_planar` and only change `I` to `L cos(theta)` - ignores the vertical kinetic energy and the gravity torque, so mechanical energy is not conserved and the invariant in `docs/01` 1.5 has no replacement; (b) angular-momentum form `d(I omega)/dt = tau` - wrong here because the cam track exerts a tangential force, so angular momentum is not conserved; (c) a numerical `I'` by finite differences - works but is noisier and costs three more profile evaluations per tick; kept only inside the tests as an independent check of the analytic derivative.

### D5 - Engagement ramps conserve horizontal angular momentum

The actuator that raises or lowers the arms exerts forces in the radial-vertical plane, so it exerts no torque about the spin axis. When `lambda` changes in a step, the mill speed is first rescaled by `I_h(phi, lambda_old) / I_h(phi, lambda_new)` (horizontal inertia only), then the cam dynamics of D4 run with `lambda` frozen for the step. The `lambda'` cross terms in the generalized momentum are neglected; the error is bounded by the ramp rate (at most 0.131 rad/s, 20 % of the cam rate, and only while the mill is slow or at rest). Documented and covered by a test that compares against a fine-step reference.

### D6 - 3D kinematics as plain pure data

`SpatialVector` (double precision, `X, Y, Z`) joins `PlanarVector`; the horizontal code is untouched. `OctopusKinematics.Compute(CamParameters, armIndex, millAngle, millOmega, millAlpha, lambda, lambdaRate, lambdaAccel)` returns a readonly struct `ArmKinematics { Theta, ThetaRate, ThetaAccel, Reach, ReachRate, ReachAccel, Height, HeightRate, HeightAccel }`. It has no state, no clock and no allocation, and arms are always evaluated in index order 0 to 3 (fixed reduction order, needed for the later parallel job system). `GreatMill.AdvancePhysics` integrates the mill (D4), computes the four `ArmKinematics` once, and passes the arm's struct into `Hub.AdvancePhysics` and then `Gondola.AdvancePhysics`. The gondola builds the hub centre from `Reach`, adds the `r''` and `2 r' phi'` terms to the horizontal field, and computes `VerticalG = (g + HeightAccel)/g`, `TotalG` from the three components. `phi''` for `theta''` is the same-step `(omega_new - omega_old)/dt`. `RideKinematics.HubCentre` and `PivotPosition` take the reach; with `lambda = 0` they return exactly the old values.

### D7 - G-force Z component and the frontend `vertical` rename

Backend: `VerticalG` (+1.000 at rest) and `TotalG` join `GondolaTelemetry`; `ForwardG`/`LateralG` keep their meaning (horizontal, no gravity). Frontend: `GForce` becomes `{ forward, lateral, vertical }`; the mapper moves `forwardG` into `forward` and `verticalG` into `vertical`; the gondola panel shows all three. This is a frontend-internal rename that makes the compiler find every consumer; the alternative (leave `vertical = forwardG` and add `trueVertical`) would keep a misleading name next to the new truth and was rejected.

### D8 - Telemetry shape and compatibility

Arms live on `MillTelemetry` (`AngleDegrees`, `ArmEngagement`, `Arms` = four `ArmTelemetry`), because the arms belong to the mill and the mill record is the one the dashboard already splits out; gondolas get `VerticalG`, `TotalG`. All are trailing parameters with defaults (`null` for `Arms`), so every `new MillTelemetry(...)` / `new GondolaTelemetry(...)` in existing code and tests still compiles; `passenger-safety` appends `Stress` to the same records and may be applied before or after (whoever applies second appends after the other's members and fixes only the positional call sites it breaks). Rounding is applied in the projection only (0.01 deg, 0.001 m, 0.001 g) to hold the estimated growth to +1.3 KB (+17 %) per 30 Hz frame; a test measures the real number against the 25 % / 2 KB budget. The elevation rate is analytic so a 30 Hz stream does not have to be differenced and the client can extrapolate.

### D9 - Server is the authority; the client phase-locks to the server angle

The elevation is a function of the *server's* mill angle, but the viewer integrates its own azimuth from rpm, so azimuth and elevation could drift apart. The frame therefore carries `AngleDegrees`; the viewer runs its local integration for smoothness and applies a first-order correction (time constant 0.25 s, never negative) toward the server angle, and extrapolates elevation with the reported rate for at most one frame. This follows the direction of `docs/adr/game-0005-simulation-authority-and-state-sync.md` (status Proposed) without depending on it. *Rejected:* computing the cam in the browser (a second copy of the cam constants that can disagree with the server) and sending only the angle (client would need the whole cam, the limits and the engagement state machine).

### D10 - Visual construction (procedural placeholders)

Built by a pure function `buildOctopusArm()` returning a `THREE.Group` so it runs under jsdom (no WebGL) in tests. Units are metres, +Y up in Three.js (physics is +Z up; the mapping is `(x, y, z) -> (x, z, -y)`, noted in the code header).

| Node | Geometry | Notes |
|---|---|---|
| `arm-root` | `Group` at the hinge | rotates about the horizontal tangent axis by `theta`; azimuth from the existing arm hinge group |
| `arm-tube` | `CylinderGeometry(0.14, 0.24, 6.0, 16)` along the arm | tapered; plus 3 collar rings (`TorusGeometry` or short cylinders) as truss hints |
| `hinge-pin` | `CylinderGeometry(0.09, 0.09, 0.9, 16)`, 2 clevis plates `BoxGeometry` | pin axis tangential; opposite arms may share a through-pin; the two pins are stacked 0.3 m apart (visual only) so they do not cross; central housing becomes a slim spindle core (r 0.45) on a base disc (r 1.05) so the pins are visible |
| `cylinder-barrel`, `cylinder-rod` | `CylinderGeometry` r 0.09 and r 0.05 | oriented each frame by `lookAt`; lugs at tower (0.8 m out, 1.4 m below the hinge) and arm (2.0 m from the hinge); pin-to-pin distance 1.844 to 2.575 m |
| `linkage-tube` | tube, same length, 0.55 m above the arm | parallelogram; ends on a vertical `hub-bracket` |
| `hub-bracket` | box 0.1 x 0.55 x 0.1 | carries the existing hub group, which is never rotated |

Budget: at most 14 added meshes per arm, 130 meshes and 80,000 triangles in total (about 55 meshes and 20,000 triangles today plus about 56 meshes and 7,000 triangles), geometry and material shared per kind, a preallocated `Vector3`/`Quaternion` scratch set so the per-frame update allocates nothing, disposal on teardown. WebGL is unavailable in jsdom, so what is automatable (structure, counts, vertical hub, cylinder length, parallelogram) is tested on the group; the actual look and frame time are a manual check with a recorded checklist in `tasks.md`.

Reduced motion keeps the existing meaning (no animation): arms are drawn parked; the panel still has the numbers.

### D11 - Safety envelope: validator, start-up check and run-time watchdog

`CamParameters` (a record with `RideParameters` defaults) is validated by `ArmEnvelope.Validate`, which returns the violated rule names. The module checks the defaults at start-up (fail fast) and the tests feed it hostile sets (n = 2, A = 25 deg, base -50 deg, n = 1.5). At run time `Ride.Advance` checks the four `ArmKinematics` for non-finite values and for rate/acceleration beyond the limits and, on a breach, performs the same entry as an operator emergency stop (`EnterEmergencyStop`), the same pattern as the NaN watchdog in `docs/01` 1.5 and the trip in `passenger-safety`. A counter `octopus.arm.limit_breach` is added through `BoogaBoosterTelemetry.Meter` (ADR-0009 pattern in the handlers; here emitted from the application service that owns the tick, not from the domain).

### D12 - Test strategy: baseline first, then invariants, then a golden master

1. **Planar baseline captured before any code changes** (task 1): a scripted 60 s run on the unmodified code is recorded to `Baselines/planar-60s.json`; after the change, a run with arms forced `Parked` must reproduce its horizontal outputs to 1e-12. This is the proof that the parked ride is unchanged.
2. **Pure-function tests** on the cam, kinematics, `I_eff`, `V'` (analytic derivative vs central difference), validator.
3. **Invariants** (property-style, seeded Bogus where randomness is needed): energy conservation and decay, symmetry (centre of mass on the axis, closed-form vertical sum `4H + 4 L sin(theta_base) cos(A cos(n phi))`, `|sum z'| <= 0.25 |phi'|`), rest reads 1.000 g, path independence, rate/acceleration sweep, minimum vertical load, clearance, no chatter, angular momentum across a `lambda` step, heaviest-legal-asymmetry start.
4. **Golden master** (`Golden/octopus-60s.v1.json`, embedded resource). Script: 16 gondolas x 2 x 75 kg, restraints secured with a fixed 5 s delay in a pre-roll (not recorded), `Safe -> Started`; at t = 0 mill 70 % and hubs 40 %; at 30 s mill direction Reverse; at 50 s `Stopping` (brake, then rest and park run into the last seconds). Recorded at each whole second: mill angle and speed, four elevations, `lambda`, `E`, hub 0 vertical acceleration, gondola 0 `VerticalG`. Second run with weights from `new Randomizer(20261014)` in [60, 90] kg that satisfy the interlocks (the test fails fast if a seed change violates them). Compared with 1e-9 absolute (relative above 1,000) because `Math.Sin/Cos` are not bit-identical across operating systems; two runs in one process must be bit-identical (`==`). Regeneration only with `BB_UPDATE_GOLDEN=1`, writes the file, and the diff is reviewed in the PR.
5. **ride-on-ecs oracle**: the golden files are the acceptance oracle for the later ECS migration (same tolerance).

### D13 - Frontend: first PrimeNG use, panel and wiring

The arm panel uses PrimeNG (`ProgressBar` for the elevation within its range, `Tag` for the engagement state, a PrimeNG `Panel`/`Card` container is optional) after the `primeng` MCP server confirms the exact 22 rc API, selectors and ARIA output. `providePrimeNG` with an `@primeuix/themes` preset is added to `app.config.ts` and must keep the dashboard's dark theme at WCAG AA contrast (checked by the axe spec). Because the pinned versions are release candidates, if the needed component is missing or its accessible output fails axe, the panel falls back to native `<progress>`/`<meter>` elements and the deviation is recorded in the PR. The numbers refresh at 4 Hz (a throttled signal), only the engagement state sits in a polite live region, and nothing in the panel is focusable. The 3D scene exports its `buildOctopusArm()` so the spec can count meshes.

### D14 - Docs are written first and are the source of every constant

`docs/06-octopus-arm-elevation.md` holds the derivations (D2 table, clearance geometry, rate/acceleration/vertical-G bounds, `I_eff` and `V`, energy, momentum, golden-master design). `docs/01` to `docs/05` and the appendix change where their statements stop being true (frames and state vector, inertia and the effective equation, losses/terminal speed note, the cart field with `r''` and Coriolis, G-force vector and imbalance with reach, the energy invariant, new parameter tables and the re-baselined sanity check). Code comments cite section numbers.

## Dependencies and contracts

**Requires from earlier changes:** none - this is the first roadmap change and builds on the current DDD code. It touches the in-flight work as follows.

- `passenger-safety` (not applied): needs `MillTelemetry`/`HubTelemetry` to gain `Stress`; this change appends different members to `MillTelemetry` (and none to `HubTelemetry`), so the two compose in either order. Its thresholds (mill 15/18 rpm, hubs 26/32 rpm) are all below the speed cap against which the arm limits are proven. The cam speed ripple (+-3.6 %, bound +-5 %) means `|rpm|` can cross 15 rpm and fall back; its trip is latched into `EmergencyStop`, so only the warning indication can flicker. Its automatic trip into `EmergencyStop` runs the sequence of D3 (hold, brake, park).
- `single-riders-queue` (not applied): partial loads change `M_k` only. Requirements in `ride-3d-dynamics` bound the effect on `V'`, the load-balance reading and the vertical momentum for any legal load.
- Published specs modified: `ride-telemetry-streaming`, `ride-state-machine`, `ride-telemetry-panels`, `ride-dashboard-layout` (delta specs in this change). `engine-braking` and `motor-direction-control` are unchanged in text; their behaviour is protected by the regression scenarios in `ride-3d-dynamics` (braking time bound, reversal through rest) and by keeping their existing tests green unchanged.

**Provides to later changes (stated as intent; names may be adjusted at apply):**

- To `ecs-foundation` and `ride-on-ecs` (docs/adr/game-0001-ecs-for-simulation-core.md, game-0002-data-oriented-component-storage.md): the physics is pure static functions over plain value structs (`CamParameters`, `ArmKinematics`, `SpatialVector`, `ArmEngagement`) with no object state, so each can become a component/system without a rewrite; per-arm data is four fixed slots indexed 0 to 3; the golden master files in D12 are the equivalence oracle.
- To `scene-graph` (game-0004-scene-graph-transform-hierarchy.md): the hinge -> arm -> upright-hub chain is the first case where a child must *not* inherit its parent's rotation; it is modelled as an explicit constraint (the hub's parent link is the arm tip position only), to be expressed as a flag on the node in the flat parent-index array.
- To `job-system` and deterministic parallel simulation (game-0003, game-0006): arm kinematics are independent per arm and are always reduced in index order 0 to 3; no shared mutation; no order-dependent float sums beyond that fixed order.
- To `ride-on-ecs`: the engagement state machine (`ArmDrive`) and the `Ride.Advance` gating (rest and `Parked`) are the behaviours to preserve.
- To `asset-pipeline` (game-0007-asset-pipeline-blender-krita-gltf.md): the named-node contract of D10, units in metres with +Y up, pivot placed at the hinge, and the rule that the animation code reads/writes only the named nodes.
- To the tycoon changes: `ArmTelemetry` and the engagement enum are stable wire members; simulation authority and snapshots follow game-0005.

**Third-party libraries:** none added. `three` (^0.169.0, `CylinderGeometry`, `TorusGeometry`, `Object3D`) and `primeng` (^22.0.0-rc.1) / `@primeuix/themes` (^3.0.0-rc.1) are already pinned; at apply time the currency check must be run: not deprecated in the pinned version (no deprecated Three.js geometry used), official docs name no different default (use the `primeng` MCP), upstream not archived and released within 12 months. Both PrimeNG pins are release candidates, which the check must record. Bogus (35.6.5) is already used by the test project for the seeded variant.

## Risks / Trade-offs

- [Energy drift of semi-implicit Euler with a position-dependent mass exceeds the 3 % tolerance] -> offline estimate is 1.5 % worst case at the speed cap; calibrate in the first red/green and, if exceeded, switch the mill to velocity-Verlet (still deterministic, one extra profile evaluation); `docs/01` allows escalation only when a test proves it, and this is that test.
- [Asymmetric load makes the gravity cam torque exceed the Coulomb friction (up to about 20 kN m for 600 kg pair difference, about 40 kN m at the legal extreme)] -> the mill rolls to a potential minimum when unbraked and unpowered, which is physical (see-saw); the spec bounds it below 70 % of stall so the ride can always start, and tests cover both.
- [Speed ripple of +-3.6 % flickers the `passenger-safety` warning near 15 rpm] -> documented; the trip itself is latched; a hysteresis belongs in that change and is not added here.
- [Total G can exceed `MaxGForce` because vertical G now adds to horizontal G (about 3.8 g horizontal at the cap already)] -> no new enforcement (non-goal); `TotalG` is reported so a later change can enforce it; noted in `docs/05`.
- [Quasi-static treatment of `lambda'` in the mill equation] -> bounded (D5); tested against a fine-step reference; both ramps are slow and happen at low speed.
- [Frontend rename `GForce.vertical` -> `forward` misses a consumer] -> the type change makes the compiler list every use; panel, mapper, state service, fakes and specs are tasks.
- [PrimeNG is a release candidate and not yet configured] -> consult the MCP server first; native fallback recorded in the PR.
- [jsdom has no WebGL, so look and frame time are not automated] -> scene-structure tests plus a manual checklist task with recorded results.
- [Golden files differ across operating systems at 1e-12] -> 1e-9 tolerance; strict equality only within a process.
- [Frame growth (+1.3 KB per frame, about 17 %) at 30 Hz per client] -> rounding in the projection, 25 %/2 KB budget test; if exceeded, drop `TotalG` (derivable) first.
- [Visual hinge offsets (pins stacked 0.3 m, spindle core) differ slightly from the physical hinge on the axis] -> visual only; physics uses the axis hinge; the hub position is always taken from the reported reach and height.
- [Pre-existing doc drift: the appendix and a comment in `RideParameters.cs` quote older inertia numbers] -> the appendix task re-derives from code; the comment fix is in scope only where this change edits the same block.
- [A start at standstill raises the arms while the mill is still] -> intended (the ride visibly "wakes up"); the ramp is slow and the hub stays level.

## Migration Plan

Additive on the wire and in the domain; no persistence (the ride is in-memory). Apply on a branch in a worktree per the repo workflow, in the group order of `tasks.md`; backend and frontend can ship together or backend first (a new server with an old client only adds ignored fields; a new client with an old server defaults the missing fields, see `arm-elevation-telemetry`). Rollback is reverting the branch; the planar baseline test keeps proving the parked ride equals the old ride.

## Open Questions

- Final tolerance values (energy 3 %, speed ripple 5 %, spin-up advantage 1.0 to 3.5 s) are calibrated on the first green run and may be tightened without changing the specs' structure.
- Exact PrimeNG components for the panel are confirmed with the `primeng` MCP at apply time (D13); the spec only requires accessible bars and a text state.
- Whether `TotalG` should be dropped from the wire to save bytes if the size budget is tight (it is derivable on the client).
