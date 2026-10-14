## MODIFIED Requirements

### Requirement: Full telemetry payload

Each streamed frame SHALL be a complete ride snapshot, not a partial delta. It SHALL include, per the `RideTelemetry` contract: the ride lifecycle state; the total count of boarded passengers currently on the ride; the mill's commanded power, sensed RPM, angle, arm engagement state, total and passenger load, imbalance and balance/overload flags, and the elevation, elevation rate, height and reach of each of its four arms; each of the four hubs' power, RPM and load; and each of the sixteen gondolas' brake state, yaw angle, RPM, forward, lateral and vertical G, total G, load, and per-seat weight, restraint state, whether the seat is occupied, and whether its restraint is secured. The members added for the arms, the mill angle and the vertical and total G SHALL be additive: a consumer that does not know them SHALL still read every other member unchanged (see the `arm-elevation-telemetry` capability).

#### Scenario: Frame carries mill and hub engine readings

- **WHEN** a telemetry frame is emitted while the ride is running
- **THEN** it contains the mill's power and RPM
- **AND** it contains the power and RPM of all four hubs

#### Scenario: Frame carries cart and seat state

- **WHEN** a telemetry frame is emitted while the ride is running
- **THEN** each gondola's brake, angle, RPM, forward/lateral/vertical G, total G and load are present
- **AND** each seat's measured weight and restraint state are present
- **AND** each seat reports whether it is occupied and whether its restraint is secured

#### Scenario: Frame carries the boarded-passenger count

- **WHEN** a telemetry frame is emitted while passengers are on the ride
- **THEN** the frame carries the total count of boarded passengers
- **AND** the count equals the number of occupied seats across all gondolas

#### Scenario: Frame carries the mill angle and the four arms

- **WHEN** a telemetry frame is emitted while the ride is running
- **THEN** the mill reports its angle and its arm engagement state
- **AND** the mill reports exactly four arm records, each with elevation, elevation rate, height and reach
