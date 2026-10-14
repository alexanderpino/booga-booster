# Spec Delta

## Purpose

Defines the first-person ride-along camera: where the eye is, how it follows a train or a test-run replay, how the player controls it, and the comfort settings that protect players sensitive to motion.

## ADDED Requirements

### Requirement: The camera rides in the first car (Tier 1)

In ride-along mode the camera SHALL be placed at the rider frame at the arc length of the lead car, offset 1.15 m along the rider up vector, looking along the tangent, with a vertical field of view of 75 degrees and a near plane of 0.05 m. The camera orientation SHALL be the rider frame (tangent, up, lateral) so that bank and pitch are felt.

#### Scenario: On a flat straight

- **WHEN** the train is on a flat straight at arc length s
- **THEN** the camera is 1.15 m above the track centreline, looks along the track and its up vector is world up

#### Scenario: On a climb

- **WHEN** the train is on `slope:+2`
- **THEN** the camera pitch is 26.57 degrees upward within 0.01 degrees

### Requirement: The camera follows replay or live state (Tier 1)

The camera SHALL take its arc length from the test-run replay (trace time to arc length, linearly interpolated) or from the live dead-reckoned train state, with the same code path. Replay SHALL support play, pause, restart and speed 0.5, 1 and 2, and a slider for position that moves the train and the G-chart cursor together.

#### Scenario: Scrub

- **WHEN** the slider is moved to 20.0 s of a replay
- **THEN** the train, the camera and the chart cursor are all at the trace sample for 20.0 s

### Requirement: Look-around is bounded (Tier 2)

The player MAY look around with the pointer or the arrow keys by up to 60 degrees to each side and 40 degrees up and down relative to the rider frame; the view SHALL return to centre when the player presses the Home key or leaves the mode.

#### Scenario: Limit

- **WHEN** the player drags 90 degrees to the right
- **THEN** the view stops at 60 degrees to the right

### Requirement: Comfort settings reduce motion (Tier 1)

When the browser reports `prefers-reduced-motion: reduce` the camera SHALL start with roll scaled to 30 % and replay speed limited to 1, and SHALL show how to change this. A visible control SHALL toggle a stabilised horizon that scales roll to 0 % to 100 % in steps. The first use SHALL show a dismissible notice that ride-along views can cause motion sickness, with a button to exit. Escape SHALL leave the mode from any state.

#### Scenario: Reduced motion preference

- **WHEN** the preference is set
- **THEN** ride-along starts with roll at 30 % and the speed selector offers 0.5 and 1 only

#### Scenario: Escape

- **WHEN** Escape is pressed in ride-along mode
- **THEN** the editor camera returns to its previous position

### Requirement: The camera never shows the inside of the track mesh (Tier 1)

The eye position SHALL be at least 0.5 m from any rail or tie centre and at least 0.3 m from the nearest support column, and the camera SHALL not be rendered through the first car body, which SHALL be hidden in first-person mode.

#### Scenario: Car hidden

- **WHEN** ride-along is active
- **THEN** the first car body is not drawn from the camera
