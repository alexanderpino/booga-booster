## ADDED Requirements

### Requirement: Source and export locations are fixed

Source files SHALL live under `designs/`: Blender files at `designs/3d/<asset-id>/<asset-id>.blend` and Krita files at `designs/textures/<asset-id>/<name>.kra`. Exported runtime files SHALL live under `src/FourDotnet.BoogaBooster.App/public/assets/`: models at `models/<asset-id>.glb` and any external texture at `textures/<asset-id>/<name>.png`. The manifest SHALL live at `public/assets/manifest.json`. An `<asset-id>` SHALL match `^[a-z][a-z0-9-]*$`. Backup and autosave files (`*.blend1`, `*.blend@`, Krita `*~` and autosave files) SHALL be git-ignored. `.glb`, `.blend` and `.kra` SHALL be marked `binary` in `.gitattributes`.

#### Scenario: Asset id and file names agree

- **WHEN** an asset has id `octopus-arm`
- **THEN** its source is `designs/3d/octopus-arm/octopus-arm.blend`
- **AND** its export is `public/assets/models/octopus-arm.glb`

#### Scenario: Blender backup file is not committed

- **WHEN** Blender saves `octopus-arm.blend` and creates `octopus-arm.blend1`
- **THEN** `git status` does not list `octopus-arm.blend1`

#### Scenario: Legacy tracked backup leaves the index

- **WHEN** this change is applied
- **THEN** `designs/3d/ride-design.blend1` is no longer tracked by git
- **AND** `designs/3d/ride-design.blend` is unchanged

### Requirement: Binary size budgets and the git-lfs trigger are explicit

The pipeline SHALL NOT use git-lfs initially. A `.glb` ride-part asset SHALL NOT exceed 300 KiB, a guest asset 150 KiB, and any other asset 500 KiB. The sum of all files under `public/assets/` SHALL NOT exceed 8 MiB. A `.blend` or `.kra` source SHALL NOT exceed 10 MiB, and the combined size of `designs/` SHALL NOT exceed 100 MiB. `docs/asset-pipeline.md` SHALL state that exceeding any source limit requires a follow-up change adopting git-lfs (covering CI checkout, Copilot CLI review, worktrees and clone docs) before the file is committed.

#### Scenario: Oversized export is rejected

- **WHEN** `octopus-gondola.glb` is 310 KiB and its class budget is 300 KiB
- **THEN** `npm run assets:validate` exits non-zero and names the file, its size and the budget

#### Scenario: Source over the lfs threshold

- **WHEN** a `.kra` file would be 12 MiB
- **THEN** the documented procedure forbids committing it and points to the git-lfs follow-up

### Requirement: Unit and axis conventions

Assets SHALL be authored in Blender with 1 Blender unit = 1 metre, Z up. glTF export SHALL use the Y-up conversion (Blender `+Z` becomes glTF `+Y`, Blender `+Y` becomes glTF `-Z`, `+X` is unchanged), so a `.glb` is already in Three.js space and the client applies no rotation or scale on load. Before export every object SHALL have scale `(1,1,1)` and mesh transforms SHALL be applied; joint empties SHALL keep zero rotation and unit scale so a joint's rotation axis is a local axis of its node. The asset origin SHALL be the natural mounting point (arm: hinge pin centre; gondola: top attachment point; building: centre of the ground footprint at `y = 0`). Linear dimensions SHALL be metres in glTF space.

#### Scenario: Arm length survives export

- **WHEN** the 6 m Octopus arm is authored along Blender `+X` from the hinge pin
- **THEN** after loading, the distance in glTF space from `joint_arm_elevation` to `socket_hub_mount` is 6.0 m within 0.001 m

#### Scenario: Up axis

- **WHEN** an asset is authored with its top 3 m above the Blender ground plane
- **THEN** the loaded model's bounding-box maximum Y is 3.0 m within 0.001 m

#### Scenario: Unapplied scale is a validation error

- **WHEN** an exported node has scale other than `(1,1,1)`
- **THEN** the validator reports `non-unit-scale` with the node name

### Requirement: Node names are stable ids with a fixed role grammar

Every glTF node name SHALL match `^[a-z][a-z0-9_]*$`, which rules out Blender duplicate suffixes (`.001`), spaces and the characters Three.js `PropertyBinding` sanitises. Names SHALL use one of these role prefixes: `root` (exactly one per file, the scene root, no mesh), `joint_<name>` (an empty at the pivot point; the animated degree of freedom is rotation about, or translation along, a local axis), `socket_<name>` (an empty marking an attachment point, with an optional numeric suffix `_0`, `_1` for repeats), `mesh_<name>` (renderable geometry), and `mesh_<name>_lod<n>` for LOD levels `n >= 1`. Names SHALL be unique within a file. A joint's id in the scene graph SHALL be the asset-instance node id plus the glTF node name, so a joint is addressable as `<instance-node-id>/<joint-name>` (ADR `docs/adr/game-0004-scene-graph-transform-hierarchy.md`).

#### Scenario: Valid hierarchy

- **WHEN** the arm file contains `root`, `joint_arm_elevation`, `mesh_arm`, `mesh_hinge_bracket` and `socket_hub_mount`
- **THEN** the validator accepts the names

#### Scenario: Blender duplicate suffix

- **WHEN** a node is named `mesh_arm.001`
- **THEN** the validator reports `invalid-node-name`

#### Scenario: Missing root

- **WHEN** a file contains no node named `root`
- **THEN** the validator reports `missing-root`

### Requirement: Joints declare axis, kind and range

Every `joint_*` node SHALL have a matching entry in the asset's manifest `joints` list giving its `kind` (`revolute`, `prismatic` or `fixed`), its `axis` (`x`, `y` or `z` in glTF space) and, for non-fixed kinds, its `min` and `max` (degrees for `revolute`, metres for `prismatic`). The Octopus conventions SHALL be: arm along `+X` from its hinge; `joint_arm_elevation` revolute about `z`, where positive angle raises the arm tip toward `+Y`; `joint_hub_spin` and `joint_gondola_spin` revolute about `y`; hydraulic rod extension prismatic along the rod's local `+Y`.

#### Scenario: Elevation sign

- **WHEN** `joint_arm_elevation` is rotated by +10 degrees about `z` on an arm lying along `+X`
- **THEN** the arm tip's world Y increases

#### Scenario: Undeclared joint

- **WHEN** a file has `joint_arm_elevation` but the manifest lists no such joint
- **THEN** the validator reports `joint-not-declared`

### Requirement: Texture rules

Textures SHALL have power-of-two width and height. Maximum edge length per class SHALL be: ride part 1024, gondola 512, shop 1024, toilet 512, guest atlas 256, track piece 512. Colour data (base colour, emissive) SHALL be authored and flagged sRGB; data textures (normal, occlusion-roughness-metalness) SHALL be linear with no colour profile applied. PBR channel packing SHALL be glTF's: one ORM texture with occlusion in R, roughness in G, metalness in B. Normal maps SHALL use the OpenGL convention (+Y up), as glTF requires. Textures SHALL be embedded in the `.glb` unless the manifest lists them as external. Every asset SHALL use at most 2 materials, except the ride arm and gondola which MAY use 4.

#### Scenario: Non power-of-two texture

- **WHEN** a base-colour texture is 1000 x 1000
- **THEN** the validator reports `texture-not-power-of-two` with the image index

#### Scenario: Texture over the class cap

- **WHEN** a toilet asset embeds a 1024 x 1024 texture and the toilet cap is 512
- **THEN** the validator reports `texture-too-large`

#### Scenario: Colour space in the loaded scene

- **WHEN** a model with a base-colour map and an ORM map is loaded in Three.js
- **THEN** the base-colour texture's colour space is sRGB and the ORM texture's is not

### Requirement: Triangle budgets per asset class

LOD0 triangle counts SHALL NOT exceed: ride arm 2,500; ride hinge 800; hydraulic cylinder (body plus rod) 600; hub drum 1,500; gondola 3,000; shop 4,000; toilet 1,500; guest 300; track piece (4 m segment) 800. These are initial budgets, re-measured against frame time when the park-world changes land, and changed only by editing `docs/asset-pipeline.md` and the validator's class table together. Triangles SHALL be counted from glTF index accessors (or position accessors for non-indexed primitives).

#### Scenario: Guest over budget

- **WHEN** a guest LOD0 mesh has 340 triangles
- **THEN** the validator reports `triangle-budget-exceeded` with the count (340) and the budget (300)

#### Scenario: Count agrees with the manifest

- **WHEN** the manifest records `triangles: 2100` for the arm and the file contains 2,144
- **THEN** the validator reports `manifest-mismatch` for the field `triangles`

### Requirement: LOD and instancing plan for guests

Guest assets SHALL provide three levels: LOD0 up to 300 triangles, LOD1 up to 120, LOD2 an impostor of at most 2 triangles. Levels SHALL be separate `mesh_*_lod<n>` nodes in one file that share one material. The runtime plan SHALL be one `InstancedMesh` per LOD level selected by camera distance with thresholds held as named constants (initial values 15 m and 40 m), driven from the simulation's position arrays, so that 5,000 guests cost no more than three draw calls for guests. Guest animation SHALL NOT use per-instance skinned skeletons. The manifest `lods` list SHALL be the single source of which nodes are which level.

#### Scenario: Manifest lists levels

- **WHEN** the guest manifest entry is read
- **THEN** `lods` has entries for levels 0, 1 and 2 with their node names and triangle ceilings 300, 120 and 2

#### Scenario: Non-shared material across LODs

- **WHEN** `mesh_guest_lod1` uses a different material than `mesh_guest`
- **THEN** the validator reports `lod-material-mismatch`

### Requirement: Provenance and licence are recorded per asset

Every asset SHALL have a manifest `provenance` record: `author`, `tool` chain (for example Blender version, Krita version), `aiAssisted` (boolean) with `generator` when true, `createdAt` (ISO date), `license` and `sources` (list of upstream references, empty when original). `license` SHALL be one of `original` (project-owned, under the repository LICENSE), `CC0-1.0` or `CC-BY-4.0`. `CC-BY-4.0` SHALL require a non-empty `attribution`. Any third-party model, texture or HDRI SHALL be listed in `sources` with a URL and the licence the source states; assets of unknown licence SHALL NOT be committed.

#### Scenario: Original asset

- **WHEN** an asset is modelled from scratch by an agent through the Blender MCP server
- **THEN** its provenance has `license: original`, `aiAssisted: true` and a non-empty `generator`

#### Scenario: Disallowed licence

- **WHEN** an entry declares `license: "CC-BY-NC-4.0"`
- **THEN** the validator reports `license-not-allowed`

#### Scenario: Attribution missing

- **WHEN** an entry declares `CC-BY-4.0` and an empty `attribution`
- **THEN** the validator reports `attribution-required`
