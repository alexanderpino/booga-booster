## ADDED Requirements

### Requirement: Job metrics use the shared meter

The job system SHALL publish its metrics on the solution's shared `BoogaBoosterTelemetry.Meter` and SHALL NOT configure OpenTelemetry itself. It SHALL count executed jobs (tag `kind` = `job`, `chunk` or `continuation`), stolen jobs, faulted jobs and cancelled jobs, and SHALL record frame duration and per-system duration (tag `system`) in milliseconds. It SHALL also expose observable gauges for queued jobs and for worker utilisation between 0 and 1.

#### Scenario: Executed counter after a frame

- **WHEN** a frame of 100 jobs completes while a meter listener is attached
- **THEN** the executed-jobs counter has increased by exactly 100

#### Scenario: Steal counter

- **WHEN** a frame runs on 4 workers with an unbalanced load
- **THEN** the stolen-jobs counter is greater than 0

#### Scenario: Frame duration recorded

- **WHEN** a frame completes
- **THEN** the frame-duration histogram records one measurement in milliseconds

#### Scenario: System duration is tagged and bounded

- **WHEN** a graph with 3 named systems runs once
- **THEN** the system-duration histogram records 3 measurements, one per system name
- **AND** a graph with more than 64 distinct system names reports the surplus under the tag value `other`

### Requirement: No span per job

The job system SHALL NOT start a distributed-tracing activity per job or per chunk. The cost of metrics SHALL be paid per worker per frame, not per job on a shared counter.

#### Scenario: No activities from job execution

- **WHEN** an activity listener is attached and 10,000 jobs run
- **THEN** no activity is started by the job system for those jobs

### Requirement: Per-frame job timeline

When timeline recording is enabled, the job system SHALL record for each frame the start and end of every job and chunk, its worker, its system, and whether it was stolen, as millisecond offsets from the frame start. Events beyond the per-worker buffer capacity SHALL be counted as dropped and SHALL NOT fail the frame. Recorded times are diagnostic only and SHALL NOT influence any simulation result.

#### Scenario: Timeline of a small frame

- **WHEN** a frame of 3 systems with 10 chunks each runs with timeline recording on
- **THEN** the recorded frame holds 3 system spans and 30 job spans
- **AND** every job span's worker index is between 0 and the worker count minus 1

#### Scenario: Spans are consistent

- **WHEN** a frame is recorded
- **THEN** each job span has an end time at or after its start time
- **AND** each job span lies within its system span

#### Scenario: Buffer overflow is counted

- **WHEN** a frame produces more events than the per-worker buffer holds
- **THEN** the frame reports a dropped-event count greater than 0 and still completes

#### Scenario: Recording off costs nothing observable

- **WHEN** timeline recording is disabled
- **THEN** no frame is stored and the provider reports no timeline

#### Scenario: Timeline does not change results

- **WHEN** the reference mini-simulation runs with recording on and off
- **THEN** the hash sequences are identical

### Requirement: Timeline history is bounded

The provider SHALL keep only the most recent configured number of frames (default 120) and SHALL return the latest completed frame on request.

#### Scenario: History is capped

- **WHEN** 500 frames complete with a history of 120
- **THEN** the provider holds exactly 120 frames
- **AND** the latest frame number is 500

#### Scenario: Recent frames in ascending order

- **WHEN** 500 frames have completed and the provider is asked for the 10 most recent frames
- **THEN** it returns frames 491 to 500 in ascending frame-number order
- **AND** asking for more frames than the history holds returns only the frames held

### Requirement: Timeline endpoint

The Simulation module SHALL expose `GET /simulation/jobs/timeline` returning the latest completed frame's timeline as JSON, `204 No Content` when recording is disabled or no frame has completed yet. The endpoint SHALL contain no business logic and SHALL dispatch to a query handler.

#### Scenario: Latest frame returned

- **WHEN** recording is on and at least one frame has completed
- **THEN** the endpoint returns `200` with the latest frame's timeline

#### Scenario: No data yet

- **WHEN** no frame has completed
- **THEN** the endpoint returns `204`

### Requirement: Timeline data carries only plain values

The timeline response SHALL contain only numbers, strings, booleans and arrays or objects of those, with camelCase property names and a `schemaVersion` field. It SHALL NOT contain handles, delegates or thread objects, so that a non-.NET producer, such as a future browser worker pool, can emit the same shape.

#### Scenario: Round trip through JSON

- **WHEN** a timeline is serialised to JSON and deserialised again
- **THEN** the result equals the original

#### Scenario: Schema version present

- **WHEN** a timeline is serialised
- **THEN** the JSON contains `schemaVersion` with the value 1
