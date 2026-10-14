## Context

Motivation and scope: `proposal.md`. Requirements: the 14 specs under `specs/`. This document is the how.

State of the system when this change is applied (apply order 10): the `Simulation` module (changes 2 to 5) provides ECS storage, systems, the fixed 1/120 s tick driver, the scene graph and a deterministic work-stealing job system; the Octopus runs on it. `park-world-grid` (6), `ride-catalog` (7), `guests-and-needs` (8) and `park-economy` (9) are specified after this change was written, so every contract with them below is **intent**, stated as a port in `Coaster.Abstractions` that the apply agent reconciles with what actually landed (see "Dependencies and contracts"). The Octopus physics already defines the felt-force maths this change reuses (`docs/05-forces-and-g-forces.md` section 5.1, and `openspec/changes/octopus-motion/specs/ride-3d-dynamics/spec.md`: rest reads exactly 1.000 g, `TotalG = sqrt(Forward^2 + Lateral^2 + Vertical^2)`).

Conventions fixed by earlier changes and reused: right-handed coordinates, **+Y up** (Three.js identical), doubles on the server, floats only inside Three.js (`scene-graph` D4); fixed step `SimulationParameters.FixedStepHz = 120` (`ecs-foundation` D8); components are `unmanaged` structs with property accessors, registered by stable name with a schema version and optional replication schema (`ecs-foundation` D3, D4); structural change through the command buffer (D6); deterministic parallelism by fixed chunks and fixed-order reduction (ADR `docs/adr/game-0006-deterministic-parallel-simulation.md`); backend authority with snapshots and deltas (ADR `docs/adr/game-0005-simulation-authority-and-state-sync.md`, status Proposed).

Research basis: `docs/research/rct-gameplay.md` sections 6, 7 and 15.5, used as **behavioural background only** (see "Provenance and legal"). Labels in that document are kept here: VERIFIED, INFERRED, UNKNOWN. Anything marked UNKNOWN there is an assumption or an apply-time task here, never a fact.

## Goals / Non-Goals

**Goals:**
- A player can draw a coaster piece by piece, see at once what is wrong with it, test it, and be told where and why it fails.
- Coaster behaviour (speed, G, ratings) is **measured from the track**, by the same fixed-step dynamics that run live; no stored per-ride numbers.
- A pure, small kernel (geometry, dynamics, metrics) that is testable against closed-form physics and independent of ECS, HTTP and Angular, plus thin adapters.
- Determinism by construction: byte-identical files, bit-identical test runs, thread-count-independent live simulation.
- A demo-ready slice (Tier 1) that stands on its own.
- Teachable: each concept is one small file, one doc section and a test that names the rule.

**Non-Goals:**
- No free-form spline editing in Tiers 1 and 2 (control points are Tier 3 at most).
- No rigid-body or per-wheel physics, no track flex, no wind (the model is 1D along the centreline).
- No economy, queue, guest or breakdown *policy*: this change defines hooks and ports and consumes ratings contracts, it does not decide prices, who queues or how often a ride breaks.
- No scenery, no themed pieces, no rider animation, no sound.
- No persistence beyond the track file and the park save (see D17).
- No glTF models (procedural placeholders; `asset-pipeline` replaces them later).
- No copy of any other game's code, tables, numbers or assets.

## Decisions

### D1. A new module `Coaster`, a pure kernel inside it, thin ECS adapters

New projects `src/Coaster/FourDotnet.BoogaBooster.Coaster` and `.Abstractions`, tests `src/Tests/FourDotnet.BoogaBooster.Coaster.Tests`, benchmarks in the project created by `ecs-foundation`. The module follows the standard anatomy (`CoasterModuleExtensions`, options, `Domain/`, `Application/`, `Features/<Feature>/`, `Endpoints/`) with two additions that mirror the `Simulation` module's engine-code precedent:

```
Coaster/FourDotnet.BoogaBooster.Coaster/
  Domain/            CoasterDesign (aggregate, DomainModel per ADR-0003), TrackPiece, PieceCatalogue,
                     DesignHistory, validation rules (domain services), CoasterParameters.cs
  Track/             pure geometry: curve primitives, piece placement, CompiledTrack, frames
  Dynamics/          pure kernel: TrainParameters, TrainState, TrainStepper, zones, MetricsAccumulator,
                     RatingMapper, TestRunner, Diagnosis        (no ECS, no I/O, no allocation in Step)
  Simulation/        ECS adapters: components, systems, gateway calls, incident events
  Application/       design store, track registry, test-run gate, ports' default adapters
  Features/          <Feature>Command|Query + handler (ExecuteAsync), one folder per feature
  Endpoints/         MapCoasterEndpoints() - no logic
```

*Why a module of its own and not a folder in `Simulation`:* `Simulation` is engine infrastructure (ECS, scene graph, jobs); the coaster is gameplay with its own invariants (a design is an aggregate with undo and validation). ADR game-0001's boundary says aggregates live in DDD modules, per-tick runtime state lives in the World, and flow is aggregate to world by gateway, world to aggregate by events. *Why not `DigitalTwin`:* it is the Octopus twin and this change must not touch it. *Why not `Controller`:* it is an empty scaffold whose stated purpose (scripted ride programmes) is different; reusing the name would confuse. The `Dynamics/` kernel is pure so the **same** `TrainStepper.Step` runs in a test run (a plain loop on the request thread) and in the live ECS system, which is what makes "test run = live behaviour" true by construction.

*Alternative rejected:* implement dynamics directly as ECS systems and run the test run by spinning up a private `World`. That would make the test run pay world setup, couple the diagnosis to the ECS, and make closed-form unit tests awkward.

### D2. The track model: an integer lattice of connectable pieces

Our own model, chosen for four reasons: connectivity and closure are exact integer arithmetic, the file format needs no floating point, the palette can show only valid next pieces, and every piece can be validated and tested alone.

- **Lattice.** Positions are (tile x, tile z, height step); tile edge `T = 4.0 m`, height step `H = 1.0 m` (assumption A1: `park-world-grid` fixes the real values; `TrackLattice` reads them from the grid port and these are the defaults). Heading is a quarter turn: 0 east (+X), 1 north (-Z), 2 west, 3 south; a left turn adds 1. With +Y up this is a counter-clockwise turn seen from above, and `left = up x forward` (checked: facing +X, left is -Z). Assumption A2: the park grid's "north" is -Z; if not, only the label changes.
- **Frame.** A piece has an entry frame and an exit frame, each (position, heading, slope class, bank class). Slope class is an integer number of height steps per tile, so the tangent of the climb angle is `class x H / T = class / 4`: class 1 is 14.04 degrees, class 2 is 26.57 degrees, class 4 is 45 degrees, class 8 is 63.43 degrees (Tier 2). Bank class is 0 in Tier 1; +-1 is 45 degrees in Tier 2. Pieces connect when exit classes equal entry classes. This makes the palette a one-line filter, and a file cannot describe a kink.
- **Primitives.** Line (straights, slopes), circular arc with optional constant slope (curves, helix), parabolic pitch transition. Tier 3 adds cubic Bezier segments for loops and corkscrews. A piece is data: id, shape, parameters, frames, drive mode, tier.
- **Why parabolas for pitch changes (derivation).** Between slope classes `a` and `b` (as tangents `ta = a/4`, `tb = b/4`) over horizontal length `L`, the profile `y(x) = ta x + (tb - ta) x^2 / (2L)` has slope `ta` at 0 and `tb` at `L`, **constant vertical curvature** `(tb - ta) / L` (to first order, exact at the vertex), and rise `L (ta + tb) / 2`. In steps: `rise = n (a + b) / 2` for `n` tiles, an integer whenever `n` is even. The minimum vertical radius is `L / (tb - ta) = 16 n / |b - a|` metres, so requiring at least 8 m gives `n >= |b - a| / 2`; with `n = max(2, 2 ceil(|b - a| / 4))` all pairs up to `|b - a| = 4` use 2 tiles (R = 8 m for 4, 16 m for 2, 32 m for 1), and steep pairs scale (|b - a| = 8: 4 tiles, R = 8 m). Constant curvature gives a flat G profile through the transition, which the diagnosis can then attribute cleanly.
- **Horizontal curves** are 90 degree arcs of radius 2 or 4 tiles (8 m or 16 m), the minimum radius 8 m being a catalogue invariant (spec). Lateral G at speed `v` is `v^2 / (R g)`: at the energy-bound speed of the starter, 14.9 m/s, radius 16 m gives 1.41 g and radius 8 m gives 2.83 g. That 2.8 versus 1.4 difference is what the `starter-tight-curves` workshop fixture shows.
- **Alternatives rejected:** (a) free control-point splines (Bezier/Catmull-Rom everywhere): closure and connectivity become numeric, validation needs tolerances, the file format needs floats, and the palette idea disappears; kept for Tier 3 pieces only, as fixed data. (b) Tile-based "voxel" track (RCT-like element placement): hides curvature behind a table, which is exactly the part this change wants to compute. (c) Clothoids for all transitions: nicer jerk, no closed forms, and `Math.Sin/Cos`-heavy; rejected for determinism and cost, noted in Risks.

### D3. Arc-length parametrisation: per-piece uniform sampling into struct-of-arrays tables

`CompiledTrack` is built by placing each piece (lattice translation plus a rotation by k x 90 degrees, using exact 0/1/-1 cosines and sines) and sampling it at `n = max(1, round(L / 0.25 m))` equal arc-length intervals. Arc length for the line and arc primitives is closed form. For the parabola, `L = integral sqrt(1 + y'^2) dx` is evaluated with composite Simpson's rule on 64 sub-intervals per tile (uses only `sqrt`), and the sample abscissae are found by bisection on that cumulative table followed by linear interpolation inside one sub-interval (error bounded below 1e-9 m by the tolerance test against a 4096-interval reference). Positions on the circle and helix use `Math.Sin/Cos` **only inside piece generation**; the dynamics-facing tables are then plain doubles.

Layout (SoA, hot and cold split):
- **Hot** (used by every dynamics step): `S[]` (cumulative arc length at samples), `Slope[]` (height gain per metre in each interval, from `Hbar`), `Hbar[]`, `Zone[]` (byte) and a small zone table. About 25 bytes per sample, 31 KB for a 300 m coaster, 15 MB for 500 coasters. `Hbar[]` depends on car count, so it is built lazily per (track, car count) and cached.
- **Cold** (test run, structure endpoint, tooling): positions, tangents, curvature vectors, rider frames, roll, heading. About 120 bytes per sample. Built on demand and kept in a small LRU (8 tracks); live dynamics never touches them. 500 coasters would otherwise cost about 75 MB for data nobody reads per tick.

Lookup is `i = hint; while (S[i] > s) i--; while (S[i+1] <= s) i++;` with the hint stored in the train state, so it is amortised O(1) and a test checks it equals binary search. For closed tracks `s` wraps modulo the length.

*Alternatives rejected:* one global uniform spacing (pieces are not multiples of 0.25 m, so spacing would be discontinuous or samples would miss piece boundaries); cubic Hermite sample interpolation (position error is already below 1 mm with chords, see spec; the extra cost buys nothing); storing curvature as a scalar (the felt force needs its direction).

### D4. Frames: rotation-minimising transport plus roll

The rider frame (T, U, L) is produced per piece by carrying the previous frame along the curve (rotation-minimising, so no spurious twist on flat circuits) and then applying the piece's roll about T (smooth step over the piece for `roll:a>b`). A flat unbanked circuit therefore has `U = (0, 1, 0)` exactly, which the rest check and the lateral/vertical decomposition rely on. For pieces with a vertical loop (Tier 3) the transport avoids the Frenet frame's flip at inflections, which is why Frenet is not used.

### D5. Dynamics: a 1D energy-conserving step with exact sub-solutions

Rejected first, so the choice is visible:
- **Symplectic Euler on `a(s, v)`** (the repo's physics style for rotation): energy oscillates around the true value rather than being monotone, so "energy never increases without a lift" could only be tested with a tolerance that hides bugs.
- **RK4:** not energy-monotone either, and slower per step.
- **Scalar lookup models used by other games** (acceleration from a pitch table, shifted constants): not physical, not derived from the repo's G maths, and not ours to copy (see legal).
- **Full 3D rigid body:** no consumer needs it.

Chosen: **operator splitting with exact sub-steps.** The potential is `U(S) = m g Hbar(S)` with `Hbar` piecewise linear on the samples, so gravity is a constant acceleration `a_i = -g Slope_i` inside each interval. One step of `dt = 1/120 s`:

```
if resting and |a_i| <= mu g cos(theta_i): stay                       // static rule
1 free motion:  t = dt
   repeat (at most 16 times): within interval i, solve d = V t + a_i t^2 / 2 exactly for the
          time to the interval end, the turning point (V crosses 0) or the end of t; advance S, V
          (V^2 = V0^2 + 2 a_i d is the energy balance), switch interval, reverse at turning points
2 rolling:      |V| <- max(0, |V| - mu g cos(theta) dt)               // never reverses
3 drag:         V   <- V / (1 + c |V| dt),  c = rho CdA / (2 m)        // exact for dV/dt = -c V|V|
4 zones (by the lead car at the new S):
     lift:    V <- max(V, min(V_lift, V_prev + a_servo dt)); V >= 0 (anti-rollback); EnergyAdded += dE
     trim:    V <- max(V_trim, V - a_trim dt) if V > V_trim
     station: V <- min(V, max(V - a_st dt, sqrt(2 a_st d))) toward the stop point; snap when d < 0.02 m and V < 0.02
     drive:   leaving the station / block release: V <- min(V_drive, V + a_servo dt)
5 caps:         |V| <= 40 m/s (flag Overspeed); non-finite => stop and flag Fault
```

Why this satisfies the energy property **exactly**: step 1 is the exact solution of a conservative system on the piecewise-linear potential, so `E = m V^2/2 + m g Hbar(S)` is conserved up to double rounding; steps 2, 3 and the brake and station cases only shrink `|V|` at fixed `S`; the lift (and the drive) are the only code that can raise `E`, and they account the rise in `EnergyAdded`. The property test (spec) therefore uses a tolerance of 1e-9 relative, which is rounding, not slack for a drifting method. Splitting error (rolling and drag applied after gravity within one step) is first order in `dt`: at the worst case it is about `mu g dt = 0.0008 m/s` per step, below the 0.05 m/s tolerances of the closed-form tests; the drag-only and rolling-only cases are exact because they involve one force.

*Train as a chain of cars:* the potential uses the car-averaged height `Hbar(S) = mean_k h(S - 2.0 k)`, evaluated at the sample points and interpolated linearly. The error against the true mean car height is bounded by `(Delta s)^2 / 8` times the vertical curvature, at most 0.8 mm for radius 10 m and 0.25 m spacing (spec bound 2 mm). Zones act on the lead car (assumption A3: rules for minimum zone lengths, spec `coaster-track-validation`, keep rear cars from being stranded).

*Mass:* gravity and rolling resistance are mass independent; only drag depends on mass (heavier loses less), and the lift's work scales with mass. This gives the "mass independence with drag off" property.

*Why the speed cap is a flag and not a clamp-only:* a train reaching 40 m/s is a design error or a bug, and the live system turns it into an incident (Tier 2); the test run turns it into `Faulted`.

### D6. The felt force, computed after the step

For the step ending at `S` with speed `V` and tangential acceleration `a_t = (V_new - V_old) / dt`: `a = a_t T + V^2 kappa`, `f = a - g_vec` with `g_vec = (0, -9.81, 0)` (Y up; octopus docs use Z up, the **definition** `f = a - g`, `G = |f| / g` is identical). Decompose: longitudinal `f.T / g`, vertical `f.U / g`, lateral `f.L / g` (positive to the rider's right). Rest is exactly (0, 1, 0). Checks tied to closed forms (spec): flat radius 16 m at 12.528 m/s gives lateral 1.000 (`V^2 = R g`); vertex of `pitch:-2>+2` (vertical radius 8 m) at 8.859 m/s gives vertical 2.000, the crest vertex at the same speed gives 0. A frictionless train on a straight slope has longitudinal G exactly 0.

Peak statistics use a boxcar of 12 steps (0.1 s) to ignore join artefacts (curvature jumps at joins are real, one-step spikes at them are sampling). G is evaluated at the lead car (assumption A4; the back car sees more airtime on crests; a measured-car option is Tier 2 candidate). The octopus code lives in `DigitalTwin` and is not reachable from this module (modules reference only `.Abstractions`); the maths is re-implemented here from the shared definition and both are tied by the numeric checks above (see Open questions: lifting it into `Shared`).

### D7. Ratings: an original saturating sum, in hundredths at the catalogue boundary

`rating = base + sum_k w_k * x_k / (x_k + h_k)` (`Saturate` uses only `+` and `/`, so it is IEEE-exact), times two penalties, rounded half away from zero to hundredths. This is deliberately a smooth, monotone, bounded-per-term function: easy to teach and to test (monotonicity is a property test), unlike per-piece lookup tables. The coefficients below are this project's own tuning (assumption A5); they are chosen so the shipped starter lands in plausible bands and are pinned by golden records, not derived from any other game.

| Term (x, half-saturation h) | Excitement w | Intensity w | Nausea w |
|---|---|---|---|
| base | 0.50 | 0.20 | 0.10 |
| max speed v (14 m/s) | 1.6 | 1.5 | 0.3 |
| average speed (8 m/s) | 1.0 | 0.8 | 0.0 |
| highest drop (12 m) | 1.2 | 1.0 | 0.4 |
| drop count (2) | 0.6 | 0.4 | 0.3 |
| length (400 m) | 0.5 | 0.0 | 0.0 |
| positive G above 1 (1.5 g) | 0.5 | 2.0 | 0.5 |
| negative G depth below 1 (1.0 g) | 0.8 | 1.2 | 0.8 |
| lateral G (1.5 g) | 0.0 | 1.0 | 1.5 |
| airtime (3 s) | 1.0 | 0.5 | 1.2 |
| inversions (2) | 1.5 | 1.8 | 1.5 |
| turns, quarter turns (6) | 0.4 | 0.2 | 0.4 |

Penalties: excitement `x min(1, duration / 30 s)` and `x 1 / (1 + 0.2 max(0, intensity - 8))`. Worked example (the spec scenario): metrics (14 m/s, 8 m/s, 12 m, 2 drops, 400 m, 2.5 g, 0.5 g minimum, 1.5 g lateral, 0 s, 0, 6 turns, 60 s) give exactly 3.6667, 4.05, 2.0667, rounded 367, 405, 207. A saturating term at its half-saturation value contributes exactly half its weight, and most terms of the example sit there, which is why it is checkable by hand.

The **maximum possible** raw values (all terms saturated) are 9.6, 10.6 and 7.0, so intensity can exceed 10.00 and excitement is pulled down by the intensity penalty, which models the idea that very intense rides are less enjoyable without a cliff. Calibration target bands for the starter (confirmed by the apply run, A5): excitement 2.0 to 4.0, intensity 2.0 to 5.5, nausea 0.5 to 3.0. The ideal averages that `park-economy` wants (research section 2, INFERRED) are that change's concern; this one exposes the three numbers.

### D8. The test run: the same stepper in a plain loop, then an explainer

`TestRunner.Run(design, parameters)` compiles (or fetches the cached compile), builds a 4-car train with all seats at 75 kg (assumption A6, the figure used in the octopus scenarios), starts it at the stop point, and loops `Step` until the lap ends, the first failure, or 72,000 steps (600 s). Metrics accumulate in `MetricsAccumulator` (a struct updated per step: running maxima, boxcar buffers, airtime, drop detector with hysteresis, heading integrator). The G trace is decimated to 30 Hz (every 4th step) with the stride doubling past 6,000 samples. Static metrics (length, inversions, turns, lift height) come from the compiled track and do not depend on the run.

*Stall detection* is |V| < 0.05 m/s for 60 consecutive steps outside the platform. *Explanation:* from the stop sample the explainer scans ahead for the first local maximum of `Hbar` (the obstacle), reports `deficit = Hbar(obstacle) - Hbar(stop)` (at least 0.05 m), `energy deficit = m g deficit`, the piece of the stop and of the obstacle, and a suggestion in height steps `ceil(deficit / H)`. The lower bound is deliberate: it ignores additional losses, so the suggestion may be one step short and the player simply tests again; the message says "at least". For a stall just after a lift, the analytic reach is `lift_top + V_lift^2 / (2 g) = +0.3186 m`, which the tall-hill scenario uses as its test oracle.

*G events* merge consecutive over-limit windows into one event with the peak, its piece and arc length. Warn and hard limits are configuration (A7, own values; the Octopus `G_max` of 4.5 g from `docs/appendix-parameters.md` is reused as the hard vertical limit so the repo has one number for "too much").

### D9. Validation: pure rules over the design and a park occupancy port

Rules are small classes behind one interface, run in a fixed order, and each returns zero or more findings; the report is sorted (piece index, code). Collision and terrain use a voxel envelope (2 m wide, 0.5 m below to 2.5 m above the centreline) quantised on cells of one tile by one tile by one height step. Voxel sharing between *adjacent* pieces is allowed; any other sharing is a collision. Vertical clearance falls out of the envelope: 3 m high plus rounding makes the minimum separation 4 steps at lattice heights, which the spec's 3-versus-4-step scenarios pin. Occupancy of the park (terrain height, path tiles, other rides, owned land) comes through `IParkOccupancy`. Codes: `OpenCircuit`, `NoStation`, `StationTooShort`, `StationApproach`, `NoApproachBrake` (W), `NoLift` (W), `LiftTooShort`, `TrackCollision`, `TerrainCollision`, `PathClearance`, `OutOfBounds`, `PieceLimit`, `HeightLimit`, `JoinMismatch`, `BlockTooShort` (T2), `BlockBrakeTooShort` (T2, from the test run), `SpanTooLong` (T2).

Validation is a pure function; the builder runs it after each edit (a 256-piece design validates in a few milliseconds, budget C4) and the preview runs it on `design + candidate`. Closed-circuit search for the Tier 2 suggestion is a bounded breadth-first search over palette pieces with fixed tie-break order, depth 12, early exit on first closure; it prunes with the Manhattan distance of the gap versus the maximum reach per piece (4 tiles for curves) to stay under a node cap of 200,000, after which it answers "none within limit".

### D10. The design aggregate, undo and redo

`CoasterDesign` is a `DomainModel` (ADR-0003 as used by `DigitalTwin`; the style-guide MCP server is consulted first) holding name, ride type, anchor, train configuration, the immutable piece list, revision and state. Operations (`Append`, `RemoveLast`, `Replace` in Tier 2) return the new state; `DesignHistory` keeps **snapshots** of the piece list (at most 200, each an immutable array of at most 256 piece references, so the cost is bounded by about 50 KB), not inverse operations: a snapshot undo is exact by construction and the property "undo restores the previous hash" cannot be broken by an asymmetric inverse. The compiled-track cache is keyed by (hash of the piece-list prefix), so appending reuses the compiled prefix and undoing a trailing append is a cache hit. Concurrency: one lock per design and a revision check, so two clients cannot interleave.

### D11. ECS integration (the live simulation)

The server design object is the aggregate; the live ride is entities. `Build` spawns, through the simulation gateway of `ecs-foundation`/`ride-on-ecs`, one coaster entity and one entity per train. Components (unmanaged, registered by stable names, schema version 1):

| Component | Data | Writer |
|---|---|---|
| `CoasterRide` | track id, 64-bit track hash prefix, status (Closed, Open, Broken, Crashed), fault kind, train count, cars, seats per car | `coaster.commands`, `coaster.blocks` |
| `CoasterTrains` | train entity ids (inline array of 8), count | spawn |
| `CoasterBlocks` (T2) | occupant train index + 1 per block (inline array of 32), block count | `coaster.blocks` |
| `TrainState` | coaster entity, train index, phase, flags, `S`, `V`, dwell ticks, lap ticks, sample hint, `EnergyAdded`, `EnergyBraked`, mirrored occupied count, payload mass | `coaster.train-dynamics` |
| `TrainSeats` (T2) | guest entity ids (inline array of 32), occupied count, payload mass | **guest system** |

The compiled track is **not** a component: it is immutable shared data in a `TrackRegistry` (id to compiled hot tables), referenced from `CoasterRide`; the 64-bit hash prefix is part of the component so the canonical world hash changes if the track differs, and a restore into a registry that lacks the track fails with a clear error naming the id. Systems (declared access; registration order is the reference semantics for the job graph of `job-system`):

| System | Reads | Writes | Parallel unit |
|---|---|---|---|
| `coaster.commands` | command queue | `CoasterRide` | one job |
| `coaster.train-dynamics` | `CoasterRide`, `CoasterBlocks`, `TrainSeats` | `TrainState` | fixed chunks of the train query (grain 256) |
| `coaster.blocks` | `TrainState`, `CoasterTrains` | `CoasterBlocks`, `CoasterRide`, incidents (command buffer) | one coaster per job |

The guest boarding system (change 8) is ordered before `coaster.commands` by an explicit `UpdateBefore` constraint, so seats written in a tick are seen in the same tick. **Why block occupancy is "as of the previous step" and still safe:** a train may leave a block brake only into a block that was free at the end of the previous step; a block gets occupied when a train's front enters it, and only the train in the block before the brake can do that, and there is at most one such train, so two trains cannot both claim a free block in the same step. The one-step lag only delays a release by 8.3 ms. This is what lets the dynamics system be a plain parallel-for over trains with disjoint writes, in line with ADR game-0006 rule 1.

Replication: `TrainState` registers a `ReplicationSchema` with fields `s`, `v`, `phase`, `occupied`; nothing else about a train's position leaves the server. The client reconstructs everything from the compiled track.

### D12. Station cycle, guest handoff, blocks, faults (Tier 1 minimal, Tier 2 full)

Phases (`TrainState.Phase`): Unloading, Loading, ReadyToDispatch, OnTrack, Arriving, Holding (block), Stopped (fault), Crashed. Durations are counted in ticks (3 s, 5 s, 2 s, 30 s max load, 8 s max unload). Tier 1 runs one train and an empty cycle (10 s) so the ride is visible; Tier 2 adds the seat buffer contract: **the coaster reads only `TrainSeats.Occupied` and `PayloadKg`**; it recounts the seats if the count disagrees with the non-empty seats and increments `seat_repairs`, so a guest-system bug is visible rather than silent. Blocks: the block brake piece is a hold-or-pass zone; `BlockBrakeTooShort` comes from the test run's speed at the zone entry so every train can always be held. Faults and crashes: a `RideFault` command sets `SafetyCutOut`/`RestraintsOpen` (policy of *when* belongs to `ride-catalog`/`park-economy`); the coaster only reacts (no dispatch, brakes hold, lift stops with anti-rollback) and emits `CoasterIncident` events through the ECS event buffer of `ride-on-ecs` (if the buffer type differs, the apply agent adapts; fallback: transient entities with an `Incident` component destroyed after the drain). Crashes (negative gap, overspeed) stop the trains and close the ride until cleared.

### D13. Track file: canonical JSON, integer-only, SHA-256

JSON is chosen over a binary format because piece ids are strings, the file is hand-editable in a workshop, diffs are readable, and canonical output is simple to specify (fixed member order, no whitespace, no floats). Hash: SHA-256 from the BCL (`System.Security.Cryptography`), lower-case hex; chosen over the FNV-1a of `ecs-foundation` because this hash is a persisted content identity, not a test oracle, and no dependency is added. Versioning: integer `version`, migration chain, strict reader (spec). Alternatives rejected: MessagePack (the Automaat cache format constraint is unrelated; no need for compactness at 256 strings), STJ default output (member order and escaping are not specified as a stability guarantee, so the writer is hand-rolled over `Utf8JsonWriter` with explicit order, verified by a byte-level test).

### D14. Sync and API

Per ADR game-0005 (Proposed): the server is authoritative; commands are HTTP POSTs; the response to every edit is the full editing view (a design is at most 256 short strings, a few KB). **Geometry is not sent per design**: the client downloads the catalogue geometry once (piece-local sample tables, ETag cached) and composes the track from the placed frames, because the catalogue is closed (Tier-limited), pieces are rigid and the composition is concatenation. That keeps the editing round trip small, makes the ghost instant (no request), and removes a second source of truth for curve maths. The cost is that client and server must concatenate identically; a shared numeric fixture (sample count, total length, every 50th position) generated by the server tests is reproduced by a Vitest spec (same technique as `scene-graph` D10).

Live train state: Tier 1 uses the ECS snapshot JSON of `ecs-foundation` (`GET /simulation/snapshot`) polled by the client (assumed 5 Hz) with dead reckoning; Tier 2 uses the quantised delta record of ADR game-0005 once that stream exists (`uint32` millimetres, `int16` centimetres per second, `uint8` phase: 7 bytes plus a 4-byte train index, 11 bytes, inside the 12-byte spec limit). ADR game-0005's target is at most 100 KB/s per client at 5,000 guests in a 300-guest viewport; coaster deltas for trains in view add at most 180 KB/s at the unrealistic 500 trains in view and are about 7 KB/s for a normal 20 (INFERRED arithmetic: 20 x 12 B x 30 Hz = 7.2 KB/s), which interest management by viewport keeps within budget; the final binary transport decision belongs to the park stream change.

### D15. Client architecture (Angular 22, Three.js, PrimeNG)

```
src/FourDotnet.BoogaBooster.App/src/app/coaster/
  track/        pure TypeScript, no Angular, no Three.js: catalogue.models, composer.ts (placed frames to
                sample table), dead-reckoning.ts, camera-pose.ts, trace.ts, fixtures (copied numbers)
  mesh/         Three.js builders: rail-tube, ties, supports, station deck, ghost, markers, disposal spy hooks
  editor/       Angular standalone components (lazy route /coaster/editor), CoasterEditorStore (signals),
                palette, pieces table, checks, test-run panel, chart (SVG), help dialog, live region
  data/         CoasterApiService (HttpClient), DTO models, mappers
```

Reasons for the split follow `ecs-foundation` D13 and `scene-graph` D9: the numeric parts are plain TypeScript so Vitest tests them without a DOM; Three.js is touched only in `mesh/` and the view component. The existing `ride-visualization.ts` is not changed. PrimeNG 22 and `@primeuix/themes` are already installed and mandated for new UI; the `primeng` MCP server is consulted before choosing among candidate components (toolbar, button, listbox or data view, tabs, table, message, progress bar, dialog, tooltip, splitter). The G chart is **custom SVG** in the first cut (a line chart with shaded ranges, a cursor and a data-table alternative) because PrimeNG's chart wrapper needs Chart.js as a peer dependency, which would be a new third-party dependency (candidate; if chosen later it must pass the currency check: not deprecated in the pinned version, official docs name no different default, upstream not archived and released within 12 months). Three.js is already pinned (0.169 at the time of writing; the version is read from `package.json` at apply time); the APIs used (`BufferGeometry`, `InstancedMesh`, `Float32BufferAttribute`, `MeshStandardMaterial`, `BufferGeometryUtils.mergeGeometries` from the examples package) are checked against the pinned version for deprecation before use.

Mesh generation details (numbers in the spec): rails are two 6-sided tubes, ties and columns are `InstancedMesh`es, everything for one coaster merges into at most 6 draw calls. Each piece owns its geometry chunk, so appending creates one chunk and merges it into the group, and removing disposes one; a full rebuild only when the catalogue, terrain or a non-trailing piece changes. The ghost is the same builder fed with the catalogue geometry placed at the end frame, material at 45 % opacity.

### D16. Ride-along camera

The eye pose is a pure function `cameraPose(track, s)` in `track/camera-pose.ts` (position = sample position + 1.15 m x up; basis = rider frame) so the live ride and the replay share it and Vitest tests it. A comfort layer scales roll (default 100 %, 30 % under `prefers-reduced-motion`), clamps look-around (Tier 2) and hides the first car mesh. No smoothing by default so the player sees what the physics produced; the stabilised horizon option is the comfort mechanism.

### D17. Persistence: none beyond the file and the park save

Designs live in an in-memory `ICoasterDesignStore` (thread-safe dictionary of aggregates), like `IRideStore` in `DigitalTwin`. A server restart loses unsaved drafts; export and the park save embed the canonical file. Persisting drafts to a database would be a separate change with its own migration; the spec states the limitation to the player. No database access happens in this change.

### D18. Observability

Instruments on the shared `BoogaBoosterTelemetry.Meter` (registered once in `ServiceDefaults`; this change adds none to the host): `boogabooster.coaster.test_run.duration` (ms histogram), `boogabooster.coaster.test_run.completed` (counter, tag `outcome`), `boogabooster.coaster.validation.duration`, `boogabooster.coaster.seat_repairs`, `boogabooster.coaster.incidents` (tag `kind`), `boogabooster.coaster.designs` (observable gauge). No span per step or per train; handlers get spans from the `CommandHandler`/`QueryHandler` bases. Per-system timings come from the job system's timeline and the `ride-on-ecs` timings dashboard once the systems are registered.

### D19. Parameters and where numbers live

All constants are in `Domain/CoasterParameters.cs` (properties on a static class or `const`, no magic numbers elsewhere) with derivations in `docs/coaster/`. Values and their status:

| Constant | Value | Status |
|---|---|---|
| tile edge, height step | 4.0 m, 1.0 m | A1: defaults until `park-world-grid` |
| sample spacing | 0.25 m | design choice; bounds in spec |
| car spacing, car mass, passenger mass (test run) | 2.0 m, 450 kg, 75 kg | A6: plausible tuning |
| g | 9.81 m/s^2 | same as `docs/appendix-parameters.md` |
| rolling coefficient mu | 0.01 | A6: plausible tuning, verify with a hand calculation at apply |
| air density rho | 1.225 kg/m^3 | A8: standard sea-level value, cite a source in the doc at apply |
| CdA | 1.0 m^2 + 0.25 m^2 per extra car | A6 |
| lift speed, servo acceleration | 2.5 m/s, 1.5 m/s^2 | A6 |
| trim brake speed, deceleration | 3.0 m/s, 3.0 m/s^2 | A6 |
| station deceleration, stop point | 2.0 m/s^2, 11.0 m | A6 |
| speed cap | 40 m/s | A6 |
| G limits (warn, hard) | vertical +3.5/+4.5, vertical low -1.0/-2.0, lateral 1.8/3.0 | A7: own values; 4.5 reuses the Octopus `G_max` |
| G window, airtime threshold | 12 steps, 0 g | design choice |
| drop minimum, hysteresis, inversion arc | 3.0 m, 0.5 m, 2.0 m | design choice |
| envelope | 2.0 m wide, -0.5 m to +2.5 m | design choice |
| limits | 256 pieces, 64 height steps, 200 history entries | design choice |

### D20. Tiers and the MVP stop line

| | Tier 1 (MVP, demo-ready) | Tier 2 | Tier 3 (stretch) |
|---|---|---|---|
| Pieces | station, straight, flat curves r2 and r4, slopes of class +-1 and +-2 with parabolic pitch transitions, lift, trim brake (43 types) | banking, steep classes 4 and 8, block brake, helix, middle edits, close-circuit help | loops, corkscrew, zero-g roll, booster, launch, shuttle mode, Bezier segments |
| Rules | closed circuit, one station, lift, collision, terrain, path, limits | span rule, block rules | inversion pieces' design checks |
| Physics | full dynamics, lift/brake/station, stall, rollback | block hold/release, faults, crashes | launch and booster drives |
| Test run | complete, diagnosis, trace, ratings | block checks, throughput | inversion metrics on real pieces |
| Operation | one train, empty cycle, open/close | multi-train, capacity, guest seats, parallel systems | |
| Client | composer, mesh, ghost, editor, replay, ride-along, markers, a11y | look-around, cost, build dialog, delta stream | LOD ribbons |
| Evidence | golden masters, analytic cases, properties, budgets C1-C6, C8, C9 | thread-count independence, C7 scale, separation property | |

**Tier 1 demo script (what "still demos" means):** (1) open `/coaster/editor`; (2) load the starter, watch it compose with rails and supports; (3) press test run, read the numbers, watch the G chart and the replay with the ride-along camera; (4) load `starter-tall-hill`, see the stall marker, the energy deficit in height steps; (5) load `starter-tight-curves`, see the lateral warning on the radius-2 curves; (6) start empty and draw a few pieces with the keyboard only, seeing the ghost; (7) build the starter into the park and see its train loop. Tier 1 therefore includes minimal build/open and a single-train system, with default adapters (`SandboxParkOccupancy`: flat 64 x 64 tile park, no paths; in-memory ride registry; cost port absent) so it works before the park changes' final shapes are known.

**Stop line:** apply can stop after the Tier 1 groups; the hand-over group then opens the pull request for the Tier 1 slice. The Tier 2 and Tier 3 requirements in the specs are tagged; before archive, if they are not implemented, the final task group moves them into a follow-up change so the published specs match the code.

## Dependencies and contracts

**Requires from `ecs-foundation` (change 2), intent:** `ComponentRegistry.Register<T>(name, schemaVersion, replication)`; `ISimulationSystem`, `SystemAccess` (reads, writes, `UpdateAfter`, `UpdateBefore`), `SystemContext` with `Commands` (`EntityCommandBuffer`) and `FixedStepSeconds`; `AddSimulationSystem<T>()`; `World`/`Query`/`Chunk` with `ChunkCount` and `GetChunk(i)`; `EntityId`; the canonical hash, `ReplicationSchema` and `GET /simulation/snapshot`; `SimulationParameters.FixedStepHz` (this module has a contract test that its own step constant equals it, as `ecs-foundation` D8 prescribes); the test harness style and benchmark project. Inline-array component fields assume the C# `[InlineArray]` feature used by `scene-graph` for names; if the ECS forbids it, use fixed-size struct fields.

**Requires from `scene-graph` (change 3):** nothing at code level. Train cars do **not** become scene nodes: car poses are computed on the client from arc length. Station and support geometry may become nodes later; not here.

**Requires from `job-system` (change 4), intent:** `IJobScheduler` parallel-for with explicit grain, the job graph builder with declared sets, the `Inline` oracle, the determinism harness and golden master pattern. (Module naming `Simulation` was reconciled across changes 2 to 5 before apply.)

**Requires from `ride-on-ecs` (change 5):** the gateway pattern (`ISimulationGateway` spawn calls), the event buffer drain for plain integration events, and the engine timings dashboard to which coaster systems appear automatically.

**Requires from `park-world-grid` (change 6), intent, behind `IParkOccupancy` and `ITrackLattice`:** tile size and height step; per cell terrain height; tile kind (path, queue path, ride footprint, empty); owned land test; reserve and release of cells for a built coaster (placement command); entrance and exit tiles for a ride's queue and exit paths (`StationAccess`: the tile beside the platform start and the tile beside the platform end, on the rider's left). Default adapter `SandboxParkOccupancy`.

**Requires from `ride-catalog` (change 7), intent, behind `ICoasterRideRegistrar`:** register a ride instance of category roller coaster referencing the design hash; publish `CoasterRideStats` (excitement, intensity, nausea in hundredths, reliability inputs, capacity, lap time, status); receive open/close and fault commands; the catalogue's coaster ride type id (`coaster.steel-basic`, assumed) and its allowed piece set. Default adapter: in-memory registry.

**Requires from `guests-and-needs` (change 8), intent:** a system that, in the `Loading` phase, writes guest ids into `TrainSeats`, raises `Occupied` and `PayloadKg` together, and in `Unloading` clears them; ordered before `coaster.commands`. Until it exists a test double writes the buffer.

**Requires from `park-economy` (change 9), intent, behind `IConstructionCostEstimator`:** a price for a piece list and a charge on build. Absent port means no estimate.

**Provides to later changes and the client:**
- `Coaster.Abstractions`: `CoasterRideStats`, ports (`IParkOccupancy`, `ITrackLattice`, `ICoasterRideRegistrar`, `IConstructionCostEstimator`), DTOs under `DataTransferObjects/<Feature>/` (`CreateDesign`, `AppendPiece`, `TestRun`, `Catalogue`, `RideTrack`, and so on), incident event types.
- Components `CoasterRide`, `CoasterTrains`, `CoasterBlocks`, `TrainState`, `TrainSeats`; systems named above.
- HTTP: the 13 routes of spec `coaster-state-sync` plus `POST /coaster/rides/{rideId}/open|close` and `GET /coaster/rides/{rideId}/track`.
- Files: `starter-coaster.v1.json`, `starter-tall-hill.v1.json`, `starter-tight-curves.v1.json` under the module's `Fixtures/` and embedded in the test project, copies of the numbers in the Angular specs.
- Docs: `docs/coaster/01-track-model.md`, `02-train-dynamics.md`, `03-metrics-and-ratings.md`, `04-editor-and-sync.md`.
- To `asset-pipeline`: stable names for procedural parts (`coaster.rail`, `coaster.tie`, `coaster.column`, `coaster.car`) as glTF replacement points.
- Metrics listed in D18.

## Provenance and legal

OpenRCT2 is GPL-3.0 and Roller Coaster Tycoon is a trademark. This change takes **ideas at the level of facts and game rules** from `docs/research/rct-gameplay.md`, never code, tables or assets, and is written in its own words:

- Used as background: that coaster stats come from a simulated test run (VERIFIED, research section 6.2), that a coaster is unrated until a test run completes (VERIFIED, section 7.3), that the track is a sequence of typed pieces with per-type piece sets (VERIFIED, section 7.3), that G limits affect whether guests accept the ride (VERIFIED, section 7.2), and that rating units are hundredths (VERIFIED, section 0.1) which we keep at the catalogue boundary.
- **Not used:** the RCT vehicle model (acceleration lookup by pitch, fixed-point drag), the per-piece G-factor tables (UNKNOWN in the research and not read), the sub-rating coefficient tables, the intensity penalty bounds, track placement rules (UNKNOWN), the 40 Hz tick, any piece list or naming. Our lattice, piece set, physics, G limits and rating function are original and are validated by closed-form physics, not by agreement with another game.
- Where a number in this change coincides with a number in the research (for example a lateral hard limit in the 3 to 4 g range), the source of ours is stated in D19 (own value, or the repo's Octopus `G_max`).
- No product name, logo or asset of the other game appears in code, UI text, piece names, fixtures or docs. Piece ids are engineering terms (`curve`, `pitch`, `slope`). The docs say "inspired by classic park-building games" at most.
- Contributor rule (in `docs/coaster/01-track-model.md`): do not copy from OpenRCT2 while implementing; consult only the research document.
- A provenance review task checks the parameter file and docs against these rules before the pull request.

## Risks / Trade-offs

- **Contracts to unwritten changes (6 to 9).** [High likelihood of name drift] -> all contact is behind five ports with default adapters, a contract-reconciliation task at the start of apply, and Tier 1 works on the sandbox adapters.
- **What lives in `Simulation.Abstractions`.** [`ecs-foundation` Risks: likely most ECS surface] -> the architecture test forbids ECS types in `Coaster.Abstractions`, not in the references; if the engine types are in the module project only, the exception recorded in a later ADR (named by `ecs-foundation`) is needed; flagged in the PR if so.
- **Trigonometry across platforms.** [`Math.Sin/Cos` are used while generating arcs and helices; their last bit may differ between CPUs/runtimes] -> same-machine bit identity holds; golden records are exact on the reference platform and relative 1e-9 elsewhere; dynamics tables are plain doubles; a scan test keeps `Math` calls out of the dynamics. Cross-machine bit identity is a stretch goal per ADR game-0006.
- **Splitting error of the integrator** -> bounded by the closed-form tests; documented in `docs/coaster/02-train-dynamics.md` with the derivation; a smaller effective step (sub-stepping) is possible per train without changing the interface.
- **Lead-car zone rule artefacts** (A3): a long train on a short brake sees the front stop first -> validation lengths (lift: train + 2 m); the Tier 2 block rules add `BlockTooShort`.
- **Car-averaged potential** is approximate between samples (2 mm bound) -> tested bound; documented.
- **Tuning constants are plausible, not sourced** (A5, A6, A7) -> pinned by golden records; the apply run records the resulting starter metrics in `docs/coaster/03`; the expectations in the specs for the three fixtures (lap 35 to 60 s, 11.5 to 14.9 m/s, tall-hill stall, tight-curve warning) are **expectations to confirm at apply**: only the 14.9 m/s upper bound and the hard-limit bound are derived (energy), the others are assumed. If the run disagrees, the spec scenario and the fixture are adjusted together and the change noted in `Decisions` of the PR.
- **Memory.** 500 coasters of 300 m with hot tables only is about 15 MB; cold tables LRU-capped; the structure endpoint builds cold tables on demand.
- **Client/server composition drift** -> shared fixture; total length cross-check on every structure load; version mismatch refetch.
- **Accessibility of a 3D editor.** [A canvas is inherently visual] -> the Pieces table is the full text equivalent, the anchor is a numeric form, marker labels are in the DOM, ride-along has comfort settings; remaining gap: selecting a *location* in the park by keyboard is the park editor's problem (change 6), not this one.
- **Fixed 75 kg in ratings** while live guests have real weights (`Queue.Person` is 30 to 150 kg) -> ratings are defined for a standard full load; live motion uses the real payload (Tier 2); the difference is small because only drag depends on mass.
- **Size (XL).** [Overrun] -> the Tier 1 stop line; tasks are small and ordered; the Tier 2 and 3 groups are independent of each other.
- **Tests that depend on shipped fixtures' behaviour** -> golden records generated by the reference implementation at apply, reviewed by hand against the analytic brackets.
- **Third-party libraries (all candidates, none asserted).** None is required: SHA-256 and JSON are BCL; the benchmark library is the one `ecs-foundation` chose; the chart is custom SVG. Candidates considered and their currency checks (not deprecated in the pinned version, official docs name no different default, upstream not archived and released within 12 months, to be verified at apply, no version asserted here): BenchmarkDotNet (already decided by change 2), Chart.js via the PrimeNG chart wrapper (only if the custom chart is replaced), FsCheck (rejected for the hand-rolled seeded harness, as in `ecs-foundation` D12).
- **AI tooling sync.** Adding the `Coaster` row to `CLAUDE.md` requires mirroring `.github/copilot-instructions.md` and running the `ai-tooling-sync` check; this is a task.

## Migration Plan

Purely additive: new projects, new routes, new components registered by name. Feature flag `Coaster:Enabled` (default true in development, false in production until reviewed) controls `AddCoasterModule()` endpoint mapping and system registration. No data migration, no persisted state, no wire format change to existing endpoints. Rollback is reverting the change; saved park files that embed coaster designs would then fail to load the coaster section, which the park save change must tolerate (note for that change). Apply runs in a worktree on a `feature/coaster-editor` branch with conventional commits and a pull request that the agent does not merge.

## Open Questions

- Should the specific-force helper be lifted into `Shared/` so `DigitalTwin` and `Coaster` use one implementation? Deferred: it needs a decision on `Shared` scope (it must not hold module logic) and would touch the Octopus code, which this change must not. The numeric ties in D6 keep them consistent meanwhile.
- Default train configuration (4 cars of 2 seats) and the 75 kg test load: deferrable, one constant each.
- Chart: custom SVG (chosen) versus the PrimeNG chart wrapper: revisit if more charts appear in the app.
- Should the hard vertical G limit track the Octopus `G_max` automatically (one configuration key) or stay a separate constant? Assumed: separate constant with the same default.
- Whether a `Replace` edit (Tier 2) should be allowed to change the exit frame by shifting the tail rigidly: assumed no; keeps edits local and the history simple.
