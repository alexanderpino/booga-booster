# Spec Delta

## Purpose
Defines the three-dimensional kinematics and dynamics of the elevating ride: the vertical hub motion, the vertical gondola acceleration and the G-force Z component, the cam-effective inertia and gravity cam torque of the mill, their energy and torque consequences, and the interaction with over-speed protection and single-rider loading.

## ADDED Requirements

### Requirement: Hub position is three-dimensional and the hub axis stays vertical

The centre of hub `k` SHALL be at horizontal distance `r_k = L cos(theta_k)` from the spin axis (with `L` = 6 m), at azimuth `phi + k * 90 deg`, and at height `z_k = H + L sin(theta_k)` above the ground (with `H` = 6.0 m the hinge height). The hub axis SHALL remain vertical for every elevation, so the hub plane stays horizontal, the four hub arms and their gondolas keep their horizontal geometry unchanged, and every gondola of a hub is at the hub's height. A gondola's position SHALL be the hub centre plus its horizontal arm offset.

#### Scenario: Hub plane stays horizontal at maximum elevation

- **WHEN** arm 0 is at 30 deg
- **THEN** the four gondola pivots of hub 0 are all at height 9.0 m
- **AND** their horizontal distances from the hub axis are all 2.0 m

#### Scenario: Parked ride equals the planar ride

- **WHEN** the engagement factor is 0
- **THEN** every hub is at reach 6.000 m and height 6.0 m
- **AND** every horizontal output of the simulation (positions, inertia, gondola forward and lateral G, load-balance reading) equals the pre-change planar value to within 1e-12

### Requirement: Gondolas feel the hub's vertical acceleration

The vertical acceleration of every gondola of hub `k` SHALL equal the vertical acceleration of the hub centre, `z'' = L cos(theta) theta'' - L sin(theta) theta'^2`, where `theta'` and `theta''` include the contribution of the mill angular velocity and acceleration and of the engagement ramp. Each gondola SHALL report a vertical specific force `VerticalG = (g + z'') / g`, where `g` = 9.81 m/s^2. `VerticalG` SHALL be signed and SHALL equal exactly 1.000 whenever the mill is at rest and the engagement is not changing. `VerticalG` SHALL be independent of the load on the ride.

#### Scenario: Rest reads one g

- **WHEN** the mill is at rest with the arms in any pose and the engagement is not changing
- **THEN** every one of the 32 seats reads a vertical specific force of 1.000 g

#### Scenario: Vertical G at the over-speed warn speed

- **WHEN** the mill turns steadily at 15 rpm (1.571 rad/s) with the arms `Engaged`
- **THEN** over a full revolution the vertical specific force of a gondola varies between 0.65 g and 1.41 g

#### Scenario: Vertical G at the over-speed trip speed

- **WHEN** the mill turns steadily at 18 rpm (1.885 rad/s) with the arms `Engaged`
- **THEN** over a full revolution the vertical specific force of a gondola varies between 0.50 g and 1.60 g

#### Scenario: Vertical G at the speed cap

- **WHEN** the mill turns steadily at the speed cap of 2.5 rad/s with the arms `Engaged`
- **THEN** over a full revolution the vertical specific force varies between 0.10 g and 2.05 g

#### Scenario: Load does not change the vertical motion

- **WHEN** two rides with different passenger loads reach the same mill angle, speed and acceleration with the arms `Engaged`
- **THEN** their arm elevations and their vertical specific forces are identical

### Requirement: Total G includes the vertical component

Each gondola SHALL report `TotalG`, the magnitude of the specific force vector `sqrt(ForwardG^2 + LateralG^2 + VerticalG^2)`. The existing `ForwardG` and `LateralG` SHALL keep their definition as the horizontal specific-force components along and across the gondola's facing direction. At rest `TotalG` SHALL be 1.000.

#### Scenario: Total G at rest

- **WHEN** the ride is at rest
- **THEN** every gondola reports `TotalG` of 1.000 and `ForwardG` and `LateralG` of 0.000

#### Scenario: Total G combines the three components

- **WHEN** a gondola reports `ForwardG` 1.0, `LateralG` 2.0 and `VerticalG` 2.0
- **THEN** it reports `TotalG` of 3.0

### Requirement: Horizontal field includes the radial and Coriolis terms of the moving hub

The horizontal acceleration field acting on a gondola SHALL include, in addition to the existing centripetal terms evaluated at the instantaneous reach `r_k`, the radial term `r_k''` (along the outward radial of the hub) and the tangential Coriolis term `2 r_k' phi'` (perpendicular to it), where `r_k = L cos(theta_k)`. The tangential Euler term `r phi''` SHALL remain excluded, as in the existing steady-state model. For a parked ride the added terms SHALL be zero.

#### Scenario: Added terms at the over-speed trip speed

- **WHEN** the mill turns steadily at 18 rpm with the arms `Engaged`
- **THEN** over a full revolution the radial term `r''` of a hub never exceeds 2.85 m/s^2 in magnitude
- **AND** the tangential term `2 r' phi'` never exceeds 3.75 m/s^2 in magnitude

#### Scenario: Gondola yaw torque is unaffected by the vertical motion

- **WHEN** a gondola swings with a vertical acceleration of 5 m/s^2
- **THEN** the vertical acceleration contributes no yaw torque; only the horizontal field drives the swing

### Requirement: The mill dynamics use the cam-effective inertia and gravity cam torque

While the arms are engaged the mill rotation SHALL obey `I_eff(phi) phi'' = tau_drive - tau_loss - (1/2) I_eff'(phi) phi'^2 - V'(phi)`, where, with `J_k = (m_arm/3 + M_k) L^2` for arm `k` (arm rod mass `m_arm` = 800 kg, hub assembly mass `M_k` including gondolas and passengers), `I_eff = sum_k J_k (cos^2(theta_k) + (d theta_k / d phi)^2)` and `V = g sum_k (m_arm/2 + M_k) L sin(theta_k)`. The drive, friction, aerodynamic drag and brake torques SHALL be unchanged. Hub rotation dynamics SHALL be unchanged. With the engagement factor at 0 the equation SHALL reduce exactly to the planar equation `I phi'' = tau_drive - tau_loss`.

#### Scenario: Effective inertia is lower than the planar value on average

- **WHEN** the arms are `Engaged` and the mill makes a full revolution under any legal load
- **THEN** the mean of `I_eff` is between 0.93 and 0.95 times the planar inertia
- **AND** `I_eff` varies by at most 8 % of its mean in either direction, twice per revolution for cam order 1

#### Scenario: Spin-up with the cam is slightly faster and ends at the same speed

- **WHEN** a fully loaded ride with the arms `Engaged` is driven from rest at 70 % mill power
- **THEN** it reaches 90 % of its own steady speed between 1.0 s and 3.5 s sooner than the planar ride does (17.7 s for the planar ride)
- **AND** its mean steady speed is within 0.5 % of the planar ride's
- **AND** its instantaneous speed varies by at most 5 % either side of the mean (about 3.6 % for the default cam)

#### Scenario: Reversal still passes through rest

- **WHEN** the mill is commanded Reverse while turning forward with the arms `Engaged`
- **THEN** it decelerates through a standstill and accelerates the other way, with no discontinuity in speed larger than 0.05 rad/s per step

#### Scenario: Braking time bound still holds

- **WHEN** the engine brake is engaged at the speed cap with the arms `Engaged`, under empty and full loads
- **THEN** the mill and hubs come to rest within the same time bound the engine-braking tests assert today, without relaxing that bound

### Requirement: Mechanical energy is conserved by the cam constraint

The mechanical energy of the mill with engaged arms, `E = (1/2) I_eff(phi) phi'^2 + V(phi)`, SHALL be conserved with zero drive and zero losses, and SHALL decrease with losses. Over 60 s of simulated time at the fixed 1/120 s step, starting at 0.5, 1.8 and 2.5 rad/s with a full load, the energy SHALL stay within 3 % of the initial kinetic energy `(1/2) I_eff phi'^2`. With zero drive and the normal losses the energy sampled every 5 s SHALL decrease strictly and SHALL never exceed 102 % of its initial value.

#### Scenario: Frictionless coast conserves energy

- **WHEN** the mill with engaged arms and a full load coasts with zero drive and zero friction for 60 s starting at 1.8 rad/s
- **THEN** the energy deviates from its initial value by less than 3 % of the initial kinetic energy at every step

#### Scenario: Coasting with losses loses energy

- **WHEN** the mill coasts with the normal friction for 60 s from 1.8 rad/s with engaged arms
- **THEN** the energy sampled at 5 s intervals decreases strictly
- **AND** it never exceeds 102 % of the initial energy

#### Scenario: Engagement changes conserve the horizontal angular momentum

- **WHEN** the engagement factor changes during one step while the drive torque is zero
- **THEN** the horizontal angular momentum of the arm assemblies is the same before and after the change to within 1e-9 relative, so the mill speeds up as the arms rise and slows down as they lower

### Requirement: The mill stops without chatter and can always be started

With zero motor drive, a coasting mill SHALL come to rest and stay at rest where the gravity cam torque `|V'|` is below the static friction (Coulomb friction plus any brake torque); it SHALL NOT oscillate in sign around zero speed. For a symmetric load the amplitude of `V'` SHALL be at most 6,000 N m (60 % of the mill Coulomb friction). For any legal load the amplitude of `V'` SHALL be at most 70 % of the mill stall torque, so the ride can always be spun up from rest. With an asymmetric load the amplitude of `V'` MAY exceed the Coulomb friction, in which case a mill without brake torque SHALL settle at a stable equilibrium of `V` (a point where `V'` = 0 and `V''` > 0) rather than where it ran out of energy.

#### Scenario: Symmetric coast stops and stays stopped

- **WHEN** a symmetric fully loaded ride with engaged arms coasts down from 1.0 rad/s with the brake released
- **THEN** the mill speed becomes exactly 0 and stays 0 for the next 10 s of simulated time

#### Scenario: Asymmetric load settles at a potential minimum

- **WHEN** hubs 0 and 2 are fully loaded, hubs 1 and 3 are empty, the brake is released, power is zero, and the mill coasts from 0.3 rad/s
- **THEN** it comes to rest within 0.05 rad of a mill angle where `V'` = 0 and `V''` > 0

#### Scenario: Start from rest under the heaviest legal asymmetry

- **WHEN** the heaviest legal asymmetric load is applied (hubs 0 and 2 at 8 x 150 kg passengers each, hubs 1 and 3 empty) and full mill power is commanded from a mill angle where `V'` is at its extreme
- **THEN** the mill speed becomes positive within 2 s and the mill completes a full revolution without stalling

### Requirement: Symmetric arms keep the net vertical momentum bounded

With the arms `Engaged` and equal hub assembly masses, the sum of the four arm heights SHALL equal `4 H + 4 L sin(theta_base) cos(A cos(n phi))` to within 1e-9 m (swing 0.212 m), the combined centre of mass SHALL stay on the spin axis, and the sum of the four hub vertical speeds SHALL satisfy `|sum z_k'| <= 0.25 |phi'|` metres per second per rad/s. With unequal hub masses the net vertical momentum `sum M_k z_k'` SHALL satisfy `|sum M_k z_k'| <= |M_0 + M_2 - M_1 - M_3| * u_max + M_max * 0.25 |phi'|`, where `u_max` is the peak vertical speed of one hub.

#### Scenario: Equal loads cancel to second order

- **WHEN** the arms are `Engaged` at 15 rpm with all hub assemblies at 2,080 kg
- **THEN** the net vertical momentum never exceeds 2,080 kg x 0.25 x 1.571 rad/s = 817 kg m/s although each hub alone reaches 2.4 m/s

#### Scenario: Alternating pairs unbalance the vertical momentum

- **WHEN** the arms are `Engaged` at 15 rpm, hubs 0 and 2 carry 8 x 75 kg passengers (2,080 kg) and hubs 1 and 3 are empty (1,480 kg)
- **THEN** the net vertical momentum never exceeds 3,700 kg m/s

### Requirement: Load-balance reading uses the actual reach

The load-balance reading (the horizontal offset of the combined centre of mass) SHALL use the instantaneous reach `r_k` of each hub assembly instead of the fixed arm length. The dispatch interlock SHALL keep being evaluated when the arms are parked, where `r_k` = 6 m and the reading equals the pre-change value. A symmetric load SHALL read exactly zero at every pose.

#### Scenario: One extra passenger while engaged

- **WHEN** hub 0 carries one extra 75 kg passenger compared with the other hubs (1,555 kg vs 1,480 kg) and the arms are `Engaged` at mill angle 0 deg (arm 0 at 30 deg)
- **THEN** the load-balance reading is 65.0 mm (within 0.5 mm)
- **AND** at mill angle 180 deg (arm 0 at 0 deg) it is 75.1 mm (within 0.5 mm)

#### Scenario: Interlock decision is unchanged at rest

- **WHEN** the ride is `Safe` with the arms parked
- **THEN** the load-balance reading and the unbalanced-load interlock give the same results as before this change for every load pattern

### Requirement: Interaction with over-speed protection

The elevation limits and the vertical-load limit SHALL be satisfied for every legal mill speed up to the hard speed cap (2.5 rad/s), so they hold whether or not an over-speed trip is configured below it. The mill and hub rotation speed reported for over-speed classification SHALL stay the instantaneous signed rpm; the cam adds a periodic speed ripple of at most 5 % either side of the mean at constant power, and the classification SHALL NOT be changed to compensate. When an over-speed trip enters `EmergencyStop` with the arms `Engaged`, the arms SHALL follow the stop sequence of this change (hold, brake, park). Arm elevation SHALL NOT itself raise the stress level. The telemetry members added by this change and by the over-speed protection change SHALL be independent trailing additions to the same records, in any application order.

#### Scenario: Margins at the over-speed trip speed

- **WHEN** the mill turns at the 18 rpm trip speed with the arms `Engaged`
- **THEN** the peak elevation rate is at most 0.50 rad/s (59 % of the 0.85 rad/s limit)
- **AND** the minimum vertical specific force is at least 0.50 g

#### Scenario: Trip with raised arms

- **WHEN** an over-speed trip enters `EmergencyStop` while the arms are `Engaged`
- **THEN** the mill brakes to rest with the arms following the cam
- **AND** the arms then park and only then does the ride enter `Offloading`

#### Scenario: Speed ripple does not flip a settled classification

- **WHEN** the mill is held at steady power so that its mean speed is 14.0 rpm
- **THEN** its instantaneous speed never exceeds 14.7 rpm, so a 15 rpm warn limit is not crossed by the ripple alone

### Requirement: Interaction with single-rider loading

Elevation, rate and vertical specific force SHALL NOT depend on which seats are occupied. Partial loads from single-rider filling SHALL change only the hub assembly masses, and therefore `I_eff`, `V'`, the load-balance reading and the net vertical momentum, within the bounds of this capability.

#### Scenario: Single rider fills a gap

- **WHEN** a single rider fills the second seat of a gondola on hub 3
- **THEN** the cam elevation of every arm is unchanged
- **AND** only the hub 3 assembly mass, the mill inertia and the load-balance reading change

### Requirement: Determinism and golden master

The elevated simulation SHALL be a pure function of its state and the fixed 1/120 s step, with no clock and no random source in the tick. A scripted 60 s scenario SHALL produce bit-identical state in two runs of the same process, and SHALL match a checked-in golden record at every simulated second to within an absolute tolerance of 1e-9 (relative 1e-9 for values above 1,000).

#### Scenario: Two runs are identical

- **WHEN** the scripted 60 s scenario runs twice from a fresh ride
- **THEN** the mill angle, mill speed, all four arm elevations and all 16 vertical G values are bit-identical at every step

#### Scenario: Run matches the golden record

- **WHEN** the scripted scenario runs (fully loaded with 32 passengers of 75 kg; 70 % mill power and 40 % hub power from the start of `Started`; mill direction reversed at 30 s; stop requested at 50 s)
- **THEN** the mill angle, mill speed, four arm elevations, engagement progress, mechanical energy, hub 0 vertical acceleration and gondola 0 `VerticalG` at each whole simulated second match the golden record

#### Scenario: Asymmetric seeded variant

- **WHEN** the same script runs with passenger weights drawn from a generator seeded with 20261014 in the range 60 to 90 kg and the load satisfies the dispatch interlocks
- **THEN** the run matches its own golden record, and a change of the seed changes the result
