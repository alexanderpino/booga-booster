## ADDED Requirements

### Requirement: A manifest describes every runtime asset

`public/assets/manifest.json` SHALL be a JSON object with `schemaVersion` (integer, initially `1`) and an `assets` array. Each entry SHALL have: `id`, `class` (one of `ride-arm`, `ride-hinge`, `ride-cylinder`, `ride-hub`, `gondola`, `shop`, `toilet`, `guest`, `track-piece`), `version` (integer, incremented on every re-export), `file` (path relative to `public/assets/`), `bytes`, `sha256` (hex, of the file), `triangles` (LOD0), `materials` (count), `lods` (list of `{level, node, maxTriangles}`; level 0 required), `joints` (list of `{node, kind, axis, min, max}`), `sockets` (list of node names), `placeholder` (key of a registered procedural builder), `source` (`{blend, textures[]}` repo-relative paths) and `provenance`. `id` values SHALL be unique and match the file name stem.

#### Scenario: Minimal valid manifest

- **WHEN** the manifest is `{"schemaVersion":1,"assets":[]}`
- **THEN** validation succeeds

#### Scenario: Duplicate id

- **WHEN** two entries share `id: "octopus-arm"`
- **THEN** validation reports `duplicate-id`

#### Scenario: Unknown class

- **WHEN** an entry has `class: "rollercoaster"`
- **THEN** schema validation fails naming the entry and the allowed classes

### Requirement: A JSON Schema is the machine contract

The manifest format SHALL be defined by `tools/assets/asset-manifest.schema.json` (JSON Schema draft 2020-12) under the Angular app. The schema SHALL set `additionalProperties: false` on every object so a typo in a field name fails validation. The manifest SHALL reference the schema through `$schema` for editor support. The loader's TypeScript types SHALL mirror the schema; a unit test SHALL parse a sample entry with both.

#### Scenario: Misspelled field

- **WHEN** an entry contains `trianges` instead of `triangles`
- **THEN** validation fails with the unknown property and the entry id

#### Scenario: Editor support

- **WHEN** the manifest is opened in VS Code
- **THEN** the `$schema` reference resolves to the schema file in the repository

### Requirement: A validation script enforces the conventions against the files

`npm run assets:validate` (run from the Angular app) SHALL execute `tools/assets/validate-assets.mjs`, which uses no network and exits non-zero on any error. It SHALL check the manifest against the schema; then for each entry that the referenced file exists; that it is a binary glTF 2.0 container (magic `glTF`, version 2, consistent length); the declared `bytes` and `sha256`; node names and the role grammar; exactly one `root`; unit scale; triangle count against the class budget and the manifest; texture dimensions and class caps; material count; every manifest joint and socket present as a node and every `joint_*` node declared; LOD nodes present; byte budgets; and the provenance rules. It SHALL print one line per error as `<asset-id>: <code>: <detail>` and a final summary. It SHALL accept `--write-derived` to rewrite only the derived fields (`bytes`, `sha256`, `triangles`, `materials`) from the files. The validator SHALL be covered by `node --test` tests that build tiny `.glb` fixtures in code.

#### Scenario: Clean run

- **WHEN** all committed assets satisfy the conventions
- **THEN** the script prints `OK: <n> assets` and exits 0

#### Scenario: Stale hash after re-export

- **WHEN** `octopus-arm.glb` is re-exported and the manifest still holds the old `sha256`
- **THEN** the script prints `octopus-arm: manifest-mismatch: sha256` and exits non-zero
- **AND** running it with `--write-derived` then fixes the manifest so a second run is clean

#### Scenario: Missing file

- **WHEN** an entry's `file` does not exist
- **THEN** the script prints `<id>: file-missing: <path>` and exits non-zero

#### Scenario: Truncated glb

- **WHEN** the header declares a length larger than the file
- **THEN** the script prints `<id>: invalid-glb: length` and exits non-zero without throwing

#### Scenario: Fixture-driven test

- **WHEN** the test builds a one-triangle `.glb` with nodes `root` and `mesh_a`
- **THEN** the validator reports no error for it and reports `triangle-budget-exceeded` when the same fixture is checked against a budget of 0

### Requirement: The runtime cache key derives from the manifest

The loader SHALL request a model as `assets/<file>?v=<first 8 hex of sha256>` so a changed export bypasses stale browser caches while an unchanged one stays cacheable.

#### Scenario: URL carries the content hash

- **WHEN** an entry has `file: "models/octopus-arm.glb"` and `sha256: "a1b2c3d4e5f6..."`
- **THEN** the request URL is `assets/models/octopus-arm.glb?v=a1b2c3d4`
