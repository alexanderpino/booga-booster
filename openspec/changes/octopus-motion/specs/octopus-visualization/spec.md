# Spec Delta

## Purpose
Defines how the dashboard shows the elevating Octopus arms: the 3D rendering of tapered-tube arms with a hinge, a hydraulic cylinder and a linkage that keeps the hub vertical, its performance budget and the seam for later glTF assets, and the accessible arm-elevation panel in the telemetry rail.

## ADDED Requirements

### Requirement: Arms are rendered as elevating tapered tubes

The 3D view SHALL render each of the four main arms as a tapered round tube (root radius 0.24 m, tip radius 0.14 m, length 6.0 m from the hinge to the hub centre) that pivots about a horizontal hinge axis perpendicular to the arm, at the top of the central tower (6.0 m above the ground). The arm elevation SHALL be taken from the telemetry arm record with the same index. A parked ride SHALL show the four arms horizontal.

#### Scenario: Arm follows the reported elevation

- **WHEN** a frame reports arm 0 at an elevation of 30.0 deg
- **THEN** the rendered arm 0 rises 30.0 deg (within 0.1 deg) above the horizontal
- **AND** its tip is 3.0 m higher than the hinge (within 0.01 m)

#### Scenario: Parked ride is flat

- **WHEN** the telemetry reports the engagement `Parked`
- **THEN** all four rendered arms are horizontal

### Requirement: The hub stays vertical on the moving arm tip

Each hub, including its four hub arms and gondolas, SHALL be positioned at the arm tip (horizontal reach `L cos(theta)`, height `L sin(theta)` above the hinge) but SHALL NOT tilt with the arm: the hub axis SHALL remain vertical for every elevation. A parallelogram linkage (a second tube of the same length, held above the arm by a fixed vertical offset of 0.55 m and ending at a vertical hub bracket) SHALL be drawn so the vertical hub follows plausibly from the mechanism.

#### Scenario: Hub up-axis at maximum elevation

- **WHEN** arm 0 is at 30 deg
- **THEN** the world-space up axis of hub 0 is vertical to within 1e-6
- **AND** the world-space positions of its four gondola pods differ in height by less than 1e-6 m

#### Scenario: Linkage stays a parallelogram

- **WHEN** an arm moves through its full range of 0 deg to 30 deg
- **THEN** the linkage tube remains parallel to the arm and of the same length to within 1e-6 m
- **AND** the hub bracket remains vertical

### Requirement: Hinge pin and hydraulic cylinder are drawn and animated

The arm root SHALL show a hinge pin (radius 0.09 m, axis tangent to the mill circle) with two clevis plates; the pin of opposite arms MAY be shared because opposite arms move in unison. A hydraulic cylinder SHALL connect a fixed lug on the tower (0.8 m out and 1.4 m below the hinge, in the arm's vertical plane) to a lug on the arm 2.0 m from the hinge. The cylinder SHALL be drawn as a barrel and a rod whose visible extension follows the pin-to-pin distance, which varies from 1.844 m (arm at 0 deg) to 2.575 m (arm at 30 deg) and SHALL stay within a stroke of 1.80 m to 2.60 m; the cylinder SHALL always point from the tower lug to the arm lug.

#### Scenario: Cylinder extends as the arm rises

- **WHEN** an arm moves from 0 deg to 30 deg
- **THEN** the cylinder pin-to-pin distance grows from 1.844 m to 2.575 m (within 0.005 m)
- **AND** the visible rod extension grows by the same amount

#### Scenario: Cylinder endpoints stay attached

- **WHEN** an arm is at any elevation between 0 deg and 30 deg
- **THEN** the barrel base is at the tower lug and the rod end is at the arm lug, each within 1e-6 m

### Requirement: Arm azimuth is locked to the server mill angle

The renderer SHALL keep the rendered arm azimuth locked to the reported mill angle so that the elevation shown for an arm matches the azimuth it is drawn at. Between frames it SHALL advance the azimuth and the elevation from the reported speed and elevation rate (extrapolation of at most one frame interval), and on each new frame it SHALL pull the rendered azimuth towards the reported angle with a first-order correction (time constant 0.25 s) without ever turning the arms backwards. When the frame lacks the mill angle or an arm record, it SHALL fall back to the locally integrated azimuth and to parked arms. Non-finite values SHALL never reach the scene graph.

#### Scenario: Azimuth stays within tolerance

- **WHEN** the mill turns steadily at 15 rpm and frames arrive at 30 Hz with a 5 ms jitter
- **THEN** after 2 s the rendered arm azimuth differs from the reported mill angle by less than 5 deg at every rendered frame

#### Scenario: Correction never reverses the motion

- **WHEN** a frame reports a mill angle 20 deg behind the rendered azimuth while the mill turns forward
- **THEN** the rendered azimuth keeps turning forward at a reduced rate until the reported angle catches up

#### Scenario: Corrupt frame is harmless

- **WHEN** a frame contains a non-finite elevation
- **THEN** the affected arm keeps its previous finite elevation and no scene-graph transform becomes non-finite

### Requirement: Reduced motion

When the user prefers reduced motion, the arms SHALL NOT animate: they SHALL be drawn at the park elevation with the hub vertical, the cylinder retracted to its park length, and the elevation SHALL still be available in the elevation panel.

#### Scenario: Reduced motion with engaged arms

- **WHEN** reduced motion is preferred and the telemetry reports arm 0 at 25 deg
- **THEN** the rendered arm 0 stays horizontal

### Requirement: Procedural placeholders behind a stable node contract

The arm geometry SHALL be built procedurally now and SHALL be addressable through named nodes (`arm-root`, `arm-tube`, `hinge-pin`, `cylinder-barrel`, `cylinder-rod`, `linkage-tube`, `hub-bracket`) so that a later glTF asset (see the asset-pipeline change) can replace the placeholders without changing the animation code. Meshes SHALL cast shadows as the existing arm boxes do.

#### Scenario: Animation drives named nodes

- **WHEN** the arm model is built
- **THEN** every node named in this requirement exists exactly once per arm
- **AND** the animation reads and writes only those nodes

### Requirement: Rendering performance budget

With the arm model added, the whole scene SHALL stay within 130 meshes and 80,000 triangles, SHALL create no geometry and no material per frame, and SHALL not allocate objects in the per-frame update. The per-frame update of the four arms SHALL take at most 0.5 ms on the reference development laptop. The shadow-map size and the WebGL renderer settings SHALL be unchanged. Geometries and materials SHALL be shared where identical and disposed on teardown.

#### Scenario: Scene budget

- **WHEN** the scene is built without a WebGL context
- **THEN** the traversal counts at most 130 meshes and at most 80,000 triangles

#### Scenario: Teardown releases the arm resources

- **WHEN** the visualization is destroyed
- **THEN** every geometry and material of the arm model has been disposed

### Requirement: Arm-elevation panel in the telemetry rail

The telemetry rail SHALL include an arm-elevation panel built with the application's component library (PrimeNG). For each arm it SHALL show the arm number, the elevation in degrees with one decimal and its unit, the hub height in metres and a bar showing the elevation within the 0 to 30 deg cam range; it SHALL show the engagement state as text (`Parked`, `Engaging`, `Engaged`, `Parking`) with an icon. Every numeric value SHALL carry its unit in its accessible name. State SHALL NOT be conveyed by colour alone. The panel SHALL render sensible values (all arms at 0.0 deg, `Parked`) when the telemetry has no arm data.

#### Scenario: Arms at different elevations

- **WHEN** arm 0 is at 30.0 deg and arm 1 is at 0.0 deg
- **THEN** the panel shows "Arm 0 30.0 degrees" and "Arm 1 0.0 degrees" with their heights in metres
- **AND** the bars reflect 100 % and 0 % of the range

#### Scenario: Engagement state is announced as text

- **WHEN** the engagement state changes from `Engaging` to `Engaged`
- **THEN** the panel shows the text `Engaged` with its icon
- **AND** a polite live region announces the new state once

#### Scenario: No arm data

- **WHEN** the latest frame has no arm records
- **THEN** the panel shows four arms at 0.0 degrees with state `Parked`

### Requirement: Arm-elevation panel is accessible and does not flood assistive technology

The panel SHALL pass automated axe-core checks at WCAG AA (including colour contrast in the application theme), SHALL expose each bar with an accessible name and its current value, SHALL contain no interactive element that needs keyboard focus, and SHALL update its displayed numbers at most 4 times per second. Only the engagement state, not the numbers, SHALL be in a live region.

#### Scenario: axe passes

- **WHEN** the panel is rendered with engaged arms
- **THEN** axe-core reports no violations

#### Scenario: Numbers are throttled

- **WHEN** 30 frames per second change the arm elevations for 1 s
- **THEN** the displayed elevation text changes at most 4 times in that second
- **AND** the live region is not updated by those changes

#### Scenario: Dashboard wiring

- **WHEN** the dashboard is rendered
- **THEN** the arm-elevation panel is in the right telemetry rail below the rotation-speed panel
