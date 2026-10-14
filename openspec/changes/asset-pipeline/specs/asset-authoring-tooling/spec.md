## ADDED Requirements

### Requirement: MCP servers and the readiness gate come from workshop-prerequisites

This change SHALL NOT add, edit or select MCP servers and SHALL NOT edit `.mcp.json`, `src/.mcp.json`, `CLAUDE.md`, `.github/copilot-instructions.md` or `README.md` for that purpose; those obligations belong to change `workshop-prerequisites`. Before any task that needs Blender, Krita or an MCP server, the apply step SHALL run `node tools/doctor/doctor.mjs --change asset-pipeline` and SHALL stop when it exits non-zero. `docs/asset-pipeline.md` SHALL link to `docs/prerequisites.md` for server setup and risk records, and SHALL state that `npm test`, `npm run build` and `npm run assets:validate` do not require the MCP servers, Blender or Krita.

#### Scenario: Doctor red

- **WHEN** `doctor --change asset-pipeline` reports `mcp:blender` as fail
- **THEN** every task tagged `[needs-blender]` or `[needs-mcp]` stays unchecked and the apply step reports the failing check and its hint

#### Scenario: Teammate without Blender

- **WHEN** a developer without Blender opens the repository
- **THEN** `docs/asset-pipeline.md` states the MCP servers can be ignored and that the build, tests and validator do not require them

### Requirement: An agent skill drives Blender to produce an asset to spec

A skill under `.claude/skills/` SHALL take a written asset spec (class, id, dimensions in metres, joints with axes and ranges, sockets, triangle budget, palette reference, texture needs) and instruct the agent to: read `docs/asset-pipeline.md` and `docs/art-direction.md`; call the Blender MCP server to build the model with the naming grammar, apply transforms, check triangle count and dimensions, save the `.blend` to its `designs/3d/<id>/` path, export `.glb` with the Y-up conversion to `public/assets/models/`, update the manifest, and run `npm run assets:validate`; hand off texture work to Krita only when the spec requires painted textures; and stop and report when a required MCP tool is unavailable instead of improvising a different pipeline. It SHALL prefer the server's structured tools over tools that run arbitrary scripts, and SHALL state each script it does run (the capability record is in `docs/prerequisites.md`). The skill SHALL be mirrored per the repository's rules for skills and SHALL stay out of the distributable plugin because it needs a local Blender.

#### Scenario: Spec to validated asset

- **WHEN** the skill is given the spec for `octopus-cylinder`
- **THEN** it ends with a `.blend`, a `.glb`, a manifest entry and a clean `assets:validate` run, or with a report naming the first failing check

#### Scenario: MCP server missing

- **WHEN** the Blender server is not connected
- **THEN** the skill stops with a message naming the missing server and the setup section of `docs/prerequisites.md`, and writes no partial files

### Requirement: Prompt templates and an asset spec template ship with the skill

The skill's `references/` directory SHALL contain an asset spec template and at least three worked prompt templates for the workshop: model a hard-surface ride part, bake and paint a texture through Krita, and review an existing `.glb` against the conventions. Each template SHALL name the expected output files and the check that proves success.

#### Scenario: Template completeness

- **WHEN** a template is opened
- **THEN** it lists input fields, expected output paths and the validating command

### Requirement: An art-direction document fixes the visual language

`docs/art-direction.md` SHALL define the style (stylised low-poly fairground, flat-shaded surfaces with painted PBR detail, slightly exaggerated proportions), a named palette with sRGB hex values seeded from the colours the 3D view uses today (arm yellow `#e6b826`, hub blue `#347ad9`, gondola green `#27b374`, motor red `#cc2e2e`, structure grey `#8d9096`, base dark grey `#47474d`), roughness and metalness ranges per material family, silhouette and readability rules at the default camera distance, and a do/do-not list. Palette changes SHALL be made in this document only. A human SHALL review and approve it before the first asset is authored from it.

#### Scenario: Palette is explicit

- **WHEN** an asset is authored
- **THEN** every base colour is a palette entry or a documented tint of one

#### Scenario: Approval gate

- **WHEN** the art-direction document has not been approved by a reviewer
- **THEN** the authoring tasks that depend on it remain unchecked
