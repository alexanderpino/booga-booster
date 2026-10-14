## ADDED Requirements

### Requirement: The loader service exposes asset state as signals

`AssetLoaderService` (root-provided, `src/app/assets/`) SHALL expose `status(id): Signal<AssetStatus>` where `AssetStatus` is one of `idle`, `loading`, `ready`, `fallback`, with a `lastError` signal holding `{assetId, code, message}` for the most recent failure, and a computed `pending` count and `progress` fraction (ready or fallback assets over requested assets) for aggregate UI. State SHALL change only through signals; the service SHALL work with zoneless change detection.

#### Scenario: Unknown asset is idle

- **WHEN** `status('octopus-arm')` is read before any `load` call
- **THEN** it is `idle`

#### Scenario: Status progression

- **WHEN** `load('octopus-arm')` is called and the file loads
- **THEN** `status` is `loading` while in flight and `ready` afterwards

#### Scenario: Progress

- **WHEN** four assets are requested and two have finished
- **THEN** `progress()` is 0.5 and `pending()` is 2

### Requirement: Manifest is fetched once and models are cached

The service SHALL fetch `assets/manifest.json` at most once per page load and SHALL de-duplicate concurrent `load(id)` calls for the same id into one request. `instantiate(id)` SHALL return an independent `THREE.Object3D` clone of the cached scene (geometry and materials shared) so several instances cost one download and one GPU upload. `dispose()` SHALL release cached geometries, materials and textures; the service SHALL call it from `DestroyRef`.

#### Scenario: Concurrent loads share one request

- **WHEN** `load('octopus-gondola')` is called 16 times in the same tick
- **THEN** the injected glTF loader is invoked exactly once

#### Scenario: Instances are independent

- **WHEN** `instantiate('octopus-gondola')` is called twice and the first clone is rotated
- **THEN** the second clone's rotation is unchanged
- **AND** both clones reference the same geometry object

#### Scenario: Manifest fetched once

- **WHEN** three different assets are loaded
- **THEN** `manifest.json` is requested once

### Requirement: The glTF loader is injectable and bounded

`GLTFLoader` SHALL be created through an injection token so tests substitute a fake. A load SHALL fail with code `timeout` if it takes longer than `ASSET_LOAD_TIMEOUT_MS` (initial value 15,000, a named constant). The model URL SHALL follow the manifest cache-key rule. Draco, Meshopt and KTX2 decoders SHALL NOT be configured by this change; a `.glb` using an extension the loader cannot decode SHALL fail with code `unsupported-extension`.

#### Scenario: Timeout

- **WHEN** the fake loader never resolves and 15,000 ms of fake time pass
- **THEN** `status` becomes `fallback` and `lastError().code` is `timeout`

#### Scenario: Unsupported extension

- **WHEN** a model requires `KHR_draco_mesh_compression`
- **THEN** the load fails with `unsupported-extension` and the status becomes `fallback`

### Requirement: Failures fall back to the procedural placeholder

When the manifest is unreachable or invalid, the asset is absent from the manifest, the file fails to load or parse, the load times out, or a manifest joint or socket node is missing from the loaded scene, the service SHALL set the status to `fallback`, record `lastError`, and make `instantiate(id)` return the object built by the placeholder registered under the entry's `placeholder` key (or under the asset id when the manifest itself failed). The failure SHALL NOT throw out of `instantiate` and SHALL be logged once per asset id. Placeholders SHALL expose the same joint and socket node names as the real asset, so callers bind joints identically in both cases. `retry(id)` SHALL clear the failure and attempt the load again.

#### Scenario: 404 on the model

- **WHEN** the model request returns HTTP 404
- **THEN** `status('octopus-arm')` is `fallback` with `lastError().code` `load-failed`
- **AND** `instantiate('octopus-arm')` returns a node containing `joint_arm_elevation` and `socket_hub_mount`

#### Scenario: Joint missing from a real file

- **WHEN** the manifest declares `joint_arm_elevation` but the loaded scene has no such node
- **THEN** the status is `fallback` and `lastError().code` is `joint-missing`

#### Scenario: Manifest unreachable

- **WHEN** `manifest.json` returns HTTP 500
- **THEN** every requested asset is `fallback` with code `manifest-unavailable` and the page still renders procedurally

#### Scenario: Retry succeeds

- **WHEN** `retry('octopus-arm')` is called after a fallback and the file now loads
- **THEN** `status('octopus-arm')` becomes `ready` and later `instantiate` calls return the real model

### Requirement: Loading state is visible with PrimeNG and accessible

The visualization host SHALL show, while `pending() > 0`, a PrimeNG progress indicator bound to `progress()` with an accessible name, and SHALL show a PrimeNG message with severity `warn` naming the failing assets and a retry action when any asset is in `fallback`. The 3D canvas SHALL remain visible and animating throughout (placeholders render immediately and are swapped when the real model is ready). The new UI SHALL pass the repository's axe-core WCAG AA spec and SHALL be built after consulting the `primeng` MCP server.

#### Scenario: Progress shown

- **WHEN** one of four assets is still loading
- **THEN** a progress indicator with an accessible name is rendered with value 75

#### Scenario: Fallback message

- **WHEN** an asset is in `fallback`
- **THEN** a warning message names the asset and offers a Retry button that calls `retry`

#### Scenario: Accessibility

- **WHEN** the visualization host is rendered in each of the states loading, ready and fallback
- **THEN** axe-core reports no violations

### Requirement: Swapping placeholder for asset keeps the pose

When an asset becomes ready after its placeholder was already in the scene, the swap SHALL happen without a frame in which the part is missing and SHALL preserve the parent joint's current rotation.

#### Scenario: Late arrival

- **WHEN** the arm joint is at +20 degrees and the real arm model becomes ready
- **THEN** the real arm is mounted under the same joint, which is still at +20 degrees
- **AND** the placeholder is removed and its geometry disposed in the same frame
