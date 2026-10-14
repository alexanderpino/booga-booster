## Why

After `ride-on-ecs` the Octopus runs on the ECS, but it floats in an empty void: nothing says where it stands, what else may stand next to it, or where guests could walk. A tycoon needs a **place** first: a tile grid with terrain, footpaths, a park entrance, and a safe way for the player to change all of it (build, remove, undo, preview the cost) while the backend stays the single authority (ADR `docs/adr/game-0005-simulation-authority-and-state-sync.md`). Later changes depend on exactly this: `ride-catalog` needs footprints and attach points to place things, `guests-and-needs` needs a path graph and distance fields to walk on, `park-economy` needs priced build commands, `coaster-editor` needs terrain and vertical clearance rules.

## What Changes

- A **park grid** in the `Simulation` module: a rectangular map of square tiles (2.0 m edge) over a **vertex-based terrain** (heights in 0.25 m levels, neighbouring vertices differ by at most one level), mapped to world coordinates (right-handed, +Y up, grid x to world +X, grid y to world +Z) and to the scene graph (`docs/adr/game-0004-scene-graph-transform-hierarchy.md`) so a placed object becomes a scene node with a deterministic name and pose.
- **Footpaths**: normal and queue path tiles that connect automatically along level or ramp edges, a derived **path graph**, **attach points** (queue entry, ride exit, shop front, park gate) that tie objects to paths, a **park entrance** that must sit on the map boundary, a **connectivity report** (is every ride reachable from the entrance), and deterministic **distance fields** (breadth-first, fixed neighbour order) for guest navigation in change 8.
- **Build mode commands through CQRS**: validate (dry run with cost preview and stable reason codes), place, place-batch (path drawing, all or nothing), remove, terrain raise/lower with smoothing cascade, undo and redo (depth 50). Rules: bounds, terrain slope and flatness, occupancy, vertical clearance, path-addition hosting, per-park limits, funds hook. Commands are applied at tick boundaries in a fixed order so placement is **deterministic and replayable**.
- **Extension points for later changes**: `IPlaceableDefinitionProvider` (the catalog supplies definitions; this change ships only `path`, `queue-path`, `park-entrance` and dev-only marker blocks), `IPlacementLifecycleHook` (spawn and tear down ride runtime; veto removal), `IParkFunds` (unlimited until `park-economy`), and read-only `IParkGridReader`.
- **Persistence and sync**: save and load of a park (versioned container, integrity checksum, atomic file replace, size limits, safe names) and **snapshot/delta sync** of grid changes: a binary keyframe over HTTP, JSON deltas over SSE with sequence numbers and an epoch, a ring of recent deltas for catch-up, full resync on loss.
- **Angular build mode with PrimeNG**: tool palette, ghost preview, tile highlight, cost and reason display, undo and redo, full keyboard operation and screen-reader announcements, covered by Vitest and axe-core specs.
- **Three.js rendering and picking**: chunked terrain and path meshes (16 x 16 tiles per chunk, rebuilt only for dirty chunks), instanced props and placeholder boxes, analytic grid ray picking (no mesh raycast on terrain), draw-call and frame budgets.
- **Docs and legal stance**: `docs/park/01-park-world-grid.md` with every derivation, and a clean-room statement: behaviour is specified in our own words with our own numbers; OpenRCT2 is GPL-3.0 and "RollerCoaster Tycoon" is a trademark, so no code, asset, name or table is copied (see `docs/research/rct-gameplay.md` for the rules we studied, and design.md "Legal and originality").

No existing endpoint, telemetry stream or published spec changes. `DigitalTwin`, `Queue`, `Weather` and the current dashboard are untouched.

## Capabilities

### New Capabilities
- `park-tile-grid`: Grid dimensions, coordinate systems, vertex terrain, slope classification, terrain editing and its smoothing cascade, and the mapping of placed objects to ECS entities and scene nodes.
- `park-footpaths`: Normal and queue path tiles, automatic connectivity, attach points, the park entrance, the path graph, connectivity report and distance fields.
- `park-build-commands`: Validate, place, batch, remove, terrain edit, undo and redo with stable reason codes, cost preview, occupancy and clearance rules, deterministic ordering and the extension hooks.
- `park-persistence-and-sync`: Park save and load, grid keyframes and deltas, sequence and epoch rules, loss recovery and wire budgets.
- `park-build-ui`: The Angular build-mode interface: palette, ghost, highlight, feedback, undo and redo, keyboard operation and accessibility.
- `park-grid-rendering`: Three.js terrain, path and prop rendering, grid picking, ghost and highlight visuals, and performance budgets.

### Modified Capabilities
<!-- None. Existing capabilities are not changed. -->

## Impact

- **New backend code**: `src/Simulation/FourDotnet.BoogaBooster.Simulation/Park/**` (grid, terrain, paths, commands, persistence, sync, systems), `Features/Park*/` command and query handlers, `Endpoints/ParkEndpoints.cs`, park DTOs and extension interfaces in `src/Simulation/FourDotnet.BoogaBooster.Simulation.Abstractions/` (`DataTransferObjects/Park/`, `Park/` contracts), events in `src/Shared/FourDotnet.BoogaBooster.IntegrationMessages/Events/Park/`.
- **Edited backend code (small)**: `SimulationModuleExtensions` (park registration), `SimulationEndpoints` or the API host calls `MapParkEndpoints()` through the module, `SimulationModuleOptions` (a `Park` section).
- **Tests**: `src/Tests/FourDotnet.BoogaBooster.Simulation.Tests/Park/**` (xUnit v3 + Moq + Bogus, seeded invariant tests, replay tests), shared test-vector files, benchmarks in `src/Benchmarks/FourDotnet.BoogaBooster.Simulation.Benchmarks`.
- **Frontend**: `src/FourDotnet.BoogaBooster.App/src/app/park/**` (api client, grid model, build state, renderer, picking, build-mode UI) and a lazy route; Vitest and `*.a11y.spec.ts` specs.
- **Docs**: `docs/park/01-park-world-grid.md`, `docs/park/legal-notes.md`, `docs/README.md` index line. `CLAUDE.md` and `.github/copilot-instructions.md` get one status sentence each (needs the `ai-tooling-sync` check).
- **Configuration**: `Simulation:Park` (map size, save directory, size limits, marker blocks flag).
- **Dependencies**: no new runtime package is planned. Anything named in design.md as a candidate must pass the currency check at apply time.
- **Depends on**: `ecs-foundation`, `scene-graph` (and, for the clock and ordering, the executor from `job-system` without any change), `ride-on-ecs` only through the shared World and tick. Consumed by `ride-catalog`, `guests-and-needs`, `park-economy`, `coaster-editor`.

> **Quality bar:** this change is for a workshop demo, not a production release. See `docs/demo-scope.md` for what may be deferred and what may not.
