## MODIFIED Requirements

### Requirement: Gondola telemetry panel

The telemetry rail SHALL include a gondola panel listing all 16 gondolas; for each gondola it SHALL show which seats are occupied and the gondola's forward (fore/aft), lateral and vertical g-forces. The vertical g-force SHALL be the true vertical specific force (1.0 g at rest), not the forward component that earlier versions of the panel labelled as vertical.

#### Scenario: Occupied gondola under load

- **WHEN** a gondola has occupied seats and is experiencing a forward g-force of +2.0 g, a lateral g-force of -0.5 g and a vertical g-force of 1.4 g
- **THEN** that gondola's row shows its occupied seats, its forward g-force with sign (push-back vs push-forward), its lateral g-force with sign, and its vertical g-force

#### Scenario: Gondola at rest

- **WHEN** the ride is at rest
- **THEN** every gondola row shows a vertical g-force of 1.0 g and forward and lateral g-forces of 0.0 g

#### Scenario: All gondolas listed

- **WHEN** the gondola panel is rendered
- **THEN** exactly 16 gondolas are shown, each individually addressable/identifiable

#### Scenario: Old frame without vertical G

- **WHEN** a frame carries no vertical g-force
- **THEN** the row shows a vertical g-force of 1.0 g
