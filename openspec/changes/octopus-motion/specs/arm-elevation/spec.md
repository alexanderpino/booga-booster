# Spec Delta

## Purpose
Defines how the four main arms of the Great Mill rise and fall like an Octopus ride: the cam profile and its default parameters, the phase stagger between arms, the engagement and parking lifecycle, the rate, acceleration and clearance limits, and how the arms behave when the ride is stopped, braked, reversed or in emergency stop.

## ADDED Requirements

### Requirement: Arm elevation is a cam function of the mill angle

Each of the four main arms `k` (0 to 3) SHALL have an elevation angle `theta_k`, measured upward from the horizontal about the arm's hinge at the top of the central tower. While the arms are fully engaged (engagement factor 1) the elevation SHALL be a pure function of the mill angle `phi`: `theta_k = theta_base + A * cos(n * phi + k * Delta)`. The elevation SHALL NOT have a motor, axis or state of its own; it SHALL depend only on the mill angle (modulo one revolution) and on the engagement factor, never on the direction of rotation, the rotation speed or the elapsed time. The default parameters SHALL be: base elevation `theta_base` = 15 deg, amplitude `A` = 15 deg, cam order `n` = 1, phase stagger `Delta` = 180 deg, park elevation 0 deg. With the defaults the elevation of every arm lies between 0 deg and 30 deg, and the arm hinge sits 6.0 m above the ground.

#### Scenario: Default elevations at mill angle zero

- **WHEN** the arms are fully engaged and the mill angle is 0 deg
- **THEN** arms 0 and 2 are at 30.0 deg and arms 1 and 3 are at 0.0 deg
- **AND** the hub of arm 0 is 9.0 m above the ground with a horizontal reach of 5.196 m
- **AND** the hub of arm 1 is 6.0 m above the ground with a horizontal reach of 6.000 m

#### Scenario: Default elevations a quarter turn later

- **WHEN** the arms are fully engaged and the mill angle is 90 deg
- **THEN** all four arms are at 15.0 deg
- **AND** every hub is 7.553 m above the ground (within 1 mm) with a reach of 5.796 m (within 1 mm)

#### Scenario: Elevation is path independent

- **WHEN** the mill reaches the same angle (modulo 360 deg) once while turning forward and once after turning in reverse, with the arms fully engaged both times
- **THEN** every arm elevation is identical in both cases to within 1e-12 rad

#### Scenario: Elevation does not depend on speed

- **WHEN** the mill passes angle 45 deg at 3 rpm and again at 18 rpm with the arms fully engaged
- **THEN** every arm elevation is identical in both passes to within 1e-12 rad

### Requirement: Arms are phase-staggered so that adjacent arms alternate

The phase offset of arm `k` SHALL be `k * Delta` with `Delta` = 180 deg by default, so that opposite arms (0 and 2, 1 and 3) always have identical elevation and adjacent arms (0 and 1, 1 and 2, 2 and 3, 3 and 0) are always in antiphase, rising and falling alternately. For any configuration the system SHALL compute the elevation of every arm from the same single mill angle (no per-arm state).

#### Scenario: Opposite arms move in unison

- **WHEN** the arms are fully engaged at any mill angle
- **THEN** `theta_0` equals `theta_2` and `theta_1` equals `theta_3` to within 1e-12 rad

#### Scenario: Adjacent arms alternate

- **WHEN** the arms are fully engaged
- **THEN** `theta_0 + theta_1` equals `2 * theta_base` to within 1e-12 rad at every mill angle

#### Scenario: Symmetric load keeps the load-balance reading at zero

- **WHEN** all four hub assemblies carry the same mass and the arms are fully engaged at any mill angle
- **THEN** the horizontal offset of the combined centre of mass from the spin axis is zero to within 1e-9 m

### Requirement: Cam profile contract

The cam profile SHALL be a smooth periodic function `p(psi)` with period 360 deg, range [-1, 1], and a continuous first and second derivative. The default profile SHALL be the pure harmonic `p(psi) = cos(psi)`. Any other profile SHALL be rejected at configuration time unless it is band-limited to harmonics 1 to 3, so that the sum of the four arm elevations (and therefore the net vertical momentum of the arm assemblies) stays free of the first three harmonics by symmetry. The cam order `n` SHALL be a positive integer so that the elevation is continuous when the mill angle wraps from 360 deg to 0 deg.

#### Scenario: Default profile is the harmonic

- **WHEN** the profile is evaluated at 0, 90, 180 and 270 deg
- **THEN** it returns 1, 0, -1 and 0 respectively (within 1e-12)

#### Scenario: Non-integer cam order is rejected

- **WHEN** a configuration sets the cam order to 1.5
- **THEN** it is rejected with a validation error and the ride does not start

#### Scenario: Elevation is continuous at the wrap point

- **WHEN** the mill angle moves from 359.999 deg to 0.001 deg with the arms fully engaged
- **THEN** no arm elevation changes by more than 0.01 deg

### Requirement: Engagement factor raises and lowers the arms

The system SHALL scale the cam motion by an engagement factor `lambda` between 0 (arms parked) and 1 (arms fully engaged), so that `theta_k = theta_park + lambda * (theta_cam,k - theta_park)`. The factor SHALL follow a smooth-step curve `lambda = 3 s^2 - 2 s^3` of a progress value `s`. Engagement SHALL be observable as one of four states: `Parked` (`s` = 0), `Engaging` (`s` rising), `Engaged` (`s` = 1) and `Parking` (`s` falling). The engagement ramp SHALL take 6.0 s (rising) and the parking ramp 8.0 s (falling) for a full 0-to-1 or 1-to-0 travel at the fixed 1/120 s simulation step; a ramp that starts from a partial progress takes proportionally less time (progress changes at a constant 1/6 or 1/8 per second).

#### Scenario: Ride starts with the arms parked

- **WHEN** a ride is created
- **THEN** its engagement state is `Parked`, `lambda` is 0, and all four arms are at the park elevation of 0.0 deg

#### Scenario: Arms rise when the ride is started

- **WHEN** the ride enters `Started` with the mill at angle 0 deg
- **THEN** the engagement state is `Engaging`
- **AND** after 3.0 s of simulated time `lambda` is 0.5 and arm 0 is at 15.0 deg (within 0.05 deg)
- **AND** after 6.0 s the state is `Engaged`, `lambda` is 1 and arm 0 is at 30.0 deg (within 0.05 deg) if the mill angle is still 0 deg

#### Scenario: Engagement ramp is smooth

- **WHEN** the engagement progress rises from 0 to 1 over 6.0 s
- **THEN** `lambda` has zero slope at both ends and its peak slope is 0.25 per second

### Requirement: Arms hold their engagement while the ride brakes and park once it is at rest

While the ride is `Stopping` or in `EmergencyStop`, the engagement factor SHALL be held at its current value (no engage or park ramp) so that the arms keep following the cam while the mill decelerates. When the mill and all hubs are at rest in those states, the system SHALL start the parking ramp and lower the arms to the park elevation. The ride SHALL NOT proceed to `Offloading` until the arms are `Parked`. While the ride is `Idle`, `Loading`, `Safe` or `Offloading` the engagement SHALL be `Parked`.

#### Scenario: Emergency stop at full elevation

- **WHEN** the ride is `Started` with the arms `Engaged` and an Emergency Stop is requested
- **THEN** the arms keep following the cam while the mill decelerates, with no step change in any elevation
- **AND** when the mill and hubs reach rest the engagement state becomes `Parking`
- **AND** the ride remains in `EmergencyStop` until the engagement state is `Parked`
- **AND** the ride then becomes `Offloading`

#### Scenario: Stop requested while the arms are still rising

- **WHEN** the ride is `Started` and 2.0 s into `Engaging` (`s` = 0.333) and a stop is requested
- **THEN** the engagement progress is held at 0.333 until rest, then ramps down to 0

#### Scenario: Passengers cannot leave with raised arms

- **WHEN** the ride has come to rest but the arms are `Parking`
- **THEN** the safety restraints stay locked
- **AND** no passenger can unboard

### Requirement: Behaviour when braked, coasting or reversed

The cam constraint SHALL apply in every motion state. With the engine brake engaged, or with power cut and the ride coasting, the arms SHALL keep the elevation implied by the current mill angle and the held engagement factor. A direction reversal SHALL NOT cause any step in elevation: when the mill slows through rest and accelerates the other way, every arm retraces the same elevation curve in the opposite sense. A ride that comes to rest in `Started` (zero power, brake engaged) SHALL stay at the elevation implied by its stopping angle with the engagement unchanged.

#### Scenario: Engine brake applied at cruise

- **WHEN** the ride is `Started` at 15 rpm with the arms `Engaged` and the engine brake is engaged
- **THEN** every arm elevation stays within the cam range of 0 deg to 30 deg while the mill stops
- **AND** the final elevations equal the cam values for the final mill angle

#### Scenario: Reversal retraces the profile

- **WHEN** the mill is commanded Reverse while turning forward with the arms `Engaged`
- **THEN** no arm elevation changes by more than the rate limit times one step during the turnaround
- **AND** the elevation sequence while turning in reverse is the elevation sequence of the forward pass played backwards

#### Scenario: Resting in Started

- **WHEN** the ride is `Started`, the engine brake is engaged, and the mill has stopped at 90 deg
- **THEN** all four arms are at 15.0 deg
- **AND** the engagement state remains `Engaged`

### Requirement: Elevation rate and acceleration limits

For every mill speed within the hard speed cap (2.5 rad/s, both directions) and every mill angle, the arm elevation rate SHALL NOT exceed 0.85 rad/s (48.7 deg/s) and the arm elevation acceleration SHALL NOT exceed 2.5 rad/s^2 (143 deg/s^2), including the contribution of the engagement ramps and of the mill's own angular acceleration. With the default parameters the peak cam elevation rate at the speed cap is 0.655 rad/s and the peak steady cam acceleration at the speed cap is 1.64 rad/s^2. A parameter set that could exceed either limit SHALL be rejected at configuration time. If a non-finite elevation value, or a value beyond either limit, is ever computed at run time, the ride SHALL enter `EmergencyStop` through the same entry behaviour as an operator emergency stop.

#### Scenario: Default parameters respect the limits across the speed range

- **WHEN** the mill speed is swept from -2.5 to +2.5 rad/s in 0.05 rad/s steps and the mill angle over a full revolution in 1 deg steps with the arms `Engaged`
- **THEN** the peak elevation rate is at most 0.66 rad/s
- **AND** the peak steady elevation acceleration is at most 1.65 rad/s^2

#### Scenario: Aggressive cam order is rejected

- **WHEN** a configuration uses cam order 2 with the default 15 deg amplitude
- **THEN** it is rejected because the peak elevation rate at the speed cap (1.31 rad/s) exceeds 0.85 rad/s

#### Scenario: Non-finite elevation trips the emergency stop

- **WHEN** an elevation value becomes non-finite while the ride is `Started`
- **THEN** the ride enters `EmergencyStop` on that step, cuts mill and hub power and engages the brakes

### Requirement: Minimum vertical load

For every mill speed within the speed cap and every mill angle with the arms engaged, the vertical specific force at every gondola SHALL stay at or above 0.10 g. A cam parameter set that violates this SHALL be rejected at configuration time. With the default parameters the minimum vertical specific force at the speed cap is 0.13 g.

#### Scenario: Default cam never unloads the seat

- **WHEN** the mill turns at the speed cap of 2.5 rad/s with the arms `Engaged`
- **THEN** the vertical specific force never falls below 0.10 g at any mill angle

#### Scenario: Large amplitude is rejected

- **WHEN** a configuration uses cam order 1 with an amplitude of 25 deg and a base elevation of 25 deg
- **THEN** it is rejected because the minimum vertical specific force at the speed cap falls below 0.10 g or the tower clearance is violated

### Requirement: Clearance rule

The configured elevation range SHALL satisfy all of the following over every mill angle and engagement value, and a configuration that violates any of them SHALL be rejected: (1) the lowest point of any gondola (hub height minus 1.15 m) is at least 0.5 m above the ground; (2) the nearest point of any gondola to the spin axis (hub reach minus 3.0 m) is at least 1.55 m from the axis, so no gondola touches the tower housing; (3) the horizontal gap between the sweep circles (radius 3.1 m) of two neighbouring hubs is at least 1.0 m whenever their vertical envelopes overlap. With the default parameters the margins are 4.35 m, 0.65 m (at 30 deg, the worst case) and 2.0 m respectively.

#### Scenario: Default parameters satisfy the clearance rule

- **WHEN** the clearance rule is evaluated for the defaults over all mill angles in 1 deg steps and all engagement values in 0.05 steps
- **THEN** the minimum ground clearance is 4.85 m or more
- **AND** the minimum gondola distance to the spin axis is 2.196 m or more
- **AND** the minimum neighbouring sweep gap is 2.0 m or more where the vertical envelopes overlap

#### Scenario: Excessive upper elevation is rejected

- **WHEN** a configuration allows an arm elevation of 50 deg
- **THEN** it is rejected because the nearest gondola would be 0.857 m from the spin axis, below the 1.55 m minimum

#### Scenario: Excessive lower elevation is rejected

- **WHEN** a configuration allows an arm elevation of -50 deg
- **THEN** it is rejected because the lowest gondola point would be below 0.5 m above the ground

### Requirement: Cam parameters live in one place and are documented

Every number used by the cam, the engagement ramps, the limits and the clearance rule SHALL be a named constant in the single physical-parameters class of the digital-twin module, with its derivation in `docs/06-octopus-arm-elevation.md` and its default in `docs/appendix-parameters.md`. No literal geometry or limit value SHALL appear in the kinematics code.

#### Scenario: Derivation is traceable

- **WHEN** a constant of the cam, ramp, limit or clearance group is changed
- **THEN** a test that evaluates the documented derivation formulas with the same constants fails if the documented default no longer satisfies the limits
