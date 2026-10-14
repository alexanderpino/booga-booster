## Why

`park-world-grid` can place footprints on a map, but it only knows paths, an entrance and marker blocks. A tycoon needs a **catalog**: the list of things a player may build, each described by data (size, cost, capacity, ratings, how it attaches to paths, how it looks) so that adding a ride is writing a file, not changing the engine. The existing Octopus must become one entry of that catalog instead of a special case, and the empty `Controller` module needs a generic ride-operation interface so any catalog ride can be opened, closed, dispatched, stopped and broken down. This also delivers the workshop moment of the roadmap: an agent adds a new ride type by writing a definition, with no code change.

## What Changes

- A new **`Catalog` module** (`Catalog` + `Catalog.Abstractions`) holding **versioned JSON definitions** validated by a JSON Schema and by semantic rules, loaded fail-fast at start-up from embedded defaults plus an optional extra directory. Kinds: flat ride, coaster (type entry only), stall, facility, path addition, staff building.
- A **roster of 20 original definitions**: the Octopus (flagship), six more flat rides (a carousel, a swinging-ship-like ride, a twist-like ride, a ferris-wheel-like ride, a bumper-car-like ride, a haunted-house-like ride), two coaster types that delegate track to `coaster-editor`, six stalls and facilities, three path furnishings and two staff buildings. Names and numbers are our own; nothing is copied from RollerCoaster Tycoon or OpenRCT2 (GPL-3.0), and the trademark is not used.
- **Ride ratings**: excitement, intensity and nausea in hundredths, computed in integer arithmetic from either a **fixed base plus one operating-level knob** (most flat rides) or from the **simulation** (the Octopus: peak G and vertical swing measured over a run), with an intensity penalty and a ride value for pricing. Coaster ratings are delegated and provisional.
- **Ride operation in the `Controller` module**: a generic `IRideController` (open, close, emergency stop, operating level, repair) and an `IRideAdapter` port; the **seven-state lifecycle of `ride-state-machine` reused unchanged**; a ride status overlay (Closed, Open, Broken); automatic dispatch; a scripted-cycle behaviour for rides without physics (timed loading, ramped spin or swing motion, occupancy); a deterministic **breakdown hook** (reliability decay, injected sampler, repair); the existing DigitalTwin Octopus plugged in through an adapter and a placement binder, with its `/ride/*` routes and dashboard unchanged.
- **Integration with the grid**: the catalog is an `IPlaceableDefinitionProvider` (locked and incompatible definitions are refused), spawns ride runtime through a placement hook, supports an **unlock/prerequisite hook** for `park-economy`, and binds **art** by asset-pipeline manifest ids with a procedural placeholder fallback (a generic tinted box sized to the footprint).
- **Angular UI with PrimeNG**: a catalog picker (categories, search, cost, key stats, locked state) feeding the build tool palette, and a ride inspector (status, ratings, reliability, price, value, open or close).
- **Tooling and documentation**: a dependency-free validation script (`npm run catalog:validate`) with the same rules and error codes as the loader, parity tests between both, and a catalog authoring guide so a person or an agent can add a definition correctly.

No existing endpoint, telemetry stream or published spec changes. `park-world-grid` gains no code in this change; it is extended only through its provider and hook contracts.

## Capabilities

### New Capabilities
- `catalog-definition-format`: The versioned JSON definition format, kinds, field rules, validation error codes, loading and versioning.
- `catalog-roster`: The 20 definitions shipped with the game and their required values.
- `catalog-ride-ratings`: Excitement, intensity and nausea models, the intensity penalty, ride value and derived throughput.
- `catalog-ride-operation`: The generic ride controller, lifecycle reuse, ride status, dispatch, scripted rides, the digital-twin adapter, throughput and breakdown hooks.
- `catalog-placement-integration`: Provider, spawning, unlock hook, art binding, read API, and adding definitions without code.
- `catalog-ui`: The Angular catalog picker and ride inspector with accessibility requirements.
- `catalog-authoring-tooling`: The validation script, parity guarantee and the authoring guide.

### Modified Capabilities
<!-- None. The ride-state-machine spec is reused by reference; the DigitalTwin ride behaves exactly as before. -->

## Impact

- **New backend modules**: `src/Catalog/FourDotnet.BoogaBooster.Catalog` and `.Abstractions` (definitions, loader, validator, rating calculators, provider, endpoints), embedded JSON under `Catalog/Definitions/**`, schema under `Catalog/Schema/`; `src/Controller/FourDotnet.BoogaBooster.Controller` and `.Abstractions` get real content (controller, lifecycle, dispatcher, scripted ride systems, breakdown system, endpoints). New test projects `src/Tests/FourDotnet.BoogaBooster.Catalog.Tests` and `src/Tests/FourDotnet.BoogaBooster.Controller.Tests`.
- **Edited backend code (small)**: `DigitalTwin` module (adapter, placement binder, registration), `BoogaBooster.slnx`, the API composition root (module registration calls only), `Simulation` module options if a counter-based sampler is missing.
- **Frontend**: `src/FourDotnet.BoogaBooster.App/src/app/catalog/**` (api client, picker, inspector, placeholder builder), `tools/catalog/validate-catalog.mjs`, an npm script, Vitest and axe-core specs.
- **Docs**: `docs/park/02-ride-catalog.md`, `docs/park/catalog-authoring-guide.md`, updates to `docs/README.md`; `CLAUDE.md` and `.github/copilot-instructions.md` status wording (needs the `ai-tooling-sync` check).
- **Configuration**: `Catalog:ExtraDefinitionsDirectory`, `Controller:Rides:RunEmptyCycles`, `Controller:Rides:DigitalTwinAutoDispatch`.
- **Dependencies**: no new runtime package is planned; anything named in design.md is a candidate subject to the currency check at apply time.
- **Depends on**: `park-world-grid` (provider, hook and funds contracts), `ride-on-ecs` (the ECS Octopus, `ride-state-machine` behaviour on both engines), `ecs-foundation` and `scene-graph` (systems, components, scene nodes), `asset-pipeline` (manifest ids, placeholder registry; optional at apply time). Consumed by `guests-and-needs`, `park-economy`, `coaster-editor`.

> **Quality bar:** this change is for a workshop demo, not a production release. See `docs/demo-scope.md` for what may be deferred and what may not.
