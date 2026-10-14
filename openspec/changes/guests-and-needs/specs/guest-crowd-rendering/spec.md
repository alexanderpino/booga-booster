# Spec Delta

## Purpose

Defines how the browser turns the guest stream into a smooth crowd: buffering and interpolation, instanced drawing with levels of detail, mood tinting, culling and frame-time budgets, and a visible bandwidth meter. The client renders state it is given and never simulates guests.

## ADDED Requirements

### Requirement: The client renders about 100 ms in the past and interpolates

The client SHALL keep the last 6 received frames per guest in a ring and render each guest at `now - 100 ms`, interpolating position linearly between the two surrounding frames and heading by the shortest angle. If the render time is beyond the newest frame, it SHALL extrapolate for at most 33 ms and then hold. A guest first seen in a keyframe SHALL appear at its position without sliding from the origin.

#### Scenario: Smooth at 60 fps from 30 Hz data

- **WHEN** frames arrive at 30 Hz and the view renders at 60 fps
- **THEN** each guest's rendered position changes on every rendered frame, and no frame moves a guest by more than its speed times the frame time plus one quantum

#### Scenario: Extrapolation cap

- **WHEN** no frame arrives for 200 ms
- **THEN** guests extrapolate for 33 ms and then stand still until data returns

#### Scenario: Heading wrap

- **WHEN** a guest turns from heading 250 to 5 (256 steps per turn)
- **THEN** the rendered heading passes through 0 and not through 128

### Requirement: Mesh and animation work is instanced

All guests SHALL be drawn with instanced meshes: at most one instanced mesh per level of detail, each holding per-instance transform and colour, updated in place without creating objects per frame. Guest count SHALL NOT change the draw-call count. Per-instance data SHALL be updated only for guests whose rendered state changed.

#### Scenario: Draw calls independent of count

- **WHEN** 500 and then 5 000 guests are visible
- **THEN** the guest draw-call count is the same (at most 6)

#### Scenario: No per-frame allocation

- **WHEN** 1 000 frames are rendered with 2 000 guests
- **THEN** the guest renderer allocates no objects per frame in steady state (checked by a heap-growth test with a tolerance)

### Requirement: Three levels of detail by camera distance

Guests within 30 m of the camera SHALL use the high mesh (about 120 triangles, with a walking bob), from 30 to 90 m the medium mesh (about 24 triangles), from 90 to 250 m a camera-facing quad, and beyond 250 m SHALL not be drawn. Switching SHALL not cause a visible position jump.

#### Scenario: Level selection

- **WHEN** guests are at 10, 50, 150 and 400 m
- **THEN** they use the high mesh, the medium mesh, the quad and are culled respectively

#### Scenario: Triangle budget

- **WHEN** 5 000 guests are visible in a mixed scene
- **THEN** the guest triangle count is at most 400 000

### Requirement: Mood is shown as a tint and never as the only cue

Each guest SHALL be tinted by its mood tier (four steps from the colour-blind-safe palette defined in the design). The inspection panel and the list SHALL carry the same information in text so tint is never the sole carrier of meaning.

#### Scenario: Tier to tint

- **WHEN** the four mood tiers are rendered
- **THEN** four distinct tints are used, each with contrast ratio at least 3:1 against the ground colour

### Requirement: Frame-time budget

On the reference laptop used for the project benchmarks, guest decoding, interpolation and instance upload SHALL take at most 2 ms per rendered frame at 5 000 visible guests, and the full scene SHALL hold 60 fps. Decoding a 300-record delta SHALL take at most 1 ms and a 5 000-record keyframe at most 3 ms. Budgets are targets recorded in a measurement note; the build fails only on the machine-independent checks (no allocations, draw calls, triangle count).

#### Scenario: Decode benchmark

- **WHEN** the decode benchmark processes a 5 000-record keyframe
- **THEN** the measured time is recorded in the doc and the allocation count is 0 beyond the output buffers

### Requirement: A bandwidth meter shows the stream

A small meter SHALL show the current bytes per second, frames per second, guests in view and dropped frames, updated once per second, readable by screen readers as a live status that does not interrupt.

#### Scenario: Meter content

- **WHEN** the stream delivers 90 KB/s with 300 guests in view
- **THEN** the meter shows about 90 KB/s and 300 guests, and its status region is polite

### Requirement: The renderer tolerates connection loss

On socket close the renderer SHALL keep the last positions visible for 3 s dimmed, then clear them, and reconnect with exponential backoff (1 s, 2 s, 4 s, capped at 10 s). After reconnect the first keyframe SHALL replace all state.

#### Scenario: Reconnect

- **WHEN** the socket drops and returns after 5 s
- **THEN** guests were dimmed after the drop, cleared at 3 s, and repopulated by the keyframe after reconnect
