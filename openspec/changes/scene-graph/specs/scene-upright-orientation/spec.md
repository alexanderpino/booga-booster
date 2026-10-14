## ADDED Requirements

### Requirement: A node can keep only the yaw of its parent's orientation

Each node SHALL have an orientation mode, `Inherit` (default) or `UprightYaw`. In `UprightYaw` mode the node's world rotation SHALL be `normalize(twistY(P.rotation) * L.rotation)`, where `twistY(q)` is the rotation about +Y obtained by the swing-twist decomposition of the parent's world rotation `q = (x, y, z, w)`: `normalize((0, y, 0, w))`. When `y` and `w` are both below 1e-12 in magnitude (the twist is undefined) `twistY` SHALL be the identity. The node's world position and scale SHALL be computed exactly as in `Inherit` mode, so the node still sits at the parent's moved point. The mode SHALL be evaluated closed-form in the update pass: no iteration and no constraint solver.

#### Scenario: Tilting arm does not tilt the hub

- **WHEN** the mill is rotated +90 degrees about Y, the arm below it is rotated +30 degrees about Z, and a hub below the arm with local position (6, 0, 0) is in `UprightYaw` mode with identity local rotation
- **THEN** the hub's world up axis (its rotation applied to (0, 1, 0)) is (0, 1, 0) within 1e-12
- **AND** the hub's world position is (0, 3, -5.196152422706632) within 1e-9

#### Scenario: Own spin adds to the yaw

- **WHEN** the same hub has local rotation +45 degrees about Y
- **THEN** its world rotation equals +135 degrees about Y within 1e-12

#### Scenario: Radial reach follows the cosine

- **WHEN** the arm pitch is varied over 0, 15, 30, 45, 60 degrees with arm length L = 6
- **THEN** the hub's horizontal distance from the mill axis is 6 cos(pitch) within 1e-9 and its height is 6 sin(pitch) within 1e-9

#### Scenario: Children of an upright node inherit normally

- **WHEN** a gondola below the upright hub arm chain is in `Inherit` mode
- **THEN** its world up axis is (0, 1, 0) within 1e-12 as long as no node between the hub and the gondola tilts

#### Scenario: Undefined twist

- **WHEN** the parent's world rotation is 180 degrees about X (quaternion (1, 0, 0, 0))
- **THEN** `twistY` is the identity and the node's world rotation equals its local rotation

#### Scenario: Root in upright mode

- **WHEN** a root node is in `UprightYaw` mode
- **THEN** its world transform equals its local transform

### Requirement: Mode changes behave like transform changes

Changing a node's orientation mode SHALL mark it dirty, and descendants SHALL be recomputed in the same update. The mode SHALL be part of the world state hash through its component.

#### Scenario: Toggle

- **WHEN** the hub mode changes from `Inherit` to `UprightYaw` and the graph is updated
- **THEN** the hub and its descendants are recomputed and the hub's world up axis becomes (0, 1, 0)
