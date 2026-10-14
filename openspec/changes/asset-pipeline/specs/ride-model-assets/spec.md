## ADDED Requirements

### Requirement: Four Octopus parts are delivered as glTF assets

The repository SHALL contain, as manifest entries with exported `.glb` files, source `.blend` files and provenance, the assets `octopus-arm` (class `ride-arm`, 6 m tapered tube or truss arm along `+X` from the hinge pin), `octopus-hinge` (class `ride-hinge`, hinge bracket and pin at the tower top), `octopus-cylinder` (class `ride-cylinder`, hydraulic cylinder body plus extending rod) and `octopus-gondola` (class `gondola`, two-seat pod hanging below its attachment point). Each SHALL satisfy the asset-conventions requirements for naming, axes, units, texture and triangle budgets. The required nodes SHALL be: arm: `root`, `joint_arm_elevation` (revolute, `z`), `mesh_arm`, `socket_hub_mount` (at the arm tip) and `socket_cylinder_anchor`; hinge: `root`, `mesh_hinge_bracket`, `mesh_hinge_pin`, `socket_arm_mount`; cylinder: `root`, `joint_cylinder_base` (revolute, `z`), `joint_cylinder_rod` (prismatic, `y`), `mesh_cylinder_body`, `mesh_cylinder_rod`, `socket_rod_end`; gondola: `root`, `joint_gondola_spin` (revolute, `y`), `mesh_gondola`, `socket_seat_0`, `socket_seat_1`.

#### Scenario: Assets validate

- **WHEN** `npm run assets:validate` runs after the slice
- **THEN** it exits 0 and lists four assets, each within its triangle and byte budget

#### Scenario: Provenance recorded

- **WHEN** the manifest is read
- **THEN** each of the four entries has a provenance record with `license`, `tool` and `aiAssisted`

### Requirement: The 3D view builds the Octopus from the assets

`RideVisualization` SHALL obtain the arm, hinge, cylinder and gondola through `AssetLoaderService.instantiate`, mount the arm model so its `joint_arm_elevation` is the node that the elevation angle from `octopus-motion` rotates, mount the hub pivot on the arm's `socket_hub_mount`, and mount each gondola on its hub-arm tip so `joint_gondola_spin` takes the free-spin angle that the procedural gondola pivot takes today. With no assets available (every asset in `fallback`) the view SHALL render the same procedural geometry as before this change and remain fully functional.

#### Scenario: Four arms, sixteen gondolas

- **WHEN** all assets are ready
- **THEN** the scene contains 4 arm instances and 16 gondola instances, and the number of distinct geometry objects for arms is 1

#### Scenario: Elevation drives the joint

- **WHEN** arm 0 has elevation +12 degrees from the motion function
- **THEN** its `joint_arm_elevation` rotation about `z` is +12 degrees within 1e-6 rad

#### Scenario: Offline fallback

- **WHEN** the asset requests all fail
- **THEN** the view renders procedural arms, hubs and gondolas and the fallback message is shown

### Requirement: The hub keeps a vertical axis

Whatever the arm elevation, the hub pivot mounted on `socket_hub_mount` SHALL keep its local `+Y` axis aligned with world `+Y`, by counter-rotating the hub pivot about `z` by the negative of the arm elevation (the compensating linkage of `octopus-motion`), and the hub's position SHALL stay at the arm tip. The assets SHALL NOT bake any compensation into the models.

#### Scenario: Vertical axis at maximum elevation

- **WHEN** arm 2 is at its maximum elevation angle
- **THEN** the hub pivot's world-space up vector equals `(0, 1, 0)` within 1e-6 on each component

#### Scenario: Hub follows the tip

- **WHEN** the arm is elevated by an angle `a` with arm length 6 m
- **THEN** the hub pivot's world position equals the hinge position plus `(6 cos a, 6 sin a, 0)` rotated about Y by the arm's azimuth, within 1e-4 m

### Requirement: The hydraulic cylinder follows the arm

The cylinder instance SHALL be placed with `joint_cylinder_base` at a fixed anchor on the tower, rotated so its axis points at the arm's `socket_cylinder_anchor` (world space), with `joint_cylinder_rod` extended by the measured distance minus the body's retracted length, clamped to the joint's declared range. This is a visual linkage only and SHALL NOT feed the simulation.

#### Scenario: Cylinder tracks the anchor

- **WHEN** the arm is at 0 degrees and then at +20 degrees
- **THEN** in both poses the cylinder's `socket_rod_end` world position is within 0.01 m of the arm's `socket_cylinder_anchor`

#### Scenario: Rod range clamp

- **WHEN** the required extension exceeds the rod joint's `max`
- **THEN** the rod is set to `max` and no error is thrown

### Requirement: Obsolete model file is retired

`public/3dmodels/boogabooster.fbx` SHALL be removed from the repository if and only if a repository-wide search finds no reference to it; the grep result SHALL be recorded in the commit message.

#### Scenario: Unreferenced FBX

- **WHEN** a search for `boogabooster.fbx` and `3dmodels` finds no source reference
- **THEN** the file is removed in its own commit
