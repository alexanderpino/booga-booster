# Spec Delta

## Purpose

Defines the binary stream that carries guest positions and states from the authoritative simulation to the browser: frame types, record layout, quantisation, interest management, distance tiers, loss recovery and the bandwidth budget. It implements the guest part of the state-sync decision in ADR game-0005.

## ADDED Requirements

### Requirement: The stream is a binary WebSocket channel at 30 Hz

The server SHALL expose one WebSocket endpoint under the park route that sends binary frames every fourth tick (30 Hz) to each connected client and accepts small client messages. Text, JSON and base64 SHALL NOT be used for guest frames. The final transport choice SHALL be confirmed by the apply-time spike required by ADR game-0005 and recorded in the design.

#### Scenario: Frame cadence

- **WHEN** a client is connected for 10 simulated seconds
- **THEN** it receives 300 frames (one per fourth tick), each with a strictly increasing sequence number

#### Scenario: No connected client

- **WHEN** no client is connected
- **THEN** no frame is encoded

### Requirement: Frames have a fixed header and typed bodies

Every frame SHALL start with: a version byte (1), a type byte (1 keyframe, 2 delta), a 32-bit sequence number, a 32-bit simulation tick and a 16-bit record count, little-endian. A keyframe SHALL list all guests in the client's interest set. A delta SHALL list guests whose quantised record changed since the last frame sent to that client, followed by a 16-bit removed count and the removed ids.

#### Scenario: Keyframe on connect

- **WHEN** a client connects
- **THEN** its first frame is a keyframe, and its type byte is 1

#### Scenario: Delta carries only changes

- **WHEN** 4 of 300 visible guests change quantised state between frames
- **THEN** the delta contains exactly 4 records

#### Scenario: Removal is explicit

- **WHEN** a visible guest leaves the park
- **THEN** the next delta lists its id in the removed list

### Requirement: Each guest record is 10 bytes

A record SHALL be: a 32-bit id (the low 22 bits entity index, the high 10 bits the low bits of the generation), x and z as unsigned 16-bit positions quantised over the park extent, heading as an unsigned 8-bit angle in 256 steps, and a state byte (bits 0 to 3 intent kind, bits 4 to 5 mood tier, bit 6 carrying an item, bit 7 sick). Guests riding inside a ride SHALL NOT be streamed. Height SHALL NOT be streamed; the client derives it from the terrain.

#### Scenario: Record size

- **WHEN** a frame with 300 records is encoded
- **THEN** the body is exactly 3 000 bytes plus the removed list

#### Scenario: Quantisation error

- **WHEN** the park extent is 512 m
- **THEN** the position quantum is 512/65 536 = 7.8 mm and decoding after encoding differs from the original by at most 7.8 mm per axis

#### Scenario: Generation reuse is visible

- **WHEN** an entity index is reused with a new generation
- **THEN** the id differs in the generation bits so the client treats it as a new guest

#### Scenario: Riders are hidden

- **WHEN** a guest is riding
- **THEN** it is absent from keyframes and appears in the removed list of the delta in which it boards

### Requirement: Interest management sends what the client can see

A client SHALL send view messages (camera focus x z and half extents) at most 10 per second. The server SHALL send guests inside the viewport expanded by 10 % and SHALL classify them by distance to the focus: near (up to 60 m) every frame, mid (up to 150 m) every third frame, far every sixth frame. A guest entering the interest set SHALL be sent at once; leaving it SHALL appear in the removed list.

#### Scenario: Pan

- **WHEN** the viewport moves so 40 guests enter and 35 leave
- **THEN** the next frame contains 40 new records and removes 35

#### Scenario: Tier cadence

- **WHEN** a guest stays in the far tier for 12 frames
- **THEN** it is sent in 2 of them (every sixth frame)

#### Scenario: Out of view

- **WHEN** a guest is outside the expanded viewport
- **THEN** no record for it is sent

### Requirement: Keyframes bound error and resynchronise

The server SHALL send a keyframe every 5 s, on connect, on a client request message, and after any skipped frame. If a client socket's send buffer holds more than one unsent frame, the server SHALL discard the older frame (latest wins), count it, and make the next frame a keyframe. The client SHALL detect a sequence gap, keep interpolating and request a keyframe.

#### Scenario: Slow consumer

- **WHEN** a client does not read for 500 ms
- **THEN** the server keeps at most one pending frame, counts the discarded ones, and sends a keyframe when the client resumes

#### Scenario: Five percent loss

- **WHEN** a test harness drops 5 % of delta frames
- **THEN** after the next keyframe the client's guest set and positions equal the server's within one quantum, with no permanent divergence

### Requirement: Encode and decode round-trip exactly

Decoding any encoded frame SHALL reproduce the encoded records bit for bit, for seeded random frames of up to 10 000 records, and a decoder SHALL reject unknown versions, truncated frames and record counts that exceed the body with a named error rather than reading out of bounds.

#### Scenario: Round trip

- **WHEN** 1 000 seeded random frames are encoded and decoded
- **THEN** every decoded record equals its source

#### Scenario: Truncated frame

- **WHEN** a frame is cut in the middle of a record
- **THEN** the decoder throws a named format error and produces no records

### Requirement: Bandwidth stays within budget

With 5 000 guests, the default tier cadence and a client view containing 300 near guests, 700 mid and 4 000 far, steady-state payload SHALL be at most 400 KB/s (30 x 300 + 10 x 700 + 5 x 4 000 = 36 000 records per second x 10 bytes = 360 KB/s plus headers). With a zoomed view of 300 near guests only, payload SHALL be at most 100 KB/s (ADR game-0005 target). The server SHALL expose bytes per second per client as a metric.

#### Scenario: Zoomed view

- **WHEN** 5 000 guests walk and a client's viewport holds 300 near guests
- **THEN** the measured payload over 10 s is at most 100 KB/s

#### Scenario: Whole park

- **WHEN** the viewport contains the whole park with the stated tier mix
- **THEN** the measured payload over 10 s is at most 400 KB/s

#### Scenario: Metric

- **WHEN** a client is connected
- **THEN** the bytes-per-second metric for that client is reported
