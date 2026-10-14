# BoogaBooster: vision and goal

## Purpose

BoogaBooster grows from a digital twin of one carnival ride into a theme-park
tycoon game in the spirit of RollerCoaster Tycoon, built end to end with AI-assisted
development as the showcase for an AI workshop.

Two halves matter equally, and neither is allowed to be a prop for the other:

1. **A working demo.** The goal is a game that works convincingly on stage, not a
   hardened product. The code stays clean, deterministic and tested where that protects
   the demo; the rest is allowed to be demo-grade. See [`demo-scope.md`](demo-scope.md)
   for exactly what may be relaxed and what may not.
2. **A teaching artefact.** The specs, ADRs, agents, skills, MCP servers and the way
   they drive the build are themselves part of the deliverable. A workshop participant
   should be able to read the repo and see how an AI-driven build is steered.

The ambition is deliberately large: show what a team and a set of agents can build
when specs, architecture decisions and tooling are done properly.

## What we are building

```
Today                      Step 1                     End state
+--------------------+     +--------------------+     +-------------------------------+
| one ride, planar   |     | an Octopus ride:   |     | a park: grid, paths, many     |
| arms spin about a  | --> | arms rise and fall | --> | ride types, coasters, stalls, |
| vertical axis      |     | as the ride spins, |     | toilets, thousands of guests  |
|                    |     | 3D physics         |     | with needs, an economy        |
+--------------------+     +--------------------+     +-------------------------------+
```

- **The Octopus first.** The four main arms rise and fall cyclically while the ride
  turns, driven by a cam profile of the main rotation angle. The hubs keep a vertical
  axis, as on the real ride. It must look realistic: tapered tube arms, a hinge pin, a
  moving hydraulic cylinder.
- **Then a park.** Buildable tile grid and footpaths, a data-driven catalogue of flat
  rides, coasters, food and drink stalls, toilets and amenities, guests who stream
  through with hunger, thirst, toilet need, nausea and money, and an economy of prices,
  upkeep, loans, research and park rating.
- **Coasters you design.** A piece-by-piece track editor with energy-based train
  physics, G-force limits, test runs and computed excitement, intensity and nausea.
- **Real assets.** Models and textures made with Blender and Krita through MCP servers,
  exported as glTF and loaded in the Three.js client.

## Engineering goals

These are the reasons the architecture changes, not decoration:

- **Entity-component-system with data-oriented design** for the simulation core, because
  a park with thousands of guests needs cache-friendly, system-oriented processing that
  one ride never did. Existing modules keep their domain-driven design.
- **A scene graph** for the nested rig (mill, arm, hub, gondola) and for the park, shared
  by id between server and client.
- **A work-stealing job system** with dependency counters and stackless continuations.
  Real fibers are not available in .NET or in the browser, so the principle is kept and
  the stack switching is replaced (see ADR game-0003).
- **Determinism as a hard requirement,** also under parallel execution: the same seed
  gives the same park at any worker count (ADR game-0006).
- **Backend-authoritative simulation** with snapshots and deltas to the Three.js client
  (ADR game-0005, proposed).

## Principles

- **Spec first.** Every step is an OpenSpec change with proposal, specs, design and
  tasks; decisions live in ADRs. Apply works one change at a time, in a worktree and
  branch, with a pull request that the agent never merges.
- **Prerequisites first.** Change 0 installs and verifies the environment (a `doctor`
  check) before anything else runs.
- **Evidence over assertion.** Numbers in specs are targets until measured. Facts about
  RollerCoaster Tycoon carry their source and a VERIFIED, INFERRED or UNKNOWN label
  (`docs/research/rct-gameplay.md`). Library choices pass a currency check at apply time.
- **Every step demonstrable.** Each change ends in something a workshop audience can
  see working; see `docs/roadmap.md`. That demo moment is the acceptance test, and
  `demo-scope.md` says which hardening may be deferred to reach it.

## Non-goals

- Not a commercial clone. RollerCoaster Tycoon is a trademark and OpenRCT2 is GPL-3.0.
  Mechanics are described in our own words with our own numbers, with original names and
  original art. No code or assets are copied.
- Not a multiplayer game or a public service. The backend is authoritative for one park.
- Not a browser-side simulation. The client renders and interacts; heavy simulation
  stays on the server.
- Not finished in one pass. The later changes (park, guests, economy, coaster editor)
  are written ahead of their dependencies and are expected to be adjusted as the earlier
  ones land.

## What done looks like

- The Octopus runs on the ECS and job system with output equivalent to the old engine
  (golden-master tests), and looks right in 3D.
- A park can be built on the grid, with paths, rides, a coaster, stalls and toilets;
  guests arrive, walk, queue, ride, eat, get sick, and leave.
- Money and park rating respond to the player's decisions; a scenario can be won or lost.
- All of it is covered by specs, tests and ADRs, and a newcomer can follow the trail from
  this document to the roadmap, the ADRs and the change that built each part.

## Where to go next

- Apply order and dependencies: [`roadmap.md`](roadmap.md)
- Architecture decisions: [`adr/`](adr/README.md)
- Gameplay research: [`research/rct-gameplay.md`](research/rct-gameplay.md)
- Environment setup: `prerequisites.md` (written by the `workshop-prerequisites` change)
- Physics of the existing ride: [`01-physics-overview.md`](01-physics-overview.md) onward

## Open items for the owner

- Workshop audience, length and date are not recorded in the repo; add them here when
  known, since they decide which demo moments get priority.
- The ADRs are all `Proposed` and need a decision.
