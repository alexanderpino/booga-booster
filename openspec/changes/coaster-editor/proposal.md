## Why

The park roadmap so far gives the player flat rides that are catalogue entries: a base rating plus one knob. The thing that makes a tycoon game a tycoon game is the ride the player designs himself, and the coaster is the clearest case: its excitement, intensity and nausea must come from the track the player drew, not from a table. That is also the best showcase for the engine built in changes 2 to 5: a deterministic 1/120 s physics that runs on ECS and jobs, a G-force model that already exists for the Octopus (`docs/05-forces-and-g-forces.md`), and a 3D client. `docs/research/rct-gameplay.md` (section 15.5) confirms the shape: coaster stats are measured from a simulated test run of the actual track, a ride is unrated until a test run completes, and failures (stall, excessive G) are the player's feedback loop. Now is the time because `park-world-grid` (6) and `ride-catalog` (7) supply the grid and the rating contract this change plugs into.

## What Changes

- Add a new gameplay module **`Coaster`** (`src/Coaster/FourDotnet.BoogaBooster.Coaster` + `.Abstractions`). It owns the coaster design aggregate, track geometry, train physics, ride metrics and the editor's HTTP surface, and plugs its simulation systems into the `Simulation` module (change 2) through the `Simulation.Abstractions` contracts only.
- Add an **original track model** on the park grid: a piece catalogue on an integer lattice (tile edge and height step taken from `park-world-grid`), pieces described by entry and exit frames (position, heading, slope class, bank class). The player builds a coaster **piece by piece** from a palette that only offers pieces that connect to the current end. Three tiers: **Tier 1 (MVP)** straights, flat curves, slopes with parabolic transitions, lift hills, trim brakes and a station; **Tier 2** banking, steep drops, multi-train operation with block sections; **Tier 3 (stretch)** loops and other inversions, launches and shuttle mode.
- Add **connectivity and validity rules** with an actionable report: closed circuit, exactly one station, lift or launch present, clearance and collision against terrain, paths and other track, ending transitions into the station, piece-count and size limits, plus a gap report when the circuit is open.
- Add a **curve representation with arc-length parametrisation**: each piece is an analytic primitive (line, circular arc or helix, parabolic pitch change), resampled at about 0.25 m arc-length spacing into a compiled track (positions, tangents, curvature vectors, rider frames, roll) used by physics, metrics and rendering.
- Add **energy-based train physics at the fixed 1/120 s step**: gravity along the track from an exact potential, rolling resistance and quadratic drag, lift chain speed servo, trim and block brakes, station drive, stall detection and rollback. The integrator is built so mechanical energy cannot increase without a drive zone, which is a property test.
- Add **computed ride metrics** (max and average speed, drops, airtime, vertical and lateral G with the specific-force maths of the Octopus docs, inversions, turns, length, duration) and a **mapping to excitement, intensity and nausea** in hundredths, the unit the `ride-catalog` contract uses.
- Add a **test run mode with failure diagnosis**: a synchronous simulation of one lap that reports where the train stalls (with the energy deficit as extra lift height), where G leaves its comfort and hard limits, and which pieces are responsible, with a G, speed and position trace for the client.
- Add **train operation** (Tier 2): train count and capacity, station load and unload with a seat-buffer handoff to `guests-and-needs`, block sections with block brakes for collision avoidance, and **breakdown and crash hooks** that `ride-catalog` and `park-economy` can drive.
- Add a **versioned, deterministic track file format** (canonical JSON with a content hash, integer-only piece data), migrations, and a **starter coaster** fixture plus two deliberately flawed variants for workshops and tests.
- Add **undo and redo** for the design (operation history, deterministic replay).
- Add **state sync per ADR game-0005**: the track structure is sent once per design version, train state is two numbers per train, so the client animates trains along the same compiled samples.
- Add the **client**: a procedural Three.js track mesh (rails, ties, supports) generated from the compiled samples, a ghost preview of the next piece, a first-person ride-along camera, and the editor UI in PrimeNG with full keyboard operation and an accessible text alternative to the 3D view.
- Add **quality gates**: determinism golden masters (known layouts with lap time and metrics inside tolerance), property tests (energy, mass independence, separation), and performance budgets for many trains on many coasters through the job system.
- Add docs under `docs/coaster/` with the derivations the code is written against. **Legal**: OpenRCT2 is GPL-3.0 and Roller Coaster Tycoon is a trademark; this change specifies its own track model, numbers and names in its own words and copies no code, table or asset (see design, "Provenance and legal").
- No breaking changes. `DigitalTwin`, `Queue`, `Weather` and the published specs are untouched.

## Capabilities

### New Capabilities
- `coaster-track-catalogue`: The data-driven piece catalogue, its lattice geometry, entry and exit frames, tiers and per-ride-type piece sets.
- `coaster-track-builder`: Piece-by-piece construction of a coaster design with a connection-aware palette, preview, undo and redo, design lifecycle and cost estimate hook.
- `coaster-track-validation`: Connectivity and validity rules, severities, the validation report, the open-circuit gap report and the test-run gate.
- `coaster-track-geometry`: The curve representation, arc-length parametrisation, rider frames, compiled track and its cached derived tables.
- `coaster-train-dynamics`: Energy-based 1D train physics at 1/120 s, drive and brake zones, stall and rollback, and the energy invariant.
- `coaster-ride-metrics`: Metric extraction during a run, the G-force maths, and the mapping to excitement, intensity and nausea.
- `coaster-test-run`: The test-run mode, outcomes, failure diagnosis, the trace and the result contract.
- `coaster-operations`: Trains, capacity, dispatch, station dwell and guest handoff, block sections, collision avoidance and fault hooks.
- `coaster-track-format`: The versioned deterministic file format, migrations, content hash and the starter fixtures.
- `coaster-state-sync`: Structure and pose split, train state replication, endpoints and DTOs, per ADR game-0005.
- `coaster-track-rendering`: Procedural Three.js geometry, ghost preview, supports and mesh budgets.
- `coaster-ride-camera`: The first-person ride-along view and its comfort settings.
- `coaster-editor-ui`: The PrimeNG editor, keyboard operation, accessibility and the test-run panel.
- `coaster-determinism-and-performance`: Golden masters, property tests and performance budgets for the coaster simulation.

### Modified Capabilities
<!-- None: no published capability changes its requirements. Contracts with park-world-grid, ride-catalog, guests-and-needs and park-economy are consumed through ports defined here (see design). -->

## Impact

- **New backend module** `src/Coaster/FourDotnet.BoogaBooster.Coaster` and `.Abstractions`, new test project `src/Tests/FourDotnet.BoogaBooster.Coaster.Tests`, benchmarks added to the project created by `ecs-foundation`. `FourDotnet.BoogaBooster.Api` gains `AddCoasterModule()` and `MapCoasterEndpoints()` calls only. `BoogaBooster.slnx` gains the projects.
- **Depends on** `Simulation.Abstractions` (components, systems, gateway, job scheduler, snapshot replication), and on intent-level ports for the park grid (`park-world-grid`), the ride catalogue (`ride-catalog`), guest boarding (`guests-and-needs`) and construction cost (`park-economy`), each behind an interface in `Coaster.Abstractions/Ports/` so the coaster compiles and tests without them.
- **Frontend** `src/FourDotnet.BoogaBooster.App/src/app/coaster/`: pure TypeScript track mirror and mesh builders, an editor store on signals, PrimeNG components, a ride-along camera, Vitest and axe-core specs. No change to the existing dashboard or `ride-visualization.ts`.
- **Docs** `docs/coaster/` (track model, train dynamics, metrics and ratings, editor and sync) and a parameters section that follows the `appendix-parameters.md` convention; constants live in the module's `CoasterParameters.cs`.
- **Configuration** `Coaster:*` options (limits, tier switch for the piece set, test-run timeout).
- **AI tooling**: adding the module to `CLAUDE.md` requires mirroring `.github/copilot-instructions.md` and the `ai-tooling-sync` check.
- **Not affected:** `DigitalTwin`, `Queue`, `Weather`, `Controller`, the Aspire graph, the published specs, and the asset pipeline (rail and car models stay procedural until `asset-pipeline` offers glTF replacements).

> **Quality bar:** this change is for a workshop demo, not a production release. See `docs/demo-scope.md` for what may be deferred and what may not.
