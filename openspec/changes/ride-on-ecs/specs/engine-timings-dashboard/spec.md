## ADDED Requirements

### Requirement: Engine timings endpoint

The DigitalTwin module SHALL expose `GET /ride/engine/timings` returning the active engine (`Legacy`, `Ecs` or `Shadow`), whether timing data is available, the job worker count, tick statistics over the last 120 ticks, per-system statistics, the latest job timeline, and in `Shadow` mode the divergence count. The endpoint SHALL contain no business logic and SHALL dispatch to a query handler.

#### Scenario: Statistics content

- **WHEN** the endpoint returns timing data
- **THEN** tick statistics hold the last, average, median, 99th percentile and maximum duration in milliseconds, the budget 8.333 and the over-budget count
- **AND** each system holds its name, last, average and maximum duration, read set, write set and dependency level

#### Scenario: Ecs engine with timing data

- **WHEN** the engine is `Ecs`, timeline recording is on and at least one tick has run
- **THEN** the endpoint returns 200 with `available` true, ten systems, and a timeline with at least one job span

#### Scenario: Legacy engine

- **WHEN** the engine is `Legacy`
- **THEN** the endpoint returns 200 with `engine` `Legacy` and `available` false
- **AND** the systems list is empty

#### Scenario: Shadow engine

- **WHEN** the engine is `Shadow`
- **THEN** the response includes the divergence count and the ECS timings

#### Scenario: Recording disabled

- **WHEN** the engine is `Ecs` and timeline recording is off
- **THEN** `available` is false and tick statistics are still returned

### Requirement: Rolling statistics are bounded and correct

The tick and per-system statistics SHALL be computed over a rolling window of the last 120 ticks, SHALL use no allocation per tick beyond the window itself, and SHALL be exact for the window (not estimated).

#### Scenario: Window of 120

- **WHEN** 500 ticks of known durations have been recorded
- **THEN** the average, median, 99th percentile and maximum are computed from the last 120 only

#### Scenario: Over-budget count

- **WHEN** 3 of the last 120 ticks took longer than 8.333 ms
- **THEN** the over-budget count is 3

### Requirement: The timings view shows tick, systems and jobs

The Angular app SHALL provide a route `/engine` that shows the active engine as a tag, the tick duration against the 8.333 ms budget, a table of systems with their durations and read and write sets, and a worker-lane timeline of the latest frame. The view SHALL be built with PrimeNG components, SHALL poll the endpoint once per second, and SHALL show an explanatory empty state when timing data is not available.

#### Scenario: Systems table

- **WHEN** the endpoint returns ten systems
- **THEN** the view lists ten rows, each with the system name, average and maximum duration and its read and write sets

#### Scenario: Budget bar

- **WHEN** the average tick is 0.25 ms
- **THEN** the budget indicator shows about 3% of 8.333 ms used, and its text value reads "0.25 ms of 8.33 ms"

#### Scenario: Worker lanes

- **WHEN** the timeline has job spans on 4 workers
- **THEN** the view draws 4 lanes, one per worker, with a block for each span positioned by start and end time

#### Scenario: Empty state

- **WHEN** `available` is false
- **THEN** the view shows a message that timings need the `Ecs` or `Shadow` engine with timeline recording
- **AND** no table or lanes are shown

#### Scenario: Polling stops when the view closes

- **WHEN** the user navigates away from `/engine`
- **THEN** no further requests to the timings endpoint are made

### Requirement: The timings view is accessible

The timings view SHALL pass automated axe-core checks for WCAG AA, SHALL be usable by keyboard, and SHALL give the worker-lane graphic a text alternative: a labelled summary and a table of the same data, so that no information is available only graphically.

#### Scenario: Axe check

- **WHEN** the view renders with timing data
- **THEN** axe-core reports no violations

#### Scenario: Lane graphic has a text equivalent

- **WHEN** the worker lanes are shown
- **THEN** the graphic has an accessible name and a following table lists each worker's busy time and job count

#### Scenario: Colour is not the only signal

- **WHEN** a tick is over budget
- **THEN** the view shows a text label and an icon in addition to any colour change

### Requirement: The view does not disturb the existing dashboard

The existing dashboard route, the telemetry SSE connection, and the 3D visualization SHALL be unchanged. The `/engine` route SHALL be lazy-loaded, and the dashboard SHALL link to it.

#### Scenario: Dashboard unaffected

- **WHEN** the app loads the dashboard route
- **THEN** the engine timings code is not loaded

#### Scenario: Link to the view

- **WHEN** the dashboard renders
- **THEN** it offers a link that navigates to `/engine`
