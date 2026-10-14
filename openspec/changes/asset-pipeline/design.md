## Context

The 3D view is one Angular component, `src/FourDotnet.BoogaBooster.App/src/app/ride-dashboard/visualization/ride-visualization.ts`, on Three.js `^0.169.0`. It builds everything from `BoxGeometry`/`CylinderGeometry` via private `box()`/`cyl()` helpers and a `materials()` palette (`0xe6b826` arm, `0x347ad9` hub, `0x27b374` gondola, `0xcc2e2e` motor, `0x8d9096` tower, `0x47474d` base), nests `Group` pivots (`mainPivot` -> `armHinge` -> `hubPivot` -> `hubArmHinge` -> `gondolaPivot`) and animates them with angles from signed rpm. There is no asset loading, `public/` holds only `favicon.ico` and an unreferenced `3dmodels/boogabooster.fbx` (160 KB), and `designs/3d/` holds `ride-design.blend` plus its tracked `.blend1` backup. No `.gitattributes` exists. No Playwright is installed; tests are Vitest with axe-core. `CLAUDE.md` requires `ai-tooling-sync` after any MCP, skill or instruction change. The `blender`/`krita` MCP servers are provided by change 0 `workshop-prerequisites`, which also owns the `doctor` readiness script.

Decisions already made elsewhere: ADR `docs/adr/game-0007-asset-pipeline-blender-krita-gltf.md` fixes Blender + Krita + glTF 2.0 + `GLTFLoader`; ADR `docs/adr/game-0004-scene-graph-transform-hierarchy.md` defines scene-graph node ids and the transform hierarchy that joint names must map onto. Both are written in parallel; this design states intent and flags conflicts in Risks. See proposal.md for motivation.

## Goals / Non-Goals

**Goals:**
- One documented, validated path from `.blend`/`.kra` to a `.glb` rendering in the 3D view.
- Conventions strict enough that an agent can produce a conforming asset from a written spec and a script can prove it did.
- The pipeline's value proven on the Octopus without changing its behaviour; every failure degrades to today's procedural look.
- Most tasks doable with neither Blender nor Krita installed.

**Non-Goals:**
- Authoring the park content (shops, toilets, guests, track pieces). Only their budgets, LOD plan and manifest classes are specified; the assets arrive with changes 7, 8 and 10.
- Hub drum, hub arms, tower, base and ground stay procedural.
- Compressed delivery (Draco, Meshopt, KTX2/Basis, WebP textures), animation clips and skinned meshes.
- A glTF viewer UI, an in-app asset browser, or running the MCP servers in CI.
- Changing telemetry, physics or any backend module.

## Decisions

### D1 - Repository layout: sources under `designs/`, exports under the Angular app's `public/assets/`

Sources: `designs/3d/<asset-id>/<asset-id>.blend`, `designs/textures/<asset-id>/*.kra`. Exports: `src/FourDotnet.BoogaBooster.App/public/assets/{models,textures}/` plus `public/assets/manifest.json`. Tooling (schema, validator, tests) in `src/FourDotnet.BoogaBooster.App/tools/assets/`. Docs in `docs/asset-pipeline.md` and `docs/art-direction.md`.

Why: `designs/` already exists and holds the first `.blend`, so authoring files stay out of the web bundle; `public/` is copied verbatim by `angular.json` (`"input": "public"`), so a `.glb` there is served at `/assets/models/...` with no build step and no import. Putting sources in `public/` would ship megabytes of `.blend`/`.kra` to every browser. Validator inside the app (not repo root) because `package.json` and Node tooling live there and `npm run assets:validate` is then one command next to `npm test`.

Alternatives: (a) keep `public/3dmodels/` - rejected, the user-specified `assets/models` + `assets/textures` split leaves room for the manifest and for non-model assets, and the FBX there is unused; (b) put exports in `src/app/assets/` and import through the bundler - rejected, hashed filenames break the manifest-driven URL and bundling binaries slows `ng build`; (c) a top-level `assets/` folder - rejected, a second place `angular.json` would need to learn about.

`designs/3d/ride-design.blend` stays where it is (user file, legacy reference). `ride-design.blend1` is removed from the index (`git rm --cached`) and ignored; the working-tree copy is untouched.

### D2 - No git-lfs now; explicit thresholds; binary attributes

All current binaries are small (136 KB `.blend`, 160 KB FBX) and the ride set is four parts, so git-lfs would add clone, CI checkout, worktree (`.claude/worktrees/`) and Copilot-CLI-review complexity for no gain. Budgets in the asset-conventions spec (`.glb` <= 300 KiB ride parts, `public/assets` <= 8 MiB, source file <= 10 MiB, `designs/` <= 100 MiB) bound repository growth: every committed revision of a binary is stored in full, so Blender "Compress File" is turned on and `.kra` is already zip-compressed. `.gitattributes` marks `*.glb *.blend *.kra *.png` as `binary` so git never attempts text merges or diffs. Crossing a source threshold triggers a follow-up change that adopts git-lfs; that change, not this one, would edit CI.

Alternative: adopt git-lfs now - rejected for the reasons above; revisit when the park content lands (change 7 onward).

### D3 - Names are ids; the grammar carries meaning

Node names are lowercase snake_case with role prefixes `root`, `joint_`, `socket_`, `mesh_` (spec `asset-conventions`). The regex `^[a-z][a-z0-9_]*$` is chosen because Three.js `GLTFLoader` sanitises node names through `PropertyBinding.sanitizeNodeName` (whitespace and reserved characters change), and Blender appends `.001` to duplicates; a name that survives neither is not a stable id. Joints are *empties* at the pivot with the DOF about a local axis (axis, kind and range declared in the manifest, not inferred from names or transforms), so the ride joint maps to a scene-graph node id by path: instance node id (ADR game-0004) + `/` + glTF node name. The simulation never reads the glTF; the client binds `getObjectByName('joint_arm_elevation')` on an instance and applies the angle it receives. When the scene-graph change defines its flat parent-index arrays, an asset instance becomes a subtree whose local node ids come from the manifest's `joints` and `sockets` lists in manifest order, so ids are deterministic across clients.

Alternatives: custom glTF `extras` carrying joint metadata - rejected for now (the Blender exporter writes custom properties to `extras`, which is workable, but it makes the contract live inside binaries; the manifest is diffable text and is validated against the file); name-encoded axes (`joint_arm_z`) - rejected, a rename would silently change semantics.

### D4 - Axis and unit conversion is done once, by the exporter

Blender is Z-up, -Y forward; glTF and Three.js are Y-up, +Z forward. The glTF exporter's "+Y Up" option performs the conversion, so the client applies no corrective rotation (a client-side `rotation.x = -PI/2` is a classic source of broken joint axes). The rules "scale 1, transforms applied, joints are empties with zero rotation" make a joint's axis a local axis of its node in both tools. The one non-obvious consequence is recorded in the spec: the glTF `z` axis is Blender `-Y`, so the arm's elevation hinge axis is declared in glTF space. A fixture-based Vitest verifies elevation sign and tip position with a loader-parsed model rather than trusting this paragraph.

### D5 - Texture and material rules favour the cheapest glTF-native path

Power-of-two sizes with per-class caps, sRGB for colour, linear for data, one ORM texture (occlusion R, roughness G, metalness B), embedded in the `.glb`, <= 2 materials per asset (4 for arm and gondola). `GLTFLoader` already assigns sRGB colour space to base-colour and emissive maps, which a test pins. Krita authors PNGs; the ORM packing is done in the Blender export node setup (or a Krita channel layout), whichever the apply step finds workable through the MCP server; the validator checks the result rather than the method. Compressed textures (KTX2) are deferred: they need the Basis transcoder served from `public/` and a `KTX2Loader` bound to the renderer, which adds a loader dependency on `WebGLRenderer` that the loader service deliberately does not have.

### D6 - Triangle budgets, LOD and instancing

Initial LOD0 budgets are in the spec (arm 2,500; hinge 800; cylinder 600; hub 1,500; gondola 3,000; shop 4,000; toilet 1,500; guest 300; track piece 800). They are chosen so the Octopus (4 arms, 4 hinges, 4 cylinders, 16 gondolas: 4x2,500 + 4x800 + 4x600 + 16x3,000 = 63,600 triangles) is far below the budget that a mid-range laptop GPU sustains at 60 fps, leaving room for the park. They are provisional: the numbers are a table in the validator and a table in the docs, changed together, and re-measured at change 8. Guests use three levels (300 / 120 / 2) as separate nodes sharing one material, rendered as one `InstancedMesh` per level switched by distance thresholds (initial 15 m and 40 m, named constants) fed from the ECS position arrays; with 5,000 guests that is three draw calls. Per-instance skeletons are excluded because skinned meshes cannot be instanced in Three.js without custom shader work; walk animation is a vertex-shader bob/sway driven by an instance attribute, designed in change 8. This change only fixes the manifest shape (`lods`) and makes `AssetLoaderService` able to hand back the geometry and material of a named LOD node; the instancing code is change 8's.

`InstancedMesh` and `LOD` are in the pinned Three.js already; no new library.

### D7 - Manifest + schema + a dependency-free validator

The manifest is hand-diffable JSON, with derived fields (`bytes`, `sha256`, `triangles`, `materials`) refreshed by `--write-derived` so humans and agents edit only intent fields. The validator parses the GLB container directly (12-byte header, JSON chunk, BIN chunk) in plain Node, which is enough to read nodes, meshes, accessors, images and image headers, so it needs no npm dependency and runs anywhere Node runs, including a machine without Blender. JSON Schema validation of the manifest itself needs a validator library: **candidate** `ajv` (devDependency, `npm view` + upstream check at apply time); if it fails the currency check the schema is enforced by a small hand-written structural check in the same script and the schema file remains documentation and editor support. The **candidate** Khronos `gltf-validator` MAY be wired as an optional extra check (`--khronos`), also subject to the currency check; it is not required for the gate. Every library named here, and Playwright in D11, must pass the currency check at apply time (not deprecated in the pinned version, official docs name no different default, upstream not archived and released within 12 months); no version is asserted in this document.

Alternatives: validate only in Vitest - rejected, assets are produced by agents outside Angular tests and need a one-line CLI; generate the manifest from the files entirely - rejected, `class`, `joints`, `placeholder` and `provenance` are intent that cannot be derived.

### D8 - `AssetLoaderService`: signals, single flight, graceful fallback

Shape (intent): `status(id)`, `lastError`, `pending`, `progress`, `load(id): Promise<void>`, `instantiate(id): THREE.Object3D`, `retry(id)`, `lodSource(id, level): {geometry, material}`, `dispose()`; collaborators `GLTF_LOADER` (injection token, default `new GLTFLoader()` imported from `three/examples/jsm/loaders/GLTFLoader.js`, the same import style as the existing `OrbitControls` import), `ASSET_FETCH` (token wrapping `fetch` for the manifest), and `PLACEHOLDER_BUILDERS` (a registry from placeholder key to a function returning an `Object3D` with the same joint and socket names as the real asset). Per-id state is one `signal<Record<string, AssetStatus>>` updated immutably so Angular zoneless change detection sees every change. Concurrent loads share one `Promise` stored by id. A `Promise.race` with a timeout bounds a load (named constant `ASSET_LOAD_TIMEOUT_MS`).

Fallback is the central design rule: `instantiate` never throws and never returns null. The placeholders are today's procedural builders, extracted from `ride-visualization.ts` into `src/app/assets/placeholders/` and given the joint/socket node names, which is also what makes the refactor reviewable: with all assets in `fallback`, the existing `ride-visualization.spec.ts` must pass unchanged in meaning. Real assets are verified against the manifest joint and socket list on load; a mismatch is a `joint-missing` fallback, because a model that loads but cannot be animated is worse than a placeholder.

The component calls `instantiate` immediately (placeholder if not ready) and swaps when the status flips to `ready`, mounting under the same joint node so pose is preserved. Loading UI uses PrimeNG (the repo mandates it for new UI): a progress bar bound to `progress()` while `pending() > 0` and a warn message with a Retry button on fallback, built after consulting the `primeng` MCP server. Component names and exact PrimeNG components are chosen at apply from that server, not here.

Alternatives: an RxJS-based service - rejected, the repo is signal-first; Angular `resource()` per asset - viable but adds API-stability questions and a status model that does not cover fallback; `GLTFLoader` inside the component - rejected, untestable and no caching across consumers (guests, shops later).

### D9 - The vertical slice keeps the hub's vertical axis by composition, not by baking

`octopus-motion` (change 1) makes the arm elevate and keeps the hub plane horizontal with a compensating linkage in the visualization. The slice preserves exactly that: the arm asset's `joint_arm_elevation` receives the elevation, the hub pivot is parented to `socket_hub_mount`, and the existing compensation (counter-rotation of the hub pivot by minus the elevation) stays in code. Assets never embed compensation; they are pure geometry plus named pivots. The hydraulic cylinder is a *visual* linkage: its base joint aims at the arm's `socket_cylinder_anchor` and the rod extends by the measured distance, clamped to the declared range. It does not feed physics (change 1 owns the kinematics). Because change 1's exact visualization code is not known when this is written, the first task of that group reads it as it stands at apply time and adapts the mounting points; the specs state behaviour (angles, vertical axis within 1e-6, tip within 1e-4 m) rather than code structure.

### D10 - MCP servers are a prerequisite owned by `workshop-prerequisites`

This change selects, configures and documents no MCP server. Change 0 `workshop-prerequisites` researches the official `blender` and `krita` MCP servers, applies the currency check, adds them to root `.mcp.json`, regenerates the `src/.mcp.json` mirror, performs the `CLAUDE.md` / `.github/copilot-instructions.md` / `README.md` obligations, and records the risk assessment and the smoke-test recipes in `docs/prerequisites.md`. This change only (a) runs `node tools/doctor/doctor.mjs --change asset-pipeline` as its first step and stops when it is red, (b) links to `docs/prerequisites.md` from `docs/asset-pipeline.md`, and (c) has the skill name the servers as declared in `.mcp.json` (`blender`, `krita`). If change 0 recorded that no Krita candidate passed, the Krita-driven tasks stay blocked and the manual Krita path documented there applies. Because the servers can probably execute code in Blender (to be confirmed from their documentation by change 0), the skill prefers structured tools and narrates any script it runs; the risk record lives in `docs/prerequisites.md`.

The skill lives in `.claude/skills/` (not in the plugin: it needs a local Blender, the plugin is for distribution to machines that may not have one). `.github/skills/` holds only OpenSpec-generated twins per `CLAUDE.md`, so no hand mirror is made; the sync check confirms.

### D11 - Visual regression: deterministic snapshot mode + Playwright in a pinned container

Determinism comes from the application, not from retries: `?snapshot=1` freezes the rig pose at named constants, fixes camera and pixel ratio, draws once and sets a ready marker after assets settle. Telemetry and queue/weather requests are intercepted by Playwright with fixed bodies so the test needs no backend, Aspire or Docker beyond the browser container (the exact SSE frame shape is read from `sse-ride-telemetry-source.ts` at apply time). Baselines are generated only in one pinned Playwright container image on Linux with software rasterisation, since Playwright's snapshot names carry the platform and GPU rendering differs; running locally on Windows compares nothing and prints how to run the container. Two baselines: assets ready, and assets blocked (fallback render), which pins the "same look as before" promise. Playwright is a **candidate** (the test package, plus the browser-bearing container image) subject to the currency check. The test is a separate script, `npm run test:visual`, not part of `npm test`, and is **not wired into `.github/workflows/`** by this change: editing the workflow triggers the major-severity review rule and CI browser minutes are a cost decision for the maintainers (see Open Questions).

Alternative: screenshot via the `playwright` MCP server only - useful for an agent's ad-hoc look (and listed in the skill as a verification aid), but it leaves no committed regression guard.

### D12 - Provenance and licence

A closed licence list (`original`, `CC0-1.0`, `CC-BY-4.0`) with mandatory `attribution` for the last, `aiAssisted` plus `generator` for agent-made assets, and a `sources` list for any third-party input. It is cheap to validate and closes the real risk of an agent pulling a model or HDRI from an online library (some MCP servers integrate such libraries) with an unknown licence. The record lives in the manifest so it ships next to the asset and survives file moves. `original` means project-owned under the repository's `LICENSE`.

### D13 - Legacy files

`public/3dmodels/boogabooster.fbx` is removed only after a repository search proves no reference (a pre-check during design found none in `.ts`, `.json` or `.md`; the apply step repeats it). FBX is not a pipeline format and shipping an unused 160 KB file in `public/` contradicts the convention. `designs/3d/ride-design.blend` is left alone.

## Dependencies and contracts

**Requires from earlier changes**
- `octopus-motion` (1): per-arm elevation angle and the hub compensating-linkage behaviour in the visualization, procedural arm/hinge/cylinder/gondola builders as the starting point for placeholders; the hub-vertical-axis invariant. Stated as intent; the apply step reads the code as it exists.
- ADR `docs/adr/game-0004-scene-graph-transform-hierarchy.md`: node id scheme the joint naming maps onto. ADR `docs/adr/game-0007-asset-pipeline-blender-krita-gltf.md`: the format decision this change implements. If either says something different from D3/D4 (for example a different axis convention or id format), the ADR wins and the conflict is fixed here before apply.
- `workshop-prerequisites` (0): `blender` and `krita` entries in root `.mcp.json` (names exactly `blender`, `krita`), the setup/smoke-test recipes and risk record in `docs/prerequisites.md`, and `node tools/doctor/doctor.mjs --change asset-pipeline` exiting 0. Stated as intent; if change 0 recorded a failed currency check for a server, the matching tasks stay blocked.
- Existing infrastructure: Angular 22 zoneless + PrimeNG 22, Vitest + axe-core, Three.js 0.169, `ai-tooling-sync` skill.

**Provides to later changes**
- To `scene-graph` (3): the naming grammar and "instance node id + `/` + glTF node name" addressing; manifest `joints`/`sockets` lists as the source of a deterministic local-id order.
- To `ride-on-ecs` (5): the visualization contract that a ride part is `instantiate(id)` plus named joints; the placeholder registry makes the migration verifiable with and without assets.
- To `ride-catalog` (7), `guests-and-needs` (8), `coaster-editor` (10): asset classes and budgets table; `AssetLoaderService` (`load`, `instantiate`, `lodSource`, `status`, `progress`); the manifest schema and validator as the gate for any new asset; the authoring skill and spec template; the Playwright harness and `snapshot=1` mode for visual guards.
- To CI maintainers: `npm run assets:validate` is cheap and dependency-light and can be added to a workflow later.

## Risks / Trade-offs

- [Blender/Krita MCP server quality or maintenance unknown] -> vetted by change 0 (D10); the pipeline and loader are usable with hand-authored assets, and tasks are tagged so the MCP-independent work lands first.
- [MCP server may execute arbitrary code in Blender or fetch third-party assets] -> risk record lives in change 0's `docs/prerequisites.md`; skill prefers structured tools; provenance rules apply to downloads.
- [Binary history growth in git] -> size budgets, Blender file compression, thresholds that force a git-lfs decision before the repo gets heavy.
- [Axis/sign mistakes between Blender and Three.js] -> declare axes in glTF space in the manifest, assert elevation sign and tip position in a fixture-based test, and a Playwright baseline.
- [The Octopus code from change 1 differs from what this design assumes] -> specs describe behaviour; the first slice task re-reads the code; placeholders keep the app working in the meantime.
- [Provisional triangle budgets are wrong] -> table is a single edit in docs + validator; re-measure at change 8.
- [Screenshot baselines flake across machines] -> one pinned container, software rendering, 1 % tolerance, deterministic snapshot mode, no baselines from developer machines.
- [`.claude/` changes (the new skill) draw a major-severity CI review] -> intentional; PR description says so.
- [Doubled draw calls from per-part assets (16 gondolas, 4 arms)] -> geometry is shared across clones; if measured as a problem, `InstancedMesh` for gondolas is a later optimisation using `lodSource`.
- [Currency check fails for a candidate library] -> fall back to hand-written validation (D7) or defer the item; record the decision.
- [Conflict with the shared "Proposed" status of simulation authority (ADR game-0005)] -> none: this change is client-side rendering only.

## Migration Plan

Additive. Order: doctor gate (task group 0); repo hygiene and docs; manifest and validator; loader and placeholders with the visualization refactored behind them and visually unchanged; prerequisite verification and skill; asset authoring; wiring; visual baselines. Rollback of any step is a revert of its commit: with assets absent or the manifest empty, the loader serves placeholders and the app renders as before. Removing the FBX is its own commit.

## Open Questions

- Should `test:visual` run in CI? Default: no in this change; maintainers decide on browser-minute cost and on adding a workflow job (which attracts the major-severity review rule).
- Whether the Krita MCP server chosen by change 0 can write textures into Krita documents or only open them is recorded in `docs/prerequisites.md`; the Krita-driven tasks here fall back to a documented manual Krita step if it cannot paint; neither outcome changes the specs.
- Whether `gltf-validator` (Khronos) is worth the dependency on top of the in-house validator: decide at apply from the currency check and the ratio of bugs it catches on the first four assets.
