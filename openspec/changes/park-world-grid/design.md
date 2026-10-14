## Context

After `ride-on-ecs` the Octopus lives in the `Simulation` module's World, but there is no map. This change adds the map and the way to edit it. Constraints that shape the approach:

- **Authority** (ADR `docs/adr/game-0005-simulation-authority-and-state-sync.md`, Proposed): the backend owns the truth. The browser sends commands and renders state. Commands are acknowledged by the next state change, not predicted.
- **ECS and scene graph** (ADRs `docs/adr/game-0001-ecs-for-simulation-core.md`, `game-0002-data-oriented-component-storage.md`, `game-0004-scene-graph-transform-hierarchy.md`): per-tick runtime state is components; the hierarchy is the scene graph; no ECS type may appear in another module's `.Abstractions`.
- **Determinism** (ADR `docs/adr/game-0006-deterministic-parallel-simulation.md`): fixed 1/120 s step, ordered commands, no wall-clock reads in systems, replay from an input log.
- **Repo rules**: modular monolith (module + `.Abstractions`), `Features/<Feature>/` command or query plus handler with `ExecuteAsync`, endpoints without logic, DTOs under `Abstractions/DataTransferObjects/<Feature>/`, no public fields, no magic numbers, xUnit v3 + Moq + Bogus (no FluentAssertions), 80 percent line coverage, Angular 22 zoneless/signals/OnPush, PrimeNG for new UI, Vitest + axe-core.
- **Research**: `docs/research/rct-gameplay.md` describes how RollerCoaster Tycoon 1/2 behaves, with VERIFIED, INFERRED and UNKNOWN labels. Its path rules are mostly UNKNOWN or secondary (section 10: "Whole-path validity rules (slopes, connection edges): UNKNOWN"; section 18 lists track and path placement validity as not read). This design therefore **specifies our own rules**, and records which research statements it leans on.

See `proposal.md` for the motivation and `specs/` for the observable behaviour. Names of ECS and scene-graph types below are **intent** taken from the finished designs of `ecs-foundation` and `scene-graph`; the apply step reconciles spellings (task 1.1).

## Goals / Non-Goals

**Goals:**
- A small, exact, testable grid model whose rules fit on one page of docs and every number is a named constant with a derivation.
- Placement that is deterministic, atomic, undoable and explains refusals with stable reason codes.
- Contracts that let `ride-catalog`, `guests-and-needs`, `park-economy` and `coaster-editor` plug in without editing this change's code (providers, hooks, readers).
- Cheap incremental updates: one placement touches a handful of tiles and one delta.
- A build UI that works with keyboard and screen reader from the start.

**Non-Goals:**
- No land purchase or ownership rectangles (research section 15.1 mentions ownership; deferred, the whole map is buildable).
- No water, no scenery items beyond path additions, no tile-level decorations.
- No guests, no pathfinding agents (only the path graph and distance fields they will use), no queue behaviour.
- No coaster track pieces; only the vertical-span and `free` anchor machinery they will need.
- No multi-user permissions or concurrent editors with conflict merging (commands are totally ordered by arrival).
- No client-side prediction of build results (ADR game-0005 option C is deferred).
- No touch input, no mobile layout.

## Legal and originality

OpenRCT2 is licensed GPL-3.0 and "RollerCoaster Tycoon" is a trademark of its owner. This project is a different product that studies the genre. Therefore:

- Behaviour is specified in our own words and our own numbers. The research document is used to understand which rules matter, never as a source of code, tables, art, sounds, names or strings.
- Tile size (2.0 m), height step (0.25 m), terrain range (0 to 63 levels), ramp grade (12.5 percent), chunk size, costs, undo depth and every limit in this change are our own choices, derived below. None is copied from OpenRCT2 data.
- Names for rides and shops (see `ride-catalog`) are original; the word "RollerCoaster Tycoon", its logos and ride names are not used in code, UI, docs outside the research note, or tests.
- No OpenRCT2 source file is copied, translated or pasted. Permalinks in `docs/research/rct-gameplay.md` are read-only references. If a future contributor wants to port a formula, they must re-derive it from the rule description and document the derivation.
- `docs/park/legal-notes.md` (task 11.2) restates this and gives the checklist for reviewers.

## Decisions

### D1 - Where it lives: `Simulation/Park/`, hosts a grid resource, not an entity per tile

The park code lives in the `Simulation` module under `Park/` (sub-folders `Terrain/`, `Paths/`, `Commands/`, `Persistence/`, `Sync/`), with contracts and DTOs in `Simulation.Abstractions`. Reason: `ecs-foundation` D1 already reserved the "ParkWorld grid concept" for this change inside the same module, the grid must share the World's tick, hash and snapshot, and a separate module would force `Abstractions` ping-pong for a tightly coupled model.

The grid is a **world-owned resource** (plain class with dense arrays) plus **entities for placed objects**. A tile is not an entity: 16 384 tiles x several components would flood the archetype tables for data that is read by index. The dense arrays (`vertexLevel[(W+1)*(H+1)]`, `tileKind[W*H]`, `pathEdges[W*H]`, `attachBits[W*H]`, `slopeDir[W*H]`, `additionEntity[W*H]`, `spanCount[W*H]` and `spans[W*H*4]`) are structure-of-arrays by design (ADR game-0002). Objects are entities with `GridPlacement` so they get ids, generations, the canonical hash, save/restore and scene nodes for free.

*Alternative:* one entity per tile with `Tile` and `Height` components. Rejected: 16 384 entities that never move; every query would iterate them; spatial lookups would still need an index. *Alternative:* everything in the grid arrays, objects too. Rejected: objects need identity, lifecycle hooks and scene nodes.

If `ecs-foundation` did not land a "world resource" concept, the grid is held by a singleton service registered next to the World and serialised by the park container (D12); the World's hash is then combined with the grid hash by the park (task 1.1 decides, nothing else changes).

### D2 - Units and grid numbers, derived

| Constant | Value | Derivation |
|---|---|---|
| `TileEdgeMeters` | 2.0 | Two adult shoulders pass on a 2 m path (adult shoulder width about 0.5 m, general knowledge); a 6 m Octopus arm is exactly 3 tiles; all ride dimensions in `docs/appendix-parameters.md` are multiples of 0.5 m. |
| `HeightStepMeters` | 0.25 | One ramp step per tile then gives a grade of 0.25 / 2.0 = 12.5 percent (about 7.1 degrees), steeper than an accessible ramp (1:12 = 8.3 percent, general knowledge) but gentle enough for a theme park; finer steps would multiply levels without changing gameplay. |
| `MinTerrainLevel`, `MaxTerrainLevel` | 0, 63 | 64 levels fit 6 bits; 63 x 0.25 = 15.75 m is above the tallest ride clearance in `ride-catalog` (about 14 m) plus margin. |
| `DefaultTerrainLevel` | 8 | 2.0 m above level 0 so the terrain can be lowered by 8 levels (2 m) for cuttings without leaving the range. |
| `DefaultMapTiles` | 128 | 256 m square. The Octopus footprint of 10 x 10 tiles (20 m) is under 4 percent of the width; 16 384 tiles keep the keyframe small (D10); the quantisation in ADR game-0005 (uint16 over a park extent) has 1 km of headroom. |
| `ChunkTiles` | 16 | A 16 x 16 chunk is 256 tiles = 512 triangles for terrain, a cheap rebuild unit (about 0.4 ms estimated, to be measured) and 64 chunks for the default map keep draw calls low. Map sides are multiples of 16. |
| `PathClearanceLevels` | 12 | 3.0 m headroom: a standing adult plus a raised arm (about 2.2 m, general knowledge) plus margin; gives bridges and coaster track a defined minimum gap. |
| `MaxSpansPerTile` | 4 | Ground object, path above a cutting, two overhead layers; a fixed bound keeps the span table dense. |
| `TerrainCostPerVertexLevelCents` | 150 | Tuning, own number; see D9. |
| `MaxCascadeVertices` | 4096 | 64 x 64 vertices, a bound on the work of one edit (well under 1 ms) and on the delta size. |
| `UndoDepth` | 50 | Enough for a build session step; bound on memory (each entry is small). |
| `MaxBatchItems` | 256 | One dragged path across the whole 256-tile map side. |
| `DeltaRingSize` | 256 | At most 256 commands of lag are catchable; beyond that a keyframe (about 20 to 150 KB) is cheaper than keeping history. |
| `MaxStreamQueue` | 64 | A slow client must not hold memory; 64 deltas at under 10 KB each is under 640 KB. |

All live in `ParkTuning` (one static class of `const` values in `Simulation/Park/ParkTuning.cs`), with the table above copied into `docs/park/01-park-world-grid.md`. Cost values are marked `TUNING` and are expected to move in `park-economy`.

### D3 - Vertex terrain with the smoothness rule

Heights are stored per **vertex**, not per tile. Adjacent tiles then share edges automatically (no cracks), slopes are natural, and the rule "eight-neighbour vertices differ by at most one level" makes every tile fall into one of only three classes (flat, ramp, rough) because a tile's four corners then span at most one level: 16 combinations, of which 2 are flat, 4 are ramps, and 10 are rough (4 single high corners, 4 single low corners, 2 saddles). A tile with two corners at `h` and two at `h+1` on a diagonal is a saddle and rough.

Editing raises or lowers one vertex by one level; the cascade restores the rule with the minimum change. The cascade is a breadth-first relaxation with a fixed neighbour order (N, NE, E, SE, S, SW, W, NW) and a queue seeded in order, so the set of moved vertices is deterministic (it is also order-independent because the relaxation has a unique minimal fixed point; the fixed order fixes the sequence of delta operations). Complexity is linear in the number of vertices moved, bounded by `MaxCascadeVertices`.

Terrain under objects is locked (spec): the validation looks at every tile that touches a vertex moved by the cascade. This is simple, never leaves an object on a changed footing, and avoids silently moving a ride. Players remove, edit, rebuild. *Alternative:* allow the edit and re-check the object; rejected because a ride footprint must stay perfectly flat and the refusal is easier to explain.

*Alternative:* per-tile heights with explicit slope enums (the tile-based approach many games use). Rejected: neighbours can disagree about a shared edge, which needs repair code, and slopes along a path would need a separate model.

### D4 - Footprints, rotation, anchors and facing

A footprint is a rectangle `width x depth` in local cells `(u, v)`, `0 <= u < width`, `0 <= v < depth`. A placement has an anchor tile (where local `(0,0)` lands) and a rotation `r` in 0..3 (quarter turns, right-handed about +Y, which turns local East toward North). The offset rotation is `R^0(u,v) = (u,v)`, `R^1 = (v,-u)`, `R^2 = (-u,-v)`, `R^3 = (-v,u)`; cells are `anchor + R^r(u,v)`. Facings rotate the same way (East becomes North under `R^1`). This choice keeps scene yaw `r x 90` degrees consistent with the grid mapping (+x is East, +y is South, world +Z is South). The scene node origin is the centre of the footprint's bounding box, so art authored with its origin at the footprint centre needs no offset. Shared **test-vector files** (`src/Tests/.../Park/TestVectors/footprint-rotation.json`) are consumed by both the C# tests and the Vitest specs so the browser's ghost and the server's cells can never silently differ.

*Alternative:* anchor at the centre. Rejected: even footprints have no centre tile.

### D5 - Vertical spans for occupancy and clearance

Each footprint tile keeps up to four `(baseLevel, topLevel, entity)` spans. An item's span is `[base, base + clearanceLevels)`. Overlap on any shared tile is a refusal; the reason distinguishes "something is at or under where I want to stand" (`Occupied`) from "something is above me and in my headroom" (`ClearanceBlocked`) by comparing bases. Ground-anchored items take `base = terrain level of the tile` (all corners equal). `free`-anchored items (used by `coaster-editor` later, and exercised in tests now by a test-only definition) carry an explicit base level and must clear the highest corner of every footprint tile.

Why spans now when only ground items ship: clearance and "path under a coaster" are in the brief for this change; adding spans later would change the grid arrays and the keyframe format, a persistence-format change. The cost is one extra array and a few tests. *Alternative:* a single occupant per tile. Rejected for the format-change reason.

Path additions (bench, bin, lamp) are not spans: they live in `additionEntity[tile]`, hosted by the path on that tile.

### D6 - Footpath connectivity is automatic, not hand-edited

Connections follow from geometry: a flat path has four walk edges, a ramp path has its two end edges. Two paths connect when both have the shared edge as a walk edge. No per-edge toggles. Rationale: fewer commands, nothing to undo separately, and one pure function from state to mask (easy to test, easy to keep identical in the client for rendering since the server sends the mask anyway). A ramp's sides are closed so guests never walk sideways across a slope, which matches the model's heights (neighbours across a ramp side have mismatched edge heights).

Queue paths connect like normal paths. The distinction lives in the **views**: `Wander` excludes Queue tiles so strolling guests never enter a queue by accident (research section 15.1 lists "guests only enter queue paths when going to a ride" as an INFERRED path rule from the secondary wiki, so this is our choice, not a verified fact), and `Any` includes them for route-to-ride.

Wide-area rejection, which the secondary Peep AI wiki mentions as guest behaviour, is **not** a placement rule here: UNKNOWN in the verified sources.

### D7 - Attach points, entrance and the connectivity report

An attach point is data on a definition: kind, local cell, local facing. `QueueEntry` accepts only a Queue path, the others only a Normal path. The path tile stores attach bits in a second mask so BFS over path edges stays clean and the renderer can open a gap in the kerb. The park entrance is a built-in definition with a gate attach point and the boundary rule; its spawn tile is data for `guests-and-needs`.

The connectivity report is a plain read: a BFS over the Any view from the gate's path tile, comparing reachable tiles with the attach-point tiles of every object. It is what `ride-catalog` uses to refuse opening a ride and what the UI shows as warnings.

### D8 - Distance fields instead of per-guest pathfinding

Change 8 will have thousands of guests choosing goals. Per-guest A* would repeat work; the research notes a node budget and junction limit for the original guest search (section 15.1, secondary source). A **distance field** per goal class (breadth-first from all matching attach tiles) answers "nearest facility of kind K" and "next step" in O(1) per guest once built, recomputed lazily when the path graph version changes. Field construction is deterministic (fixed neighbour order, integer distances). Distances are tile steps, not metres (uniform edge cost; a ramp counts as one step, which understates a ramp by under 2 percent of length, accepted).

*Alternative:* A* per guest with budgets. Deferred: it can be added on top using the same graph; the field covers the common queries and is the parallel-friendly, cache-friendly choice.

### D9 - Commands: CQRS handlers enqueue, a system applies at the tick start

Endpoints parse into commands and dispatch to `ICommandHandler<T>` implementations (`PlaceObjectCommandHandler` and so on, `Features/ParkPlaceObject/` ...) deriving from `CommandHandler<T>`. A handler does **not** touch the World. It stamps the request with the next arrival sequence, enqueues it in a lock-free-by-design `ParkCommandQueue` (a bounded `Channel`-like structure; candidate `System.Threading.Channels`, part of the BCL, no new package) and awaits a completion source. A `ParkCommandSystem` runs first in every tick, dequeues everything present at that moment in sequence order, validates, applies, records the command in the log, and completes the awaiters after the tick's command phase. This gives:

- one writer (the tick), so no locks around the grid;
- a total order that replays (`ParkCommandLog`: tick, sequence, type, payload);
- acknowledgement semantics matching ADR game-0005 ("acknowledged by the next snapshot").

Validation (`/validate`) is a **query**: it runs on the request thread against a consistent read view (the between-ticks window already used by the snapshot handler in `ecs-foundation` D10, a lock held by the loop during a tick; the query waits at most one tick). It uses the same pure `PlacementValidator` as the command path (same code, different caller), which is the guarantee behind the "Validate then place agree" scenario.

Cost: `Cost = definition.BuildCostCents` for placements, `TerrainCostPerVertexLevelCents x vertexLevelsMoved` for terrain, removal free. The `IParkFunds` hook is `bool TryCharge(long cents, ChargeReason reason)` and `void Refund(long cents, ChargeReason reason)` with a default `UnlimitedParkFunds`. `park-economy` replaces it. A command validates fully, then charges, then applies; if apply cannot fail after validation (all checks are pure on the same state inside the single writer) there is no rollback path to get wrong.

Undo: entries store *inverse intent* (definition id, anchor, rotation, base level, charged amount; vertex before-values for terrain), not entity ids. Undo is executed as a normal command inside the tick (so it is ordered, logged, validated). Entity ids and serials for re-created objects are new (ids are never reissued, ADR game-0002).

### D10 - Persistence and sync formats

**Save container** (`BBPK`): magic, `u16 formatVersion = 1`, width, height, `u32 nextSerial`, terrain levels (one byte each), definition table (count, then UTF-8 id and `u16` placed-with version per entry), then a length-prefixed World blob produced by `ecs-foundation`'s `WorldSerializer`, then `u64 FNV-1a` over all preceding bytes (own 15-line implementation, same as `ecs-foundation` D9, no dependency). The grid arrays other than terrain are **derived**: after the World is restored, the loader walks `GridPlacement` entities in slot order and rebuilds spans, additions, path masks, attach bits and indexes. One source of truth; the same rebuild code is used after delta-free operations. The loader validates before swapping: it builds the complete new World and grid off to the side and swaps them between ticks.

Files are written to `<name>.bbpark.tmp` then moved over the target with `File.Move(overwrite)` semantics (atomic replace on one volume; apply verifies on Windows and Linux). Name pattern `^[a-z0-9][a-z0-9-]{0,63}$`, directory from configuration, size limit 64 MiB read with a bounded stream, so an uploaded or planted file cannot cause path traversal or memory exhaustion.

**Sync**: ADR game-0005 prefers binary snapshots with deltas and says SSE carries text only; the final park transport is decided in a spike there. Grid changes are **low rate** (player actions) and **small**, so: the **keyframe is binary over plain HTTP GET** (no transport question, cacheable by `epoch`+`seq` ETag) and **deltas are JSON text over SSE** (reuses the existing SSE stack and `Last-Event-ID`-free design). The delta model (epoch, seq, baseSeq, ops) is transport-agnostic, so if the ADR's WebSocket channel lands, it carries the same payloads. A delta carries explicit per-tile state rather than asking the client to recompute connectivity, so the browser has no copy of the path rules. Footprint cell expansion for hover highlighting is the only rule the client duplicates (D4 test vectors).

Epoch changes on load; sequence restarts at 0 for the new epoch. Catch-up uses `GET /park/grid/delta?since=`. A cap on concurrent streams and a bounded per-stream queue stop slow or abusive clients from affecting the tick.

Budget arithmetic (targets): empty 128 x 128 keyframe: vertex levels 129 x 129 = 16 641 B raw, tile records run-length encoded to a few bytes when empty, so about 17 KB (target 20 KB). All paths: 16 384 tile records x about 8 B (kind 1, mask 1, attach 1, slope 1, occupant 4) = 131 KB + 16 641 B levels, about 148 KB, hence the 150 KB target (tight on purpose; if exceeded the occupant is dropped for paths and recovered from placements). JSON delta for one path: `{"op":"tile","x":4,"y":4,"k":1,"e":6,"a":0,"s":0,"o":123}` is about 60 B; with 4 neighbour tiles and one object add (about 150 B) that is about 500 B (target 1 KB). 10 x 10 marker: 100 tile ops x about 60 B = 6 KB + object (target 10 KB). All figures are estimates to be measured by task 6.9.

### D11 - Rendering: chunked meshes, instanced props, analytic picking

Terrain and paths are merged geometry per 16 x 16 chunk, rebuilt when dirty, because their shape depends on heights and masks per tile, which InstancedMesh cannot express without custom shaders. Props (placeholders, benches, bins, lamps) are repeated identical meshes and use `InstancedMesh`, one per definition id. Articulated rides arrive through the `SceneMirror` from `scene-graph` and later through the asset loader of `asset-pipeline`; the park renderer does not draw them (no double drawing).

Picking walks the ray with a digital differential analyser over grid cells (cost bounded by the cells crossed, at most `width + height` for any ray) and intersects the two triangles of each tile using the vertex levels. Compared with raycasting 64 terrain chunk meshes, this needs no BVH, allocates nothing, is exact, and is a short, teachable algorithm. Triangle diagonal choice per tile is a pure function of the corner levels (split along the diagonal joining the two lower corners or, for Flat and Ramp tiles, either, as both are planar), used by mesh builder and picker alike (shared function, tested together).

Three.js is pinned at 0.169 in the repo; `InstancedMesh`, `BufferGeometry`, `Raycaster` and `OrthographicCamera` APIs used must be re-verified against the pinned release at apply time (currency check b). No new library is planned; `OrbitControls` is **not** used because the camera needs custom pan/orbit limits and cursor following, which a 100-line controller covers without importing the examples tree (candidate noted, not adopted).

### D12 - Angular structure

`src/app/park/` with `park-api.service.ts` (HTTP + SSE), `grid/grid-model.ts` (typed arrays mirroring the keyframe; applies deltas), `build/build-state.service.ts` (signals: tool, definition, rotation, cursor, dry-run result, history), `render/park-renderer.ts`, `render/grid-picker.ts`, `ui/build-panel.component.ts`, `ui/park-view.component.ts`. PrimeNG components for toolbar, buttons, dialog, messages and tooltip are **candidates**; the apply step consults the `primeng` MCP server and records the chosen components. The keyboard model keeps a roving tab stop for the palette (arrow keys within the toolbar) and one focusable park view with `role="application"` (needed so arrow keys reach the page; the accessible name and description state this) — flagged for a screen-reader smoke test at apply time (task 9.9).

### D13 - Third-party candidates and currency checks

No new runtime package is planned. Candidates that may be proposed during apply, each needing the currency check (not deprecated in the pinned version, official docs name no different default, upstream not archived and released within 12 months) before use: BenchmarkDotNet (already a candidate in `ecs-foundation`; reuse that project's decision), `System.IO.Hashing` (rejected in favour of the own FNV for now), `System.Threading.Channels` (part of the shared framework of the pinned .NET; verify it is not deprecated in the pinned version), Three.js `OrbitControls` (rejected, see D11), PrimeNG components (pinned 22; consult MCP). Versions are chosen at apply time, never asserted here.

## Dependencies and contracts

**Requires from earlier changes (intent; reconcile names at apply):**
- `ecs-foundation`: `World`, `EntityId`, `ComponentRegistry.Register<T>(name, schemaVersion, replication)`, `EntityCommandBuffer` and playback, `IEntityLifecycleObserver`, `ISimulationSystem` / `SystemAccess` / `SystemContext`, `AddSimulationSystem<T>()`, `FixedTickDriver` and the between-ticks window used by snapshot handlers, canonical hash, `WorldSerializer` (restore into a new World with ids preserved), `BoogaBoosterTelemetry` pattern, `SimulationModuleExtensions` / `SimulationEndpoints`, the test project `FourDotnet.BoogaBooster.Simulation.Tests` with `RandomOperationSequence`, the benchmark project.
- `scene-graph`: `SceneNode` and `SceneParent` / `SceneName` / `LocalTransform` components, node creation with unique sibling names, stable node ids, `SceneMirror` (Angular), `GET /simulation/scene/*`, right-handed +Y-up frame. This change creates the `park_root` node.
- `job-system`: nothing required to compile; park systems register as ordinary systems with declared access. The distance-field and chunk-rebuild loops are single-threaded here and can later become parallel-for over rows without API change.
- `ride-on-ecs`: the shared World and the Octopus entities exist; nothing at code level is consumed. (The roadmap demo "drop the Octopus in the park by clicking" is delivered by `ride-catalog` with the real definition; this change demonstrates the same flow with the `marker-10x10` block, see Risks.)
- Repo: `CommandHandler<,>`/`QueryHandler<,>`, `DomainValidationException`, `IIntegrationEventPublisher` and `[TopicName]` (README in `Shared/FourDotnet.BoogaBooster.IntegrationMessages`), `BoogaBoosterTelemetry.Meter`.

**Provides to later changes (names are intent):**
- `Simulation.Abstractions` (`Park/`): `IPlaceableDefinitionProvider` (`TryGet(string id, out PlaceableDefinition definition, out PlacementDenial denial)`; `IEnumerable<PlaceableDefinition> All`), `PlaceableDefinition` (id, version, minimum compatible version, footprint width/depth, allowed rotations bitmask, ground or free anchor, clearance levels, build cost cents, host kind, scene-node flag, max per park, attach points, category tag), `AttachPoint` (kind, cell, facing), `Facing`, `IPlacementLifecycleHook` (`OnPlaced(in PlacementContext, EntityCommandBuffer)`, `CanRemove(in PlacementContext)`, `OnRemoved(...)`), `IParkFunds`, `ChargeReason`, `IParkGridReader` (read-only: dimensions, vertex level, tile kind, slope class, path edges, attach bits, occupant entity at tile, `PathGraphVersion`, `BuildDistanceField(sources, view)`, `WalkingDistance`, connectivity report), request/response DTOs `PlaceRequest`, `BatchRequest`, `RemoveRequest`, `TerrainEditRequest`, `BuildResultResponse`, `BuildPreviewResponse`, `ParkConnectivityResponse`, `GridKeyframe`/`GridDelta` DTOs and reason-code constants.
- Components (registered in `Simulation`): `GridPlacement` (X, Y, Rotation, BaseLevel, DefinitionIndex, Serial), `PathTile` (Kind, Edges, AttachEdges, SlopeDir), `PathAddition` (HostTile), `ParkEntranceTag`.
- Integration events: `park-object-placed`, `park-object-removed`, `park-terrain-changed`.
- HTTP: the endpoints in `specs/park-build-commands`, `specs/park-persistence-and-sync`.
- Angular: `ParkApiService`, `GridModel`, `BuildStateService`, `ParkRenderer`, `GridPicker`, and the reason-message map.
- To `ride-catalog`: provider, hook, attach-point and funds contracts above; the rule that definition ids match `^[a-z][a-z0-9-]{0,63}$`.
- To `guests-and-needs`: `IParkGridReader`, distance fields, spawn tile of the entrance, path views (Wander, Any).
- To `park-economy`: `IParkFunds`, the events, `TUNING` costs.
- To `coaster-editor`: `free` anchor, vertical spans, `ClearanceBlocked`, terrain locks.

## Risks / Trade-offs

- **Roadmap demo shifted** (the roadmap says "build a path network and drop the Octopus in the park by clicking") → the real Octopus definition belongs to `ride-catalog`. This change demos paths, entrance and the `marker-10x10` block; the Octopus drop is shown as soon as change 7 is applied. Flag to the user when applying.
- **Contract names differ from `ecs-foundation` / `scene-graph` as built** → task 1.1 reconciles first and records deviations at the top of `docs/park/01-park-world-grid.md`; the specs describe behaviour, not types, so they survive renames.
- **Resource vs component for the grid** (D1) → decided at task 1.1 by reading the landed `World`; both variants are described; only the registration code differs.
- **Round trip of the World blob and the derived grid rebuild may disagree after an `ecs-foundation` format bump** → the save container carries its own `formatVersion` and a `worldFormatVersion` read from the blob header; a mismatch is `UnsupportedFormatVersion`.
- **JSON deltas are chatty for big placements** → measured by the size tests (D10); if the 10 KB target fails by more than 2x the fallback is run-length tile ops for rectangles, a format-version-compatible addition.
- **ADR game-0005 is Proposed and may pick WebSocket for the park stream** → payloads are transport-agnostic; only the endpoint class changes.
- **Rule-duplication in the client (footprint expansion, triangle split)** → shared test vectors and a shared-function test; everything else (connectivity, validity) is server truth.
- **A single writer queue can stall if a tick overruns** → the command wait is bounded: a request that waits over 2 s returns 503 and the command, if still queued, is dropped by sequence token; the metric `park.build.queue_wait` shows it.
- **Keyboard cursor and `role="application"`** → some screen readers switch off browse mode in the view; mitigated by the live-region announcements and a smoke test; a non-canvas alternative (a table of nearby tiles) is a possible follow-up.
- **Economy tuning numbers are guesses** (path 8.00, queue path 12.00, entrance 5 000.00, terrain 1.50 per vertex-level, all `TUNING`) → isolated in `ParkTuning` and in definition data; `park-economy` rebalances.
- **Terrain lock under objects** may annoy players → by design, and the refusal reason says which object blocks; revisit if playtests complain.
- **Open question (deferrable):** should removal refund part of the build cost? Default 0; `park-economy` decides with the ledger. Changing it later alters costs, not the specs of this change.
- **Open question (deferrable):** exact PrimeNG component set; decided at apply with the MCP server.
