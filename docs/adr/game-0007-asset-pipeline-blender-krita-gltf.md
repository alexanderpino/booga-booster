# game-0007: Asset pipeline Blender + Krita -> glTF 2.0 -> Three.js

- Status: Proposed (the user must accept)
- Date: 2026-10-14
- Deciders: project owner
- Related: game-0004

## Context

The 3D view currently uses procedural primitives (BoxGeometry arms in `ride-visualization.ts`). Realistic rides, scenery and guests need authored assets. A seed exists: `designs/3d/ride-design.blend`. Agents with Blender MCP and Krita MCP can author and edit assets, which is part of the workshop showcase. Assets must map onto the scene graph joints (game-0004) so animation comes from simulation, not from baked clips.

## Decision drivers

1. Standard, web-friendly format with first-class Three.js support.
2. Node names and pivots that match simulation nodes.
3. Small download size and GPU-friendly textures.
4. Traceable provenance and licences (AI-generated and third-party content).
5. A human review gate before anything ships.
6. Reproducible export (scriptable).

## Considered options

### Option A: glTF 2.0 binary (.glb) via Three.js GLTFLoader (recommended)
- Pros: open standard, PBR materials, node hierarchy preserved, single-file, loader included with Three.js examples (to be confirmed for pinned 0.169 at apply time).
- Cons: no native support for some Blender features (procedural nodes must be baked).

### Option B: FBX or OBJ
- Pros: widely supported by DCC tools.
- Cons: proprietary (FBX) or no hierarchy/materials (OBJ); heavier loaders.

### Option C: Keep procedural geometry only
- Pros: no assets, no licences.
- Cons: cannot reach the realism the Octopus and park goals need.

## Decision

Option A.

1. **Pipeline**: Blender (via Blender MCP) for meshes, rigs-as-hierarchy and baking -> Krita (via Krita MCP) for textures/sprites (PNG sources, kept as .kra in `designs/`) -> export from Blender to `.glb` -> runtime load with `GLTFLoader` -> loaded nodes are bound by name to simulation nodes. Mesh compression (Draco/meshopt) and KTX2 textures are optional later optimisations, each subject to the currency check.
2. **Naming conventions** (so scene-graph joints map): `<Kind>_<Index>[_<Part>]`, PascalCase, ASCII, no spaces or dots. Examples: `Mill`, `Arm_0`..`Arm_3`, `Arm_0_Truss`, `ArmHinge_0`, `ArmCylinder_0`, `Hub_0`, `HubSpin_0`, `HubArm_0_2`, `Gondola_0_2`. Rules: (a) every simulation-driven transform is an **Empty/Object node named exactly as in the backend node list**, with its **origin at the physical pivot** (hinge pin for arms, rotation axis centre for hubs); (b) meshes are children of those nodes and never carry the animated transform; (c) the rest pose is the pose at angle 0 / elevation 0; (d) Y-up, metres, scale 1 applied (Blender export option for +Y up), no negative scale; (e) the linkage parts (hydraulic cylinder, compensating rods) are separate child nodes so the client can aim them from the same snapshots.
3. **Texture sizes**: power-of-two; 1024 x 1024 default, 2048 for hero assets (ride arms, gondolas), 256 or 512 for props; sRGB for colour maps, linear for normal/ORM maps; target total scene texture memory budget agreed at apply time (suggest < 64 MB GPU after decode, to be measured). Single .glb size budget 5 MB per ride (target).
4. **Licences and provenance**: each asset has a sidecar `<asset>.provenance.md` or manifest row: author (human or tool, including which AI/MCP agent), source files, licence (CC0/CC-BY/own), date, prompts if generated. No asset enters without a licence; CC-BY needs an attributions list. Third-party stock only from licences compatible with the repo's licence (confirm).
5. **Seed**: `designs/3d/ride-design.blend` is the starting model; first task is an audit (names, pivots, scale, materials) and a rename pass to this convention, not a rebuild.
6. **Review gate**: an automated validation (glTF validator, name/pivot check script, size budgets) runs in CI; then a human review of renders (screenshots from Blender MCP and from the browser) before the asset is merged. Agents can propose, never self-approve.
7. **Scripting**: export uses a committed Blender Python script (headless) so output is reproducible and not hand-clicked.

## Consequences

- Good: realistic visuals, standard tooling, names tie art to simulation; the workshop shows agent-driven DCC work.
- Bad: binary assets in git (consider Git LFS, decide at apply time); MCP-authored geometry may need human clean-up; naming discipline needed.
- Neutral: procedural placeholders remain as fallback when a node is missing in the glb.

## Validation

- CI: Khronos glTF-Validator reports 0 errors for every .glb (tool existence and version to be confirmed at apply time).
- Script test: every node name expected by the backend node list exists exactly once in the .glb; every driven node's origin is within 1 mm of the declared pivot.
- Visual: rest-pose render matches reference screenshot (pixel diff threshold agreed at apply time); animated pose test: Arm_0 at 30 degrees elevation puts Hub_0 at the formula position within 1e-3 m.
- Budget: .glb <= 5 MB, load time < 1 s on a local dev server (target).
- Provenance check: 100% of assets in `src/.../assets` have a manifest entry (script).

## References

- docs/adr/game-0004-scene-graph-transform-hierarchy.md
- designs/3d/ride-design.blend (repo seed)
- glTF 2.0 specification (Khronos) and Three.js GLTFLoader docs - NOT fetched in this session; unverified for the pinned versions.
