## MODIFIED Requirements

### Requirement: Central ride visualization

The dashboard SHALL render a visual representation of the ride showing the central mill, its four main arms that rise and fall cyclically while the ride runs, and its 16 gondolas arranged around it on hubs that stay vertical, reflecting live rotation, live arm elevation and per-gondola occupancy.

#### Scenario: Ride rotating

- **WHEN** the central mill has a non-zero rotation speed
- **THEN** the visualization animates the gondolas' rotation in the mill's current direction at a rate reflecting the mill speed

#### Scenario: Ride stopped

- **WHEN** the central mill rotation speed is zero
- **THEN** the visualization is static

#### Scenario: Occupancy shown per gondola

- **WHEN** a gondola has one or more occupied seats
- **THEN** the visualization visually indicates that gondola as occupied, distinct from empty gondolas

#### Scenario: Arms rise and fall while the ride runs

- **WHEN** the ride is running with the arms engaged
- **THEN** the visualization shows opposite arms rising together while the adjacent arms fall, and each hub stays upright

#### Scenario: Ride stopped with raised arms

- **WHEN** the central mill has stopped while an arm is still raised
- **THEN** the visualization shows that arm raised and static until the arms are parked
