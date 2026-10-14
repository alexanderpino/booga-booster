## ADDED Requirements

### Requirement: The 3D view has a deterministic snapshot mode

When the app is opened with the query parameter `snapshot=1`, `RideVisualization` SHALL render a single static frame instead of an animation loop: fixed camera position and target (named constants), no `OrbitControls` damping, device pixel ratio 1, rig angles taken from fixed constants (mill angle, hub angle, per-arm elevation, gondola phases), and no wall-clock time or randomness. After all requested assets are `ready` or `fallback` and one frame is drawn, it SHALL set `document.documentElement.dataset.vizReady = 'true'`. The mode SHALL change no behaviour when the parameter is absent.

#### Scenario: Ready marker

- **WHEN** the page is loaded with `?snapshot=1` and the assets have settled
- **THEN** `data-viz-ready="true"` is set on the document element after the first drawn frame

#### Scenario: Normal mode unchanged

- **WHEN** the page is loaded without the parameter
- **THEN** the animation loop and controls behave as before and `data-viz-ready` is absent

### Requirement: Playwright compares screenshots of the 3D view

An end-to-end test under `e2e/` SHALL open the app at a fixed 1280 x 720 viewport with `snapshot=1`, stub the telemetry and queue/weather HTTP calls with fixed responses via request interception (no backend), wait for `data-viz-ready`, and compare a screenshot of the canvas element to a committed baseline with a maximum differing pixel ratio of 0.01. Baselines SHALL be produced and updated only in the pinned Playwright container image used for CI-equivalent runs, because rendering differs across operating systems and GPUs; the browser SHALL be launched with software rasterisation. The test SHALL run via `npm run test:visual` and SHALL NOT be part of `npm test`. Playwright and any related package are candidates and SHALL pass the currency check before being added.

#### Scenario: Unchanged scene passes

- **WHEN** the view is rendered twice in the same container
- **THEN** both screenshots match the baseline within the threshold

#### Scenario: Changed asset fails

- **WHEN** the arm model is replaced by one with a visibly different silhouette
- **THEN** the screenshot comparison fails and the diff image is written to the test output directory

#### Scenario: Fallback render is a separate baseline

- **WHEN** the test blocks `**/assets/models/**` requests
- **THEN** the view renders the procedural fallback and matches its own baseline `ride-fallback`

#### Scenario: Not in the unit-test run

- **WHEN** `npm test` runs
- **THEN** no Playwright test executes
