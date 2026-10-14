# Roadmap: BoogaBooster to Octopus ride and park simulation

Date: 2026-10-14. Architecture decisions: [docs/adr/README.md](adr/README.md) (all Proposed until accepted).

## How to apply

`/opsx:apply` handles one change at a time. Apply each change in its own git worktree and branch, with logical conventional commits, and open a pull request. The agent never merges; a human reviews and merges. Apply in the order below, starting with change 0; later changes are specified before earlier ones exist, so contracts in their `design.md` are intent until the earlier change lands.

**Doctor gate.** Change 0 (`workshop-prerequisites`) delivers `node tools/doctor/doctor.mjs`. Every other change starts with a task group `## 0. Preconditions` whose first task is to run `node tools/doctor/doctor.mjs --change <change-name>` and to stop if it exits non-zero. Nothing is applied on a red machine. `node tools/doctor/doctor.mjs --lint-gate` checks that every change listed here carries the group. Install steps live in `docs/prerequisites.md` (written by change 0).

## Dependency graph

```
 0 workshop-prerequisites   (doctor gate; Blender/Krita MCP servers)
        |
        +---------------------------------------------------+
        v                                                   |
 1 octopus-motion ----------------------------------------> 11 asset-pipeline
        |                                                   (any time after 0 and 1)
        v
 2 ecs-foundation
        |
        +------------------+
        v                  v
 3 scene-graph       4 job-system
        |                  |
        +--------+---------+
                 v
        5 ride-on-ecs
                 |
                 v
        6 park-world-grid
                 |
                 v
        7 ride-catalog
                 |
                 v
        8 guests-and-needs
                 |
                 v
        9 park-economy
                 |
                 v
        10 coaster-editor
```

Notes: 0 gates every other change (doctor must be green, per change). 3 and 4 are independent of each other and could be applied in parallel branches, but the recommended order is serial. 5 needs 1, 2, 3, 4 (octopus kinematics migrate onto ECS + scene graph + jobs). 10 builds on 6 and 7 (grid, catalog); it can reuse 9 for costing. 11 only needs 0 (doctor green, `blender`/`krita` MCP servers) and 1 (for the node list to name).

## Changes

| # | Change | Size |
|---|---|---|
| 0 | workshop-prerequisites | L |
| 1 | octopus-motion | L |
| 2 | ecs-foundation | L |
| 3 | scene-graph | M |
| 4 | job-system | XL |
| 5 | ride-on-ecs | L |
| 6 | park-world-grid | M |
| 7 | ride-catalog | M |
| 8 | guests-and-needs | XL |
| 9 | park-economy | L |
| 10 | coaster-editor | XL |
| 11 | asset-pipeline | M |

Sizes are rough (S/M/L/XL), relative to the archived `reverse-ride-direction` change.

### 0 workshop-prerequisites
- Goal: make every prerequisite present and verifiable before any change is applied: a dependency-free, read-only `doctor` script (checks .NET SDK, Node/npm, git, Docker daemon, Dapr runtime, `openspec`, Aspire secrets, optional Blender/Krita/Playwright, every MCP server declared in `.mcp.json`, `ai-tooling-sync`, optional `--deep` build smoke), `docs/prerequisites.md` with per-OS install steps, the Blender/Krita MCP servers (moved here from asset-pipeline), and the `## 0. Preconditions` gate in every change.
- Depends on: nothing.
- Delivers: `tools/doctor/` with tests, `docs/prerequisites.md`, hint catalogue, `blender`/`krita` entries in `.mcp.json`, the doctor gate and its lint.
- Demo moment: on a fresh machine `node tools/doctor/doctor.mjs` turns red lines into exact install steps until it prints `doctor: OK`.
- Size: L.

### 1 octopus-motion
- Goal: the 4 main arms rise and fall cyclically (cam-driven function of main angle, no new motor); hub stays vertical via compensating linkage; physics 2D to 3D (vertical acceleration, G-force Z component, radial reach L cos(theta)); docs 04/05/appendix updated; realistic procedural visuals (tapered truss arms, hinge pin, cylinder).
- Depends on: nothing.
- Delivers: elevation kinematics, 3D physics, updated docs and 3D view, visible Octopus motion.
- Demo moment: the existing ride starts to "breathe" up and down live, with the G-force vertical component appearing on the telemetry chart.
- Size: L.

### 2 ecs-foundation
- Goal: the new Simulation module with a World, entities, SoA archetype storage, queries, system scheduling (single-threaded first), deterministic ordering. See game-0001, game-0002, game-0006.
- Depends on: nothing technically; sequenced after 1.
- Delivers: `Simulation` + `Simulation.Abstractions`, `IWorld`, `ISystem`, benchmarks vs library candidates.
- Demo moment: live benchmark - 10,000 entities updated in a fraction of a millisecond, with the memory layout drawn on screen.
- Size: L.

### 3 scene-graph
- Goal: flat-array transform hierarchy with dirty flags and topological update; constraint nodes. See game-0004.
- Depends on: 2.
- Delivers: Parent/Local/World transform components, transform system, `HubLinkage` constraint.
- Demo moment: drag a mill angle slider and watch the hub stay level while the arm rises, with the node array shown.
- Size: M.

### 4 job-system
- Goal: work-stealing scheduler, dependency counters, stackless coroutines, system graph from read/write sets; thread-count independence harness. See game-0003, game-0006.
- Depends on: 2.
- Delivers: `IJobScheduler`, deque, parallel-for over chunks, determinism test harness; optional TS worker variant spike.
- Demo moment: CPU core utilisation graph climbing as workers increase while the state hash stays identical.
- Size: XL.

### 5 ride-on-ecs
- Goal: migrate Octopus physics and kinematics from the DDD aggregate runtime into ECS systems on the scheduler; DigitalTwin telemetry becomes a projection; boundary per game-0001; binary snapshot spike per game-0005.
- Depends on: 1, 2, 3, 4.
- Delivers: ride as entities/systems, parity tests against the old results, gateway and event drain.
- Demo moment: A/B toggle old vs new engine on screen, same trajectories, new one faster.
- Size: L.

### 6 park-world-grid
- Goal: terrain grid, paths, placement of rides, spatial queries and pathfinding basis.
- Depends on: 5.
- Delivers: grid component storage, placement commands, path network, spatial index.
- Demo moment: build a path network and drop the Octopus in the park by clicking.
- Size: M.

### 7 ride-catalog
- Goal: data-driven ride definitions (models, parameters, costs, capacities), more than the Octopus.
- Depends on: 6.
- Delivers: catalog module/aggregate, definitions loaded into the World, UI picker.
- Demo moment: an agent adds a new ride type by writing a spec, with no code change in the engine.
- Size: M.

### 8 guests-and-needs
- Goal: thousands of guests with needs, queues, path following, ride choice; interest-managed streaming.
- Depends on: 6, 7 (and 5 for streaming).
- Delivers: guest components/systems, queue logic, snapshot/delta stream (game-0005).
- Demo moment: 5,000 guests streaming through the park while the bandwidth meter shows the delta stream.
- Size: XL.

### 9 park-economy
- Goal: tickets, prices, costs, ratings and the ledger as DDD aggregates fed by World events.
- Depends on: 8.
- Delivers: Economy module, event-drain consumers, price UI.
- Demo moment: raise the Octopus ticket price and watch queues shorten and revenue curve change live.
- Size: L.

### 10 coaster-editor
- Goal: in-browser track editor (splines, banking), generated physics along the track, ride tests.
- Depends on: 6, 7 (reuses 9 for cost).
- Delivers: track spline data, editor UI, physics integration for track vehicles.
- Demo moment: draw a coaster, press "test run", watch the G-force trace.
- Size: XL.

### 11 asset-pipeline
- Goal: glTF 2.0 asset conventions, manifest and validator, `AssetLoaderService`, the authoring skill, prompt templates and art direction, driven through the Blender and Krita MCP servers that change 0 provides, to GLTFLoader with a review gate. See game-0007.
- Depends on: 0 (doctor green with `--change asset-pipeline`; `blender`/`krita` MCP servers declared and documented there) and 1 (node names); can be applied any time after 1.
- Delivers: audited `ride-design.blend`, export script, validator in CI, glb-loading visualisation with procedural fallback.
- Demo moment: an agent models and textures a gondola in Blender/Krita and it appears in the running ride.
- Size: M.
