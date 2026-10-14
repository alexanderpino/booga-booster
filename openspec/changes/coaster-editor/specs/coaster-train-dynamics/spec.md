# Spec Delta

## Purpose

Defines the physics of a train moving along a compiled track at the fixed 1/120 s step: an energy-based one-dimensional model with gravity from an exact potential, rolling resistance, quadratic drag, lift, brake and station zones, rest and rollback behaviour, and the invariant that mechanical energy cannot grow without a drive zone.

## ADDED Requirements

### Requirement: The train state is distance and signed speed, stepped at 1/120 s (Tier 1)

A train SHALL be described by its arc-length position S of the lead car centre, its signed speed V along the direction of increasing arc length, its car count, its mass and its phase. One simulation step SHALL advance the state by exactly 1/120 s and SHALL be a pure function of the state, the compiled track, the train parameters and the dispatch inputs; it SHALL read no clock and no random source and SHALL allocate no managed memory.

#### Scenario: Pure step

- **WHEN** the same state is stepped twice from copies
- **THEN** the two results are bit-identical

#### Scenario: No allocation

- **WHEN** 100,000 steps run after a warm-up
- **THEN** the allocated bytes on the thread are unchanged

### Requirement: Gravity acts through the car-averaged height (Tier 1)

The potential of a train SHALL be m g Hbar(S), where Hbar is the piecewise linear car-averaged height table and g = 9.81 m/s^2. Within one sample interval the gravity acceleration along the track SHALL be the constant -g x (Hbar_{i+1} - Hbar_i) / (S_{i+1} - S_i). A step SHALL cross sample intervals using the closed-form motion for constant acceleration inside each interval, so the work-energy balance holds exactly up to rounding, and a train SHALL reverse correctly at a turning point inside an interval.

#### Scenario: Frictionless descent

- **WHEN** a 1-car train with rolling resistance and drag disabled is released from rest on five `slope:-2` pieces (22.36 m of track, 10.0 m of height)
- **THEN** its speed at the bottom is 14.007 m/s within 0.05 m/s and it arrives after 3.193 s within 0.03 s

#### Scenario: Valley symmetry

- **WHEN** a frictionless 1-car train is released from rest at height 4.0 m on one side of a symmetric valley
- **THEN** it comes to rest on the other side at 4.0 m within 2 mm of height and it returns to its start after the same time

### Requirement: Rolling resistance and drag only remove energy (Tier 1)

Rolling resistance SHALL decelerate the train by mu g cos(theta) toward zero speed with mu = 0.01 and SHALL NOT reverse its direction. Aerodynamic drag SHALL be quadratic, a = -c V |V| with c = rho CdA / (2 m), rho = 1.225 kg/m^3, and CdA = 1.0 m^2 for the first car plus 0.25 m^2 for each further car, solved exactly over the step as V / (1 + c |V| dt). A heavier train SHALL lose less to drag. The values are defaults held in the parameter set; tests MAY override them.

#### Scenario: Drag only

- **WHEN** a 600 kg 1-car train coasts on flat track at 20 m/s with rolling resistance disabled for 10 s
- **THEN** its speed is 16.609 m/s within 1e-3 m/s

#### Scenario: Rolling only

- **WHEN** the train coasts on flat track at 5 m/s with drag disabled
- **THEN** it stops after 127.4 m within 0.1 m and stays at exactly zero speed

#### Scenario: Mass and drag

- **WHEN** a 600 kg and a 1,200 kg train start at the same speed on flat track with drag enabled
- **THEN** the heavier train is faster after 10 s

### Requirement: A resting train stays at rest unless gravity wins (Tier 1)

A train at rest SHALL stay at rest (speed exactly 0) while the magnitude of the gravity acceleration is not larger than the rolling deceleration mu g cos(theta). Otherwise it SHALL start to move downhill. Static and kinetic resistance are the same coefficient.

#### Scenario: Flat

- **WHEN** a train at rest on flat track is stepped for 10 s
- **THEN** its speed is exactly 0 and its position is unchanged

#### Scenario: Slope

- **WHEN** a train at rest on `slope:-1` is stepped for 1 s
- **THEN** its speed is 2.28 m/s within 0.05 m/s, pointing downhill

### Requirement: A lift constrains the speed of the train upward (Tier 1)

While the lead car is in a lift zone the speed SHALL be raised toward the lift speed of 2.5 m/s at no more than 1.5 m/s^2 whenever the physics alone would leave it lower, independent of gravity. A faster train SHALL keep its speed. The speed SHALL NOT become negative in a lift zone (anti-rollback). The energy supplied by the lift SHALL be accumulated as `EnergyAdded` in joules.

#### Scenario: Ramp

- **WHEN** a train at rest enters a lift zone on a slope
- **THEN** its speed reaches 2.5 m/s after 200 steps (1.667 s) within one step and stays at 2.5 m/s until the zone ends

#### Scenario: No rollback on the lift

- **WHEN** the lift is stopped by a fault while the train is on it
- **THEN** the train holds at speed 0 and does not roll back

#### Scenario: Energy accounting

- **WHEN** a train climbs a lift
- **THEN** `EnergyAdded` is at least m g times the height gained

### Requirement: Trim brakes cap the speed (Tier 1)

While the lead car is in a trim brake zone the speed SHALL be reduced toward 3.0 m/s at no more than 3.0 m/s^2, and SHALL NOT be changed if it is lower. The energy removed SHALL be accumulated as `EnergyBraked`.

#### Scenario: Short brake run

- **WHEN** a train enters a 16.0 m trim brake zone at 10 m/s
- **THEN** it leaves at 3.0 m/s

#### Scenario: Brake too short

- **WHEN** a train enters the same zone at 12 m/s
- **THEN** it leaves at 6.93 m/s within 0.05 m/s because the deceleration limit binds

### Requirement: The station stops the train at a fixed point (Tier 1)

The station piece SHALL define a stop point at 11.0 m from its start. Inside the platform the speed SHALL be limited to sqrt(2 a d) with a = 2.0 m/s^2 and d the distance to the stop point, reduced at no more than 2.0 m/s^2, and the train SHALL stop with its lead car within 0.05 m of the stop point. A train that cannot be slowed enough SHALL pass the stop point and be flagged `Overshoot`. Leaving the station SHALL use a drive that raises the speed to 2.5 m/s at 1.5 m/s^2.

#### Scenario: Normal arrival

- **WHEN** the lead car enters the platform at 3.0 m/s
- **THEN** the train stops within 0.05 m of the stop point and its phase becomes Unloading

#### Scenario: Too fast

- **WHEN** the lead car enters the platform at 8.0 m/s
- **THEN** the train passes the stop point and `Overshoot` is set

### Requirement: Mechanical energy never increases without a drive zone (Tier 1)

Define E = 1/2 m V^2 + m g Hbar(S). For every step in which the lead car and the train are outside lift and station-drive zones, E after the step SHALL NOT exceed E before the step by more than 1e-9 x max(1, E) joules. With resistance and drag disabled, E SHALL be constant to the same tolerance. This SHALL be checked on random tracks, random states and random train parameters.

#### Scenario: Random tracks

- **WHEN** 200 seeded random open tracks are run from random states with no drive zones
- **THEN** E never rises beyond the tolerance in any step

#### Scenario: Lift is the only source

- **WHEN** a lap is run on the starter coaster and E is summed over steps outside drive zones
- **THEN** the total change is not positive, and the total rise of E over the lap is not more than `EnergyAdded`

### Requirement: Trajectory does not depend on mass when only gravity acts (Tier 1)

With drag disabled and rolling resistance independent of mass, the trajectory (S and V per step) SHALL be identical for trains of different mass on the same track.

#### Scenario: Equivalence

- **WHEN** a 600 kg and a 2,400 kg train run the starter layout with drag disabled
- **THEN** S and V are bit-identical at every step for 60 s

### Requirement: Speed is capped and open ends absorb energy (Tier 1)

The speed magnitude SHALL NOT exceed 40 m/s; a step that would exceed it SHALL clamp it and set the flag `Overspeed`. On an open track (used only by tests and tools) the position SHALL be clamped to the track ends with speed set to 0, and the lost energy SHALL count as dissipation.

#### Scenario: Cap

- **WHEN** a step would produce 41 m/s
- **THEN** the speed is 40 m/s and `Overspeed` is set

#### Scenario: End stop

- **WHEN** a train reaches the end of an open track at 5 m/s
- **THEN** S equals the track length, V is 0, and E has not increased

### Requirement: Invalid inputs are rejected (Tier 1)

Train parameters with a non-positive mass, car count outside 1 to 8, or non-finite values SHALL be rejected when the train is created, with a message naming the field. A step SHALL never produce a non-finite state; if one would result the train SHALL be stopped and flagged `Fault`.

#### Scenario: Bad mass

- **WHEN** a train is created with mass 0
- **THEN** creation fails naming the mass field
