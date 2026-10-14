## MODIFIED Requirements

### Requirement: Automatic condition-driven transitions

The simulation SHALL perform the following transitions automatically when their physical condition is met, without an operator command, each time the ride advances:

- **Safe → Loading** when the ride is no longer safe (e.g. a secured restraint opens or the load becomes unbalanced).
- **Stopping → Offloading** when the ride has come to a complete rest and its arms are parked.
- **Emergency Stop → Offloading** when the ride has come to a complete rest and its arms are parked; entering `Offloading` releases the safety constraints.
- **Offloading → Idle** when the last rider has left (every seat is empty).

#### Scenario: Safe demotes to Loading when safety is lost

- **WHEN** the ride is `Safe` and a secured restraint subsequently opens
- **THEN** on the next advance the ride's state becomes `Loading`

#### Scenario: Stopping settles into Offloading at rest

- **WHEN** the ride is `Stopping` and its rotation has reached a complete rest and its arms are parked
- **THEN** the ride's state becomes `Offloading`

#### Scenario: Stopping waits for the arms

- **WHEN** the ride is `Stopping`, its rotation has reached a complete rest, and its arms are still being lowered
- **THEN** the ride's state remains `Stopping`
- **AND** on the first advance after the arms are parked the state becomes `Offloading`

#### Scenario: Offloading returns to Idle when empty

- **WHEN** the ride is `Offloading` and the last occupied seat becomes empty
- **THEN** the ride's state becomes `Idle`

### Requirement: Emergency stop from any active state

The ride SHALL accept an Emergency Stop request from any of the active states `Loading`, `Safe`, `Started`, and `Stopping`. Entering `Emergency Stop` SHALL immediately cut engine power and apply the brakes. Once the ride has come to a complete rest and its arms are parked it SHALL release the safety constraints and move to `Offloading`; the arms SHALL keep following the cam while the ride decelerates and SHALL be lowered only after rest.

#### Scenario: Emergency stop while running

- **WHEN** the ride is `Started` and an Emergency Stop is requested
- **THEN** the ride's state becomes `Emergency Stop`
- **AND** engine power is cut and the brakes are applied

#### Scenario: Emergency stop releases constraints on coming to rest

- **WHEN** the ride is in `Emergency Stop` and reaches a complete rest with its arms parked
- **THEN** the safety constraints are released
- **AND** the ride's state becomes `Offloading`

#### Scenario: Emergency stop with raised arms keeps the constraints locked

- **WHEN** the ride is in `Emergency Stop`, has reached a complete rest, and the arms are not yet parked
- **THEN** the safety constraints remain locked
- **AND** the ride's state remains `Emergency Stop`

#### Scenario: Emergency stop rejected when not in an active state

- **WHEN** the ride is `Idle` and an Emergency Stop is requested
- **THEN** the request is rejected with a domain validation error
- **AND** the ride's state remains `Idle`

### Requirement: Safety constraints and motion are confined by state

Safety constraints SHALL be locked on entering `Started` and SHALL only be released while the ride is at rest with its arms parked (on entering `Offloading`, whether reached from `Stopping` or `Emergency Stop`). The ride SHALL only be in motion while `Started`, `Stopping`, or `Emergency Stop`; in `Idle`, `Loading`, `Safe`, and `Offloading` the ride SHALL be at rest and its arms SHALL be parked. Arm motion (raising, following the cam, lowering) counts as ride motion.

#### Scenario: Constraints lock when the ride starts

- **WHEN** the ride transitions from `Safe` to `Started`
- **THEN** the safety constraints are locked
- **AND** occupied restraints can no longer be opened

#### Scenario: Constraints are never released while moving

- **WHEN** the ride is `Started`, `Stopping`, or `Emergency Stop` and still rotating
- **THEN** the safety constraints remain locked

#### Scenario: Arms are parked whenever the ride is loaded or unloaded

- **WHEN** the ride is `Idle`, `Loading`, `Safe` or `Offloading`
- **THEN** all four arms are at the park elevation
