## Context

`park-world-grid` (change 6) places footprints and knows paths, an entrance and marker blocks. It exposes extension points: `IPlaceableDefinitionProvider`, `IPlacementLifecycleHook`, `IParkFunds`, `IParkGridReader`. This change fills the first two with real content and gives the empty `Controller` module its purpose. See `proposal.md` for motivation and `specs/` for observable behaviour.

Facts this design leans on:

- `Controller` is today two `.csproj` files and nothing else (`CLAUDE.md` status table: "intended home for ride control/operations logic"). `DigitalTwin` owns the Octopus: the `Ride` aggregate with the seven-state machine (published spec `openspec/specs/ride-state-machine/spec.md`), `IRideStore`, `RideLoadingCoordinator`, `/ride/*` endpoints and the SSE telemetry. After `ride-on-ecs` the Octopus has an ECS engine behind the same `IRideStore`; after `octopus-motion` telemetry carries `VerticalG` and `TotalG` per gondola.
- Roadmap goal for change 7: "data-driven ride definitions (models, parameters, costs, capacities), more than the Octopus"; demo: "an agent adds a new ride type by writing a spec, with no code change in the engine".
- Research (`docs/research/rct-gameplay.md`, labels honoured): ride type descriptors are data in the studied game (section 12, VERIFIED); flat rides are a base rating plus one operator knob (section 6.4, VERIFIED for the studied ones); ratings are hundredths and should stay hundredths at the catalog boundary (section 17, INFERRED rule of thumb); the intensity penalty and the value formula exist (section 6.1 and 6.5; both marked "first-pass read, re-verify" in section 18, so **we do not reuse their bounds or coefficients**); breakdown mechanics (section 8, VERIFIED) are only used as inspiration for a simpler own model; stall and facility behaviour is described in section 5.4 and 10 (needs hooks) and consumed by change 8 not here. Anything UNKNOWN in the research (hiring cost, exact research list, G-factor tables) is not needed and not assumed.
- Style reference: the archived `reverse-ride-direction` change.

## Goals / Non-Goals

**Goals:**
- A definition format a human or an agent can write correctly from the guide, validated identically by the server and by a script.
- The Octopus as one catalog entry with no loss of existing behaviour.
- One generic ride-operation surface over three behaviours (scripted, digital-twin, coaster-delegated).
- Ratings that are deterministic, integer, explainable and testable with numbers.
- A catalog that works without art (placeholders) and without economy or guests (hooks with safe defaults).

**Non-Goals:**
- No guest behaviour, no queue simulation, no needs mapping (tags only).
- No pricing logic, research tree, or ledger (hooks only).
- No coaster track, editor or test run (type entries and a rating contract only).
- No multi-instance DigitalTwin HTTP API (`/ride/*` stays singular; max one Octopus).
- No hot reload of definitions; no definition editing through HTTP.
- No new art. Placeholder boxes only; real assets come from `asset-pipeline`.
- No mechanics/staff dispatch for repairs (a repair command is the hook; `park-economy` or a staff change automates it).

## Legal and originality

OpenRCT2 is GPL-3.0; "RollerCoaster Tycoon" is a trademark. The catalog is the place where copying is most tempting (ride lists, numbers). Rules for this change:

- Behaviour is specified in our own words and numbers. The research doc lists categories and example values for studying game design; none of its ride base ratings, costs, upkeep values, prices, multipliers or modifier tables is used. Every number in `specs/catalog-roster` and `specs/catalog-ride-ratings` is our own tuning value, chosen so the roster is playable and the examples are checkable.
- Ride and shop names are original (Starlight Carousel, Corsair Swing, Whirlwind Twirl, Skyline Wheel, Bumper Arena, Haunted Manor, Sausage Grill, Fizz Kiosk, Frost Cart, Trinket Shop, Info Kiosk, Restroom). "Octopus" is the project's own ride. Descriptions are written fresh. No third-party sprites, sounds, strings or object data.
- A **denylist** file `docs/park/denylist-names.txt` (task 2.3) lists third-party ride and brand names that tests and the validator reject in names, ids, descriptions and art ids.
- No OpenRCT2 code is read into this repo; the research permalinks are references only. The authoring guide states the rule for contributors and agents.

## Decisions

### D1 - Two modules with a clear split: `Catalog` (data) and `Controller` (operation)

```
Simulation.Abstractions  <-- Catalog.Abstractions <-- Controller.Abstractions
        ^                         ^                         ^
        |                         |                         |
   Catalog (module)  ----->  Catalog.Abstractions     Controller (module) --> Catalog.Abstractions,
                                                                              Simulation.Abstractions,
   DigitalTwin (module) ---> Controller.Abstractions                           DigitalTwin.Abstractions
```

- `Catalog` owns definitions, the loader and validator, rating calculators, the unlock hook, the provider that adapts definitions to `park-world-grid`, and the read endpoints. No runtime state of rides.
- `Controller` owns runtime: ride components and systems, dispatcher, scripted behaviour, breakdown, the generic `IRideController`, `/rides` endpoints, and the `IPlacementLifecycleHook` that creates ride runtime when a ride-kind definition is placed. It reads definitions through `ICatalogReader` (from `Catalog.Abstractions`).
- `DigitalTwin` implements `IRideAdapter` and `IRideRigBinder` (from `Controller.Abstractions`) for the Octopus. `DigitalTwin` references `Controller.Abstractions` only, per the modular-monolith rule.

*Alternative:* one `Catalog` module that also runs rides. Rejected: the roadmap names `Controller` as the home of operations, and data (static, validated at start) and runtime (per tick) change for different reasons. *Alternative:* put the catalog into `Simulation`. Rejected: the catalog is content, not engine; `Simulation` must stay free of game content so `ecs-foundation`'s rule "engine does not know the game" holds.

Reuse of `DigitalTwin.Abstractions.RideState` in `Controller` avoids a duplicate enum with 7 identical members; the cost is that `Controller.Abstractions` references `DigitalTwin.Abstractions`. Revisit if a third module needs the enum (then promote it to `Shared`; a pure move). Recorded in Risks.

### D2 - Definition format: JSON, versioned, one file per definition

JSON because the repo already uses `System.Text.Json`, the browser reads JSON natively, JSON Schema gives editor support, and agents write it reliably. Alternatives: YAML (needs a parser dependency and has implicit-typing traps such as `no` becoming a boolean), C# object initialisers (a code change per ride, defeating the goal), TOML (no schema ecosystem). Money is integer cents and time integer seconds so there is no float in the data except motion parameters (rad/s, rad), which are doubles consumed by a deterministic integrator.

Illustrative file (`starlight-carousel.json`; numbers from the roster spec):

```json
{
  "$schema": "../../Schema/catalog-definition.schema.json",
  "schemaVersion": 1,
  "id": "starlight-carousel",
  "version": 1,
  "compatibleFrom": 1,
  "kind": "flat-ride",
  "category": "gentle",
  "name": "Starlight Carousel",
  "description": "A canopy of painted mirrors turning slowly over a ring of seats.",
  "footprint": { "width": 6, "depth": 6, "rotations": [0, 1, 2, 3], "clearanceLevels": 20 },
  "placement": { "anchor": "ground", "host": "ground", "sceneNode": true, "maxPerPark": null },
  "attach": [
    { "kind": "QueueEntry", "cell": [0, 2], "facing": "West" },
    { "kind": "RideExit", "cell": [0, 3], "facing": "West" }
  ],
  "economy": { "buildCostCents": 1200000, "upkeepPerMinuteCents": 150, "defaultPriceCents": 200 },
  "operation": {
    "behaviour": "scripted-cycle",
    "capacity": 24,
    "loadSecondsMin": 15, "loadSecondsMax": 45,
    "rideSeconds": 75, "unloadSeconds": 15,
    "levels": { "count": 3, "default": 2 },
    "motion": { "kind": "spin", "axis": "y", "maxRateRadPerSec": 0.6,
                "rampUpSeconds": 5, "rampDownSeconds": 6 }
  },
  "ratings": { "model": "fixed-base",
               "base": { "excitement": 95, "intensity": 20, "nausea": 25 },
               "stepPerLevel": { "excitement": 10, "intensity": 5, "nausea": 5 },
               "valueWeights": { "excitement": 40, "intensity": 20, "nausea": 10 } },
  "breakdown": { "baseUnreliability": 12, "kinds": ["safety-cut-out"] },
  "unlock": { "initiallyUnlocked": true, "prerequisites": [] },
  "art": { "assets": [], "placeholder": "catalog-box", "tint": "#c9a45c" }
}
```

The Octopus differs in `operation` (`behaviour: digital-twin`, `twin: { rigKey: "octopus", ... }`), `ratings` (`model: simulation-derived`, `base`, `provisional`, `valueWeights`) and `placement.maxPerPark: 1`. A stall has `shop: { serves, serviceSeconds, concurrent }` instead of `operation`/`ratings`/`breakdown`. The exact required set per kind is in `specs/catalog-definition-format`; the schema encodes it with `if/then` per kind.

Versioning: `schemaVersion` changes only when the format changes incompatibly (then a migration note and a loader that reads both for one release). `version` is the author's counter for the definition; `compatibleFrom` marks the oldest saved version that still loads. The catalog hash (FNV-1a 64 over LF-normalised file bytes in id order) detects any edit; CI-style checks can diff it.

### D3 - Loading: validate everything, fail the start, never half-load

All embedded files plus the optional extra directory are parsed with `System.Text.Json` (`JsonDocument`, not object binding, so unknown properties and wrong types are reported with a path instead of being ignored or throwing). A hand-written validator walks the document and emits `(file, id, path, code, message)` records; all are collected, sorted by file then path, and one `InvalidOperationException` with the report fails start-up. Rationale: a catalog with a silently missing ride is worse than a loud start failure, and an agent iterating on a definition needs every problem in one run. JSON Schema validation libraries are **not** used server-side (candidates exist for .NET; each would need the currency check, and the semantic rules need hand code anyway); the schema file is for editors and the script. The loader and script share fixtures to keep them in step (D8).

Extra definitions cannot override embedded ones, which keeps the roster stable and the hash meaningful. No hot reload: definitions are baked into runtime state (spawned components) and reloading would need a migration story; restart is the contract and is stated in the guide.

### D4 - Ratings: two models, integer arithmetic, our own numbers

Why integer hundredths: deterministic across machines (ADR `docs/adr/game-0006-deterministic-parallel-simulation.md` float rules), exact in tests, and comparable with the research's unit convention (section 17).

**Fixed-base** for rides without physics: `rating = base + (level - 1) * step`. One knob, as the research observes for flat rides (section 6.4, INFERRED design consequence).

**Simulation-derived** for the Octopus, because it is the only ride with real physics and the project's showcase. Inputs are two maxima that the digital twin can measure over one run: the peak total G over all seats (`gPeak`) and the vertical swing (`vSwing`, largest minus smallest vertical G), in hundredths of g. Formulas (all integer, truncating):

```
excess = max(0, gPeak - 100)          // g above the standing 1 g
v      = min(vSwing, 150)             // cap at 1.5 g of swing
E = E0 + excess*60/100  + v*40/100
I = I0 + excess*150/100 + v*30/100
N = N0 + v*120/100      + excess*20/100
```

Meaning: each extra 1.00 g of peak adds 0.60 excitement, 1.50 intensity and 0.20 nausea; each 1.00 g of vertical swing adds 0.40, 0.30 and 1.20 respectively. The shape follows the research's observation that G contributes to all three ratings with intensity weighted most and nausea driven by vertical motion (sections 6.3, VERIFIED for the studied game), but **the coefficients are ours** and are calibrated so the shipped Octopus lands in a sensible range:

| gPeak | vSwing | E | I | N | note |
|---|---|---|---|---|---|
| 100 | 0 | 250 | 220 | 180 | standing still, equals base |
| 190 | 100 | 344 | 385 | 318 | reference example in the spec |
| 300 | 150 | 430 | 565 | 400 | hard run |

The penalty `E *= 3/4` for each of the bounds 900, 1100, 1300 on raw intensity gives the "too intense is less fun" shape without a cliff (research section 6.1 has a similar idea; its bounds are marked first-pass there and are not copied).

The provisional baseline for the Octopus (placeholder 300/320/250) is replaced at apply time by a **measured reference run** (task 4.5): run the deterministic ECS Octopus at default power for one Started period, take `gPeak` and `vSwing` from telemetry, apply the formula, and store the result as the baseline. This makes the number reproducible and teachable ("the baseline is a measurement, here is the command").

Value: `(E*wE + I*wI + N*wN) / 100` in cents with weights 40/20/10 (ours). The research (section 6.5) names a similar weighted-sum idea plus an age factor; the age factor and the duplicate-ride discount are `park-economy`'s because they need time and the park's ride list.

Where the pure calculators live: `Catalog` (`IRatingCalculator` in `Catalog.Abstractions`), because the math is data-model knowledge; `Controller` calls it.

### D5 - Operation: one controller surface, three behaviours, one dispatcher

`IRideController` (CQRS commands and queries under `Controller/Features/`) works on **rides identified by placement serial**. Behind it, each ride has an `IRideAdapter`-shaped runtime:

| Behaviour | State lives in | Lifecycle implementation | Motion |
|---|---|---|---|
| `scripted-cycle` | ECS components in `Controller` | `GenericRideLifecycle` (pure, table-driven, mirrors `ride-state-machine`) | `joint_motion` node angle from `RideMotion` |
| `digital-twin` | the DigitalTwin ride | the existing `Ride` aggregate via `DigitalTwinRideAdapter` | the twin's physics and scene nodes |
| `coaster-delegated` | none yet | none | none; `Open` is refused with `BehaviourUnavailable` until `coaster-editor` registers a behaviour |

`IRideAdapter` (in `Controller.Abstractions`): `State`, `Capacity`, `Occupied`, `IsSafeToStart`, `IsAtRest`, `TryRequestTransition(RideState target, out string? refusal)`, `SetOperatingLevel(int)`, `ReadMotionSample()` and `ResetMotionSample()` (maxima since reset: peak total G, max and min vertical G, max lateral G). A `RideDispatcher` runs against the adapter, so the same dispatch rules serve scripted rides and (when enabled) the twin.

**Status overlay** (Closed, Open, Broken) lives in the Controller, not in the lifecycle: the lifecycle is a physical truth ("is it moving, are restraints locked"), status is an operator and park truth ("may new guests start a cycle"). This keeps `ride-state-machine` untouched, which the brief requires ("reused from the existing ride-state-machine capability").

**Scripted ride tick** (system order within the Controller group, after the grid's command system and before scene-graph propagation): `RideDispatchSystem` (reads status, occupancy, timers; requests transitions) then `RideMotionSystem` (updates rate, angle, writes the `joint_motion` local rotation) then `RideBreakdownSystem` then `RideStatsSystem` (publishes derived ratings changes). Declared read and write sets let the scheduler order them. Timers are integer tick counters (seconds x 120), never wall-clock reads (ADR game-0006 rule 6). The conversion constant `TicksPerSecond` comes from `SimulationParameters.FixedStepHz`.

**Motion maths** is deliberately simple and deterministic. Spin: `rate_k = target * min(1, k / rampUpTicks)` while Started, linear decrease from the current rate over the ramp-down while Stopping, `angle += rate / 120` wrapped to [0, 2 pi). Swing: `angle = envelope * profile(phase)`, `phase = (ticksSinceStart mod periodTicks) / periodTicks`, where `profile` is a parabolic sine approximation using only add, multiply and compare, so no `Math.Sin` is needed. Reason: ADR game-0006 rule 4 says the bit-stability of `System.Math` functions across CPUs is not verified, and a visual ride gains nothing from calling `Sin`. The approximation `y = 4/pi*x - 4/pi^2*x*|x|` (refined with `0.225*(y*|y| - y) + y`) is a well-known public technique with a worst-case error around 0.1 percent; the constants are named in the motion constants class. The Octopus does not use this system.

**Breakdown**: components `RideReliability {Value, ChecksDone}`; check every 600 ticks while Open; `d` from a counter-based generator `CounterRandom.Next(worldSeed, streamId, rideSerial, checkNumber)` (SplitMix64 mixing, public-domain algorithm; use the one provided by `ecs-foundation` or `job-system` if it exists, otherwise add a 10-line internal one, task 5.8). `d = (ulong)((UInt128)x * 10000 >> 64)` avoids modulo bias. Decay `baseUnreliability` per check and threshold `(10000 - reliability) / 4`: with `baseUnreliability` 12 the per-check probability reaches 1 percent after about 34 checks (2.8 minutes) and 5 percent after about 167 checks (14 minutes). These are tuning choices, simpler than the research's age tables (section 8) and independent of its constants. Only one breakdown kind (`safety-cut-out`) behaves; the `kinds` list is validated against a fixed set for future kinds.

**Octopus integration** (`DigitalTwinRideAdapter`, `OctopusRigBinder`): thin classes in `DigitalTwin/Application/` that read the existing store. Placing the Octopus calls `IRideRigBinder.Attach(sceneNodeId)`, which reparents the ride's scene root (scene-graph reparent) under the placement node; removal calls `Detach()` and returns the root to the origin. The `/ride/*` routes address the single ride regardless; `maxPerPark` 1 matches that. Auto-dispatch of the twin is off by default (`Controller:Rides:DigitalTwinAutoDispatch`) because today the operator drives it from the dashboard and loading depends on the Queue module's groups; when on, the same dispatcher requests Loading, Safe, Started, Stopping for the Octopus based on adapter data.

**Shared lifecycle table**: `src/Tests/Shared/ride-lifecycle-transitions.json` (linked into `Controller.Tests` and `DigitalTwin.Tests`) lists every (from, to, guard) pair with accepted or refused; both test projects run it. This is the proof that "reused unchanged" is true without moving DigitalTwin code.

*Alternative:* move the DigitalTwin `Ride` state machine into `Shared/Core` and use it everywhere. Rejected for this change: it touches working, tested code (330 tests) and `ride-on-ecs` is mid-flight in `DigitalTwin`; the duplicate is small (a table) and test-locked. Follow-up recorded in Risks.

### D6 - Placement integration

`CatalogPlaceableProvider : IPlaceableDefinitionProvider` maps each definition to `PlaceableDefinition` (cost, footprint, rotations bitmask, clearance, anchor, host, scene-node flag, max per park, attach points, version, `compatibleFrom`) and consults `IUnlockProvider` (default `AllUnlocked`) to return `DefinitionLocked`. `RideRuntimeHook : IPlacementLifecycleHook` (in `Controller`) creates components on `OnPlaced`, answers `CanRemove`, and cleans up on `OnRemoved`; a sibling hook in `Catalog` is not needed because stalls need no runtime in this change beyond data read through `ICatalogReader` by later changes (`guests-and-needs` looks up `serves` tags by definition id from `GridPlacement`).

Art binding is data plus client logic: `art.assets` are asset-pipeline manifest ids (kebab grammar, same as the manifest `id` rule); `placeholder` is a key in the client's placeholder registry (asset-pipeline D8 behaviour: unknown ids give `fallback` status and `instantiate` returns the placeholder). This change registers `catalog-box`. If `asset-pipeline` is not applied yet, `AssetLoaderService` does not exist, and the catalog client uses the placeholder directly through a small `ArtResolver` interface with a default placeholder-only implementation; when `asset-pipeline` lands, a binding swaps the implementation (task 8.4 states both paths).

Read API: the list endpoint returns the definition plus derived statistics (ratings at default level, value, throughput, locked state). The `ETag` combines the catalog hash and an unlock revision number so changes of unlock state invalidate client caches.

### D7 - UI with PrimeNG

Picker: a filterable grouped listbox (PrimeNG candidate components `Listbox`, `IconField`/`InputText`, `Tag`, `Message`; confirm via the `primeng` MCP server at apply time). Inspector: a side panel with `Button`s, a `Slider` or `SelectButton` for the level, `ProgressBar`-style reliability meter with a text value. Polling at 2 Hz for the inspector is simple and cheap for one ride; when `guests-and-needs` lands the same data can ride the delta stream from `park-world-grid`/`game-0005`. Placeholder rendering plugs into the park renderer's instanced-placeholder mechanism from change 6 (definition id to size and tint) and drives `joint_motion` from the scene snapshots through the `scene-graph` mirror.

### D8 - Tooling: schema, script, parity, guide

Three artefacts, one truth: (1) the JSON Schema for editors; (2) the C# loader as the authority at start-up; (3) the Node script for authors and agents. To keep (2) and (3) from drifting there is a **fixture directory** `src/Tests/FourDotnet.BoogaBooster.Catalog.Tests/Fixtures/` with one `valid/` and one `invalid/<ErrorCode>-<n>.json` file per code plus `expected.json` listing `(file, code, path)` triples. The xUnit test and a `node --test` test both read it. A coverage test fails if an error code has no invalid fixture, and another compares the guide's error table to the code list. The guide's JSON examples carry `<!-- valid -->` or `<!-- invalid: Code -->` markers and a test extracts and validates them, so documentation cannot rot silently. The script is dependency-free like the asset validator (`asset-pipeline` D7) and lives beside it (`src/FourDotnet.BoogaBooster.App/tools/catalog/validate-catalog.mjs`).

### D9 - Third-party candidates

None planned for runtime. Candidates considered and rejected with reason: JSON Schema validators (server-side; semantic rules need code anyway), YAML parsers (format choice), FsCheck (the seeded random harness style of `ecs-foundation` is reused). If apply wants one, it must pass the currency check: not deprecated in the pinned version, official docs name no different default, upstream not archived and released within 12 months. PrimeNG components and Three.js APIs are re-verified against the pinned versions (PrimeNG 22, Three.js 0.169).

## Dependencies and contracts

**Requires from earlier changes (intent; reconcile names at apply):**
- `park-world-grid`: `IPlaceableDefinitionProvider`, `PlaceableDefinition`, `AttachPoint`, `AttachKind`, `Facing`, `IPlacementLifecycleHook` and `PlacementContext`, `IParkGridReader` (attach-point connected state per placement), `BuildReasons` (`DefinitionLocked`, `ObjectBusy`, `MaxPerParkReached`), the rule that definition ids match `^[a-z][a-z0-9-]{0,63}$`, save container carrying (id, version) pairs and the `DefinitionIncompatible` load rule, scene node `park_root` and node naming `<id_with_underscores>_<serial>`, marker blocks for tests, the Angular `ParkRenderer` placeholder mechanism and `BuildStateService`.
- `ride-on-ecs`: the Octopus on the ECS behind `IRideStore`; both engines pass the `ride-state-machine` behaviour; the `RideState` enum; the pure lifecycle policy if it was extracted (reuse is optional, the shared table test is mandatory); `DigitalTwin:Simulation:Engine` flag (the adapter works with any engine through `IRideStore`).
- `octopus-motion`: `VerticalG` and `TotalG` on gondola telemetry for the motion sample.
- `scene-graph`: reparenting, node creation, local transform writes, `SceneMirror`.
- `ecs-foundation`: systems, components, command buffer, tick, seed, the counter-based sampler if present.
- `asset-pipeline` (optional): manifest ids, `AssetLoaderService`, placeholder registry; the `class` enum of its manifest is closed (`ride-arm`, ..., `shop`, `toilet`, `guest`, `track-piece`) and has no class for whole flat rides or staff buildings: add classes there when real art for these arrives (Risks).
- Repo: CQRS handler base classes, `BoogaBoosterTelemetry`, `IIntegrationEventPublisher`.

**Provides to later changes (intent):**
- `Catalog.Abstractions`: `ICatalogReader` (`TryGet(id)`, `All`), `CatalogDefinitionDto` family under `DataTransferObjects/Catalog/`, `IUnlockProvider` and `UnlockStatus`, `IRatingCalculator`, `RideRatings`, `RatingSource`, `CatalogHash`.
- `Controller.Abstractions`: `IRideController`, `IRideAdapter`, `RideMotionSample`, `IRideRigBinder`, `IRideBoardingPort` (`Board(count)`, `Occupied`, `Capacity`) for `guests-and-needs`, `IRidePriceSetter` for `park-economy`, `RideStatus`, `RideOperationResponse`, reason codes, integration events `ride-broke-down` and `ride-repaired`.
- HTTP: `/catalog/definitions`, `/rides`.
- To `guests-and-needs`: definitions' `serves` tags, shop `serviceSeconds`, `concurrent`, ride ratings and prices for ride choice, `IRideBoardingPort`, ride status and queue attach information.
- To `park-economy`: `IUnlockProvider` implementation point, `IRidePriceSetter`, upkeep and build costs in data, breakdown events, ride value.
- To `coaster-editor`: `coaster` kind, `track.pieceSet`, `coaster-delegated` behaviour registration point, the delegated rating contract (`RideRatings` marked measured), track piece cost fields.
- To `asset-pipeline`: the list of art ids and the `catalog-box` placeholder key.

## Risks / Trade-offs

- **Contracts from `park-world-grid` and `ride-on-ecs` are intent** → task 1.1 reconciles names first and records deviations; specs describe behaviour only.
- **Duplicated lifecycle table** (generic lifecycle versus DigitalTwin `Ride`) → shared JSON test table in both test projects; follow-up change to promote one implementation into `Shared`.
- **`Controller.Abstractions` references `DigitalTwin.Abstractions` for `RideState`** → acceptable now; promote to `Shared` if a third consumer appears.
- **Octopus footprint, clearance and baseline ratings depend on geometry and a measured run** → tasks 3.2 and 4.5 compute them from `RideParameters` and a reference run; table values are placeholders until then, recorded in `docs/park/02-ride-catalog.md`.
- **One DigitalTwin instance** → the `/ride/*` API and `IRideStore` are singular; `maxPerPark` 1 enforces it. Multi-instance is a follow-up (`/rides/{id}` already exists for the generic surface).
- **Auto-dispatch of the twin off by default** → in park mode the Octopus needs an operator, which is faithful to today but weaker for the demo; flag documented, turning it on is a configuration change.
- **Ratings from a single run can be gamed by one extreme run** → ratings update only at the end of a complete run and use maxima; a smoothing rule (average the last 3 runs) is a possible follow-up (open question below).
- **Asset manifest class list is closed** → asset-pipeline must add classes (for example `flat-ride`, `staff-building`) before real art for these definitions can be validated; until then the placeholder is used. Flagged to the user.
- **Hand-written validator in two languages** → parity fixtures and the code-coverage test; any new rule starts with a fixture.
- **Currency of tuning numbers** (costs, prices, upkeep) → all `TUNING`, in data, rebalanced in `park-economy`.
- **Scene node `joint_motion` for placeholders is not an asset joint** → it follows the asset conventions (`joint_` prefix, snake_case) so a real asset can declare the same joint later.
- **Open question (deferrable):** average ratings over the last three runs instead of the last run? Changes only the stats system and one scenario; default is the last run.
- **Open question (deferrable):** should closing a Loading ride with riders aboard dispatch at once rather than after the minimum load time? Current rule is the minimum load time; changes timing numbers only.
