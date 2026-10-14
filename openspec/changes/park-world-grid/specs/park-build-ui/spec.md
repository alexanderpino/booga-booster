# Spec Delta

## Purpose

Defines the Angular build-mode interface for the park: how a player chooses a tool, previews a placement, sees cost and reasons, places, removes and undoes, using mouse or keyboard alone and with assistive technology.

## ADDED Requirements

### Requirement: Build mode shell

The app SHALL provide a lazy-loaded `/park` route that shows the park view and the build panel. It SHALL load the grid by fetching the keyframe and then following the delta stream. State SHALL be held in signals; components SHALL use OnPush change detection and no zone.js. New UI SHALL use PrimeNG components.

#### Scenario: First load

- **WHEN** the player opens `/park`
- **THEN** the keyframe is fetched, the grid is shown, and the delta stream is connected

#### Scenario: Stream lost

- **WHEN** the stream disconnects
- **THEN** a visible, announced banner reports the connection loss, the client reconnects with back-off, and on reconnect resynchronises by delta catch-up or keyframe

### Requirement: Tool palette

The palette SHALL offer these tools: Inspect, Path, Queue path, Park entrance, Place object (with a definition list), Raise terrain, Lower terrain, Demolish, plus Undo and Redo buttons. Exactly one tool SHALL be active. Each tool SHALL have a visible text label, an icon, a tooltip and a keyboard shortcut (keys 1 to 8 select the tools in the listed order). Undo and Redo SHALL be disabled when the server last reported them unavailable.

#### Scenario: Select a tool by key

- **WHEN** the player presses `2` with the focus in the park view
- **THEN** the Path tool becomes active and its button reports a pressed state

#### Scenario: Undo disabled

- **WHEN** the server reports no undo history
- **THEN** the Undo button is disabled and still has an accessible name

### Requirement: Ghost preview and validity

With a placing tool active, the view SHALL show a ghost of the object at the cursor tile with its rotation. The client SHALL ask the server to validate the placement (dry run) after the cursor settles, at most one request in flight and at most one per 60 ms, and SHALL ignore any response that is not for the latest request. While no response has arrived the ghost SHALL show a neutral state. Cheap local checks (inside the map) SHALL apply immediately. The ghost SHALL show valid and invalid states by colour, by a distinct icon or pattern, and by text.

#### Scenario: Valid placement

- **WHEN** the dry run returns valid with cost 800
- **THEN** the ghost shows the valid state and the panel shows the cost `8.00`

#### Scenario: Stale response ignored

- **WHEN** the cursor moves twice quickly and the response for the first position arrives last
- **THEN** the panel shows the result for the second position

#### Scenario: Not by colour alone

- **WHEN** a placement is invalid
- **THEN** the panel lists the human-readable reasons as text and the ghost carries a non-colour marker

### Requirement: Reason messages

The client SHALL map every server reason code to a short plain-language message and SHALL show all reasons returned. An unknown code SHALL be shown as a generic message that includes the raw code.

#### Scenario: Known reason

- **WHEN** the reason `Occupied` is returned
- **THEN** the text states that something is already built there

#### Scenario: Unknown reason

- **WHEN** the reason `FutureReason` is returned
- **THEN** the text reads as a generic refusal and contains `FutureReason`

### Requirement: Placing paths by dragging

With a path tool active, pressing the pointer on a tile and dragging SHALL show a straight line of ghosts along the dominant axis and SHALL send one batch on release. Releasing outside the map SHALL cancel. With the keyboard, holding Shift while moving the cursor SHALL extend the same line and Enter SHALL send it.

#### Scenario: Drag a line

- **WHEN** the player drags from `(4, 4)` to `(9, 5)` with the Path tool
- **THEN** the ghost line covers `(4, 4)` to `(9, 4)` and one batch of six placements is sent on release

#### Scenario: Cancel

- **WHEN** the player presses Escape during a drag
- **THEN** nothing is sent and the ghost disappears

### Requirement: Keyboard operation

Every function SHALL be reachable by keyboard. The park view SHALL be focusable and expose a tile cursor moved one tile by an arrow key and eight tiles by Ctrl plus an arrow key. Enter or Space SHALL apply the active tool at the cursor; Delete SHALL demolish; `R` and Shift+`R` SHALL rotate; Escape SHALL cancel the current action then return to the Inspect tool; Ctrl+Z SHALL undo; Ctrl+Y and Ctrl+Shift+Z SHALL redo; `?` SHALL open a keyboard help dialog. Tab SHALL move focus out of the park view; there SHALL be no keyboard trap.

#### Scenario: Cursor and place

- **WHEN** the park view has focus, the Path tool is active, the player presses Right twice and Enter
- **THEN** the cursor moved two tiles East and a path placement was sent for that tile

#### Scenario: Escape chain

- **WHEN** the Path tool is active and no drag is running and Escape is pressed
- **THEN** the Inspect tool becomes active

#### Scenario: No trap

- **WHEN** the player presses Tab in the park view
- **THEN** focus moves to the next control after the view

### Requirement: Screen reader support

The park view SHALL have an accessible name and a description pointing at the keyboard help. A polite live region SHALL announce the cursor tile description whenever the cursor moves (coordinates, terrain level, what is on the tile), the outcome of every command (placed with cost, refused with reasons, undone, redone) and connection changes. Errors SHALL be announced assertively. Announcements SHALL be rate-limited to one per 250 ms with the latest winning.

#### Scenario: Cursor announcement

- **WHEN** the cursor moves onto a tile holding a Normal path at level 8
- **THEN** the live region text names the coordinates, level 8 and a Normal path

#### Scenario: Refusal announcement

- **WHEN** a placement is refused with `Occupied`
- **THEN** an alert announces the refusal and its reason

### Requirement: Accessibility conformance

The build panel and dialogs SHALL pass the repository's axe-core WCAG AA checks with zero violations in these states: idle, tool selected, valid ghost, invalid ghost with reasons, error banner, help dialog open. Controls SHALL have visible focus, text contrast SHALL meet AA, and any motion on the ghost SHALL be disabled when the user prefers reduced motion. Touch input is out of scope for this change.

#### Scenario: axe states

- **WHEN** the a11y spec renders each listed state
- **THEN** axe reports no violations

#### Scenario: Reduced motion

- **WHEN** the user prefers reduced motion
- **THEN** the ghost does not pulse or animate

### Requirement: Cost and undo feedback

After each command the panel SHALL show the amount charged or refunded and a short history of the last five commands. Undo and Redo SHALL update the grid view only through the delta stream, not by local prediction.

#### Scenario: Placement feedback

- **WHEN** a path placement succeeds with cost 800
- **THEN** the panel shows "Path placed, cost 8.00" and the entry appears in the history

#### Scenario: No local prediction

- **WHEN** the player presses Undo
- **THEN** the grid changes only after the corresponding delta arrives
