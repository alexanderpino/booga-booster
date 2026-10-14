# Spec Delta

## Purpose

Defines how coaster state reaches the browser under the backend-authoritative model of ADR `docs/adr/game-0005-simulation-authority-and-state-sync.md`: commands go to the server over HTTP, the track structure is sent once per design version, and the live state of a train is two numbers that the client animates along the same track.

## ADDED Requirements

### Requirement: The editor is driven by HTTP commands and queries (Tier 1)

The system SHALL expose: `GET /coaster/catalogue`; `POST /coaster/designs`; `GET /coaster/designs/{id}`; `POST /coaster/designs/{id}/pieces`; `DELETE /coaster/designs/{id}/pieces/last`; `POST /coaster/designs/{id}/undo`; `POST /coaster/designs/{id}/redo`; `POST /coaster/designs/{id}/preview`; `POST /coaster/designs/{id}/test-run`; `GET /coaster/designs/{id}/test-run`; `GET /coaster/designs/{id}/export`; `POST /coaster/designs/import`; `POST /coaster/designs/{id}/build`. Endpoints SHALL parse requests and dispatch to handlers; they SHALL contain no rules. A broken rule SHALL be answered with 400, a revision or validation conflict with 409, an unknown id with 404.

#### Scenario: Unknown design

- **WHEN** `GET /coaster/designs/{id}` is called with an unknown id
- **THEN** the answer is 404

#### Scenario: Conflict carries the current revision

- **WHEN** an append is sent with a stale revision
- **THEN** the answer is 409 and the body holds the current revision and the current design

### Requirement: Every mutation returns the full editing view (Tier 1)

The response to a create, append, remove, undo, redo or import SHALL contain the design (revision, state, pieces with their placed lattice frames, train configuration, content hash), the validation report, the palette, the cost estimate if available, and the undo and redo availability, so the client needs no second request to redraw.

#### Scenario: One round trip

- **WHEN** a piece is appended
- **THEN** the response holds the new revision, the placed frame of every piece, the report and the palette

### Requirement: The catalogue geometry is sent once and cached (Tier 1)

`GET /coaster/catalogue` SHALL carry a catalogue version and an `ETag` derived from it. A request with a matching `If-None-Match` SHALL be answered with 304. The client SHALL cache the response for the session and SHALL refetch it when a design response names a different catalogue version.

#### Scenario: Conditional request

- **WHEN** the catalogue is requested with the current ETag
- **THEN** the answer is 304 with no body

### Requirement: Track structure is separated from train state (Tier 1)

`GET /coaster/rides/{rideId}/track` SHALL return the track version (the first 16 hexadecimal characters of the design hash), the placed frame of each piece, the total length, the arc length of the platform stop point and the list of zones (kind, start arc length, end arc length). It SHALL NOT carry sample tables: the client composes the track from the cached catalogue geometry and the placed frames. A client SHALL reject train state whose track version differs from the structure it holds and SHALL refetch the structure.

#### Scenario: Client and server agree on length

- **WHEN** the client composes the starter coaster from the catalogue geometry and placed frames
- **THEN** its total length equals the server's total length within 1e-9 m

#### Scenario: Version mismatch

- **WHEN** a train state carries track version `a1b2` while the client holds `c3d4`
- **THEN** the client ignores the state and refetches the structure

### Requirement: Train state is replicated as distance and speed (Tier 1)

Each train SHALL replicate its arc length, signed speed, phase and occupied seat count through the ECS snapshot (`GET /simulation/snapshot`), and nothing else about its position. The client SHALL derive position, orientation and car positions from the compiled track.

#### Scenario: Snapshot fields

- **WHEN** a train entity is in the snapshot
- **THEN** its replicated component has the fields `s`, `v`, `phase` and `occupied`

### Requirement: The client dead-reckons between updates (Tier 1)

Between snapshots the client SHALL advance the arc length by speed times elapsed time, SHALL wrap it at the track length, and SHALL blend toward the next snapshot value over 100 ms. With updates at 5 Hz the position error against the server SHALL stay below 0.25 m over the starter lap.

#### Scenario: Starter lap

- **WHEN** the server trace of the starter lap is sampled at 5 Hz and the client dead-reckons from it
- **THEN** the largest difference to the 30 Hz server trace is below 0.25 m

### Requirement: The live stream is a quantised delta (Tier 2)

When the park stream of ADR game-0005 exists, train state SHALL be sent as a delta record of train index, arc length as an unsigned 32-bit integer in millimetres, speed as a signed 16-bit integer in 0.01 m/s and phase as 8 bits, at most 12 bytes per train per frame, and only for coasters inside the client's viewport. The client SHALL render 100 ms in the past and interpolate arc length linearly. A lost delta SHALL be repaired by the next keyframe.

#### Scenario: Quantisation error

- **WHEN** an arc length and a speed are encoded and decoded
- **THEN** the arc length differs by less than 1 mm and the speed by less than 0.005 m/s

#### Scenario: Bandwidth

- **WHEN** 500 trains are in view and all move
- **THEN** a frame is at most 6,000 bytes plus the stream header, and 30 frames per second are at most 180 KB/s

### Requirement: Test-run and replay use the same data (Tier 1)

The test-run result SHALL carry arc length per sample so the client replays the lap with the same composed track and the same camera code as the live ride, without any additional geometry request.

#### Scenario: Replay without a request

- **WHEN** a test result is displayed and replayed
- **THEN** no request other than the test-run request itself is made
