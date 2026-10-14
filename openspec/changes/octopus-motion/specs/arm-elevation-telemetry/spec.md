# Spec Delta

## Purpose
Defines the additive telemetry contract that carries arm elevation, mill angle, arm engagement and vertical G on the existing 30 Hz server-sent-events stream, including its precision, size budget and compatibility with clients and servers that do not know the new fields.

## ADDED Requirements

### Requirement: Mill telemetry reports its angle, the arm engagement state and the four arms

The mill telemetry SHALL additionally report the mill angle in degrees in the range [0, 360), the arm engagement state (`Parked`, `Engaging`, `Engaged` or `Parking`), and a list of exactly four arm records ordered by arm index 0 to 3. Each arm record SHALL carry: the arm index; the elevation in degrees (upward positive, 0 = horizontal); the elevation rate in degrees per second (signed, computed analytically from the model and not by differencing frames); the hub height above the ground in metres; and the hub horizontal reach from the spin axis in metres. The mill angle SHALL use the same sign convention as the signed mill rpm, so that a positive rpm increases the angle.

#### Scenario: Frame while the arms are engaged

- **WHEN** a frame is emitted with the arms `Engaged` at mill angle 0 deg
- **THEN** the mill reports angle 0.00, engagement `Engaged` and four arm records
- **AND** arm 0 reports elevation 30.00, height 9.000 and reach 5.196
- **AND** arm 1 reports elevation 0.00, height 6.000 and reach 6.000

#### Scenario: Frame for a parked ride

- **WHEN** a frame is emitted while the ride is `Loading`
- **THEN** the engagement is `Parked`
- **AND** every arm reports elevation 0.00, rate 0.00, height 6.000 and reach 6.000

#### Scenario: Elevation rate sign follows the motion

- **WHEN** the mill turns forward and arm 0 is rising
- **THEN** arm 0 reports a positive elevation rate
- **AND** when the mill turns in reverse at the same angle the rate of arm 0 is negative with the same magnitude

### Requirement: Gondola telemetry reports vertical and total G

Each of the sixteen gondola records SHALL additionally report `VerticalG` (signed vertical specific force in g, +1.000 at rest) and `TotalG` (magnitude of the full specific force vector in g). The existing `ForwardG` and `LateralG` fields SHALL keep their meaning and values.

#### Scenario: Vertical G at rest

- **WHEN** a frame is emitted while the ride is at rest
- **THEN** every gondola reports `VerticalG` 1.000 and `TotalG` 1.000

#### Scenario: Existing horizontal G values are unchanged for a parked ride

- **WHEN** the same scripted run is replayed with the arms held `Parked`
- **THEN** every gondola's `ForwardG` and `LateralG` equal the values the pre-change simulation produced for that script, to within 0.001 g

### Requirement: Reported values are finite, rounded and coherent

Every numeric value of the new fields SHALL be finite. The mill angle and the arm elevation and rate SHALL be rounded to 0.01 (deg or deg/s), the height and reach to 0.001 m, and the G values to 0.001 g, in the telemetry projection only; the simulation state SHALL NOT be rounded. All new values in one frame SHALL come from the same simulation step as the rest of that frame.

#### Scenario: Rounding applies to the projection only

- **WHEN** the simulation runs for 60 s and a frame is projected at every step
- **THEN** the unrounded simulation state is identical to a run in which no frame was projected

#### Scenario: Frame fields come from one step

- **WHEN** a frame is emitted
- **THEN** its arm elevations equal the cam values for its own mill angle and engagement state to within the rounding step

### Requirement: Additive wire contract on the existing stream

The new fields SHALL be additional members of the existing record types and SHALL be serialized on the existing `GET /ride/telemetry/stream` endpoint with the existing event name, the existing member-naming convention (camelCase) and the existing enum encoding (numeric index, accepted by the client also as a name). No existing member SHALL be renamed, removed, retyped or reordered. The frame rate SHALL stay 30 Hz and the emission gating (frames only while the ride is not `Idle`) SHALL be unchanged. Constructing the existing records without the new members SHALL remain possible, with the new members taking the defaults: angle 0, engagement `Parked`, no arms, `VerticalG` 1.0 and `TotalG` 1.0.

#### Scenario: Old consumer ignores the new members

- **WHEN** a consumer that models only the pre-change members reads a new frame
- **THEN** it reads every pre-change member with the same values it would have received before this change

#### Scenario: New consumer reads a pre-change frame

- **WHEN** the new frontend receives a frame without the new members
- **THEN** it maps the mill angle to 0, the engagement to `Parked`, the arms to four parked arms at 0 deg and every `VerticalG` and `TotalG` to 1.0
- **AND** it does not throw and the other panels keep working

#### Scenario: Stream rate and gating unchanged

- **WHEN** the ride is `Idle`
- **THEN** no frames are emitted
- **AND** while the ride is active exactly one frame is emitted per 1/30 s tick of the injected time provider

### Requirement: Frame size budget

The serialized size of a frame SHALL grow by at most 25 % and by at most 2.0 KB compared with the same frame before this change, measured on a fully loaded ride with the arms engaged. The estimated growth is 1.3 KB (about 17 %).

#### Scenario: Size growth within budget

- **WHEN** a frame of a fully loaded, running ride is serialized before and after the change
- **THEN** the new frame is no more than 25 % and no more than 2,048 bytes larger
