## Why

Every visual in the 3D view is a procedural `BoxGeometry` or `CylinderGeometry` built in `ride-visualization.ts`. That is fine for a physics demo but cannot carry the Octopus (tapered tube arms, hinge pin, hydraulic cylinder) or the tycoon-style park (shops, toilets, thousands of guests, track pieces). ADR `docs/adr/game-0007-asset-pipeline-blender-krita-gltf.md` chose the direction: model in Blender, paint textures in Krita, both driven by AI agents through MCP servers, exported as glTF 2.0 (`.glb`) and loaded with Three.js `GLTFLoader`. This change turns that decision into a repeatable, validated, teachable pipeline, and proves it end to end by replacing the procedural Octopus arm, hinge, hydraulic cylinder and gondola with glTF assets. It is also the clearest showcase in the repo of agents, skills and MCP servers doing creative work under a spec.

## What Changes

- **Asset conventions** (documented in `docs/asset-pipeline.md` and enforced by the validator): repo folder layout for sources (`designs/`) and exports (`src/FourDotnet.BoogaBooster.App/public/assets/`), snake_case node/pivot/empty naming that maps ride joints to scene-graph node ids (ADR `docs/adr/game-0004-scene-graph-transform-hierarchy.md`), unit and axis rules (metres, Blender Z-up to Three.js Y-up, applied transforms), texture rules (power of two, size caps, sRGB vs linear, ORM packing), triangle budgets per asset class, and an LOD plus instancing plan for thousands of guests.
- **Binary hygiene**: `.gitignore` entries for Blender/Krita backup and autosave files, a `.gitattributes` marking `.glb`/`.blend`/`.kra` binary, size budgets, and an explicit threshold at which git-lfs becomes necessary (not adopted now).
- **Asset manifest** `public/assets/manifest.json` with a JSON Schema and a dependency-free Node validation script (`npm run assets:validate`) that also inspects each `.glb` (header, triangle counts, node names, texture sizes, byte budget, sha256) and the provenance/licence record.
- **Angular `AssetLoaderService`**: signal-based status per asset, in-flight de-duplication and caching, `GLTFLoader` behind an injection token, timeout, joint-presence check, and automatic fallback to the procedural placeholder; a PrimeNG loading/fallback indicator in the visualization host.
- **First vertical slice**: the Octopus arm, hinge, hydraulic cylinder and gondola in `RideVisualization` come from glTF assets, while the hub keeps a vertical axis through the compensating linkage that `octopus-motion` introduces. The procedural builders stay as the registered placeholders.
- **Authoring tooling**: an agent skill that drives the Blender MCP server to produce an asset to a written spec, prompt templates for the workshop, and `docs/art-direction.md` (style, palette). The `blender` and `krita` MCP servers themselves (selection, `.mcp.json` entries, the `src/.mcp.json` mirror, `CLAUDE.md` / `.github/copilot-instructions.md` / `README.md` obligations, risk record, smoke tests) are owned by `workshop-prerequisites` (change 0); this change only declares the dependency and gates on `doctor`.
- **Provenance**: every asset carries a licence and provenance record (author, tool chain, AI assistance, sources) in the manifest; only an allow-listed set of licences passes validation.
- **Visual regression**: a Playwright screenshot test of the 3D view with a fixed camera, fixed rig pose and stubbed telemetry.
- Retires the unreferenced `public/3dmodels/boogabooster.fbx` (FBX is not a pipeline format) once a grep proves nothing loads it; the tracked Blender backup `designs/3d/ride-design.blend1` leaves the index.

## Capabilities

### New Capabilities
- `asset-conventions`: Layout, naming, unit/axis, texture, triangle-budget, LOD/instancing, binary-hygiene and provenance rules every asset must satisfy.
- `asset-manifest`: The manifest format, its JSON Schema and the validation script that enforces conventions against the exported `.glb` files.
- `asset-loader`: The Angular service that loads, caches, validates and falls back for glTF assets, and exposes loading state.
- `ride-model-assets`: The Octopus arm, hinge, hydraulic cylinder and gondola as glTF assets in the 3D view, with the hub vertical-axis behaviour preserved.
- `asset-authoring-tooling`: The authoring skill, prompt templates, the art-direction document, and the declared dependency on the MCP servers and `doctor` gate provided by `workshop-prerequisites`.
- `scene-visual-regression`: The deterministic Playwright screenshot test of the 3D view.

### Modified Capabilities
<!-- None. The published capabilities describe telemetry, controls and domain behaviour; none specifies how the 3D view is built, so no existing requirement changes. -->

## Impact

- **Frontend** (`src/FourDotnet.BoogaBooster.App`): new `src/app/assets/` (loader service, placeholder registry, status UI), changes to `ride-visualization.ts` (builders behind the loader), `public/assets/` (manifest, `.glb` files), `tools/assets/` (schema, validator, tests), `e2e/` (Playwright), `package.json` scripts and devDependencies (candidates only: a Playwright test package; optional JSON Schema validator and Khronos glTF validator, each subject to the currency check).
- **Repo root**: `designs/` (source `.blend`/`.kra`), `.gitignore`, `.gitattributes`, `docs/asset-pipeline.md`, `docs/art-direction.md`, `.claude/skills/<authoring skill>/`. (`.mcp.json`, `src/.mcp.json`, `CLAUDE.md`, `.github/copilot-instructions.md` and `README.md` are no longer touched here; see `workshop-prerequisites`.)
- **Backend / telemetry**: none. No endpoint, DTO or physics change.
- **Dependencies**: requires `workshop-prerequisites` (change 0: `blender`/`krita` MCP servers declared and available, `doctor --change asset-pipeline` green) and `octopus-motion` (the elevation and compensating-linkage behaviour the slice must preserve). Provides node-naming and manifest contracts to `scene-graph`, `ride-catalog`, `guests-and-needs` and `coaster-editor`.
- **Governance**: adds a skill under `.claude/`, so the PR will be flagged major by the CI review by design and needs human review; the `ai-tooling-sync` check must pass. It no longer touches `CLAUDE.md` or MCP configuration.

> **Quality bar:** this change is for a workshop demo, not a production release. See `docs/demo-scope.md` for what may be deferred and what may not.
