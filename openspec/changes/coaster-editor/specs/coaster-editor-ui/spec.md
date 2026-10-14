# Spec Delta

## Purpose

Defines the coaster editor screen: how a player starts a design, picks and places pieces, reads validation and test results, and replays a lap, entirely with a keyboard if necessary and with an equivalent text view of what the 3D canvas shows. It is built with PrimeNG, the mandated library for new UI.

## ADDED Requirements

### Requirement: The editor is a lazy route with four regions (Tier 1)

The editor SHALL be a lazy-loaded route `/coaster/editor` with a toolbar (undo, redo, test run, ride-along, export, import, build), a piece palette, the 3D view, and a side panel with tabs Pieces, Checks and Test run. It SHALL use PrimeNG components for toolbar, buttons, list selection, tabs, tables, messages and progress; the exact components SHALL be confirmed with the `primeng` MCP server before use. It SHALL be zoneless, signal-based and `OnPush`.

#### Scenario: Route loads lazily

- **WHEN** the application starts without visiting the editor
- **THEN** the editor code is not part of the initial bundle

### Requirement: Starting a design (Tier 1)

On first open the editor SHALL offer: start the starter coaster, start an empty design with anchor tile x and z, height and heading entered in a form, or import a file. Each choice SHALL end with a design in the editor and focus on the palette.

#### Scenario: Starter

- **WHEN** the player chooses the starter coaster
- **THEN** the editor shows 51 pieces, no errors and the palette focused

#### Scenario: Empty design form

- **WHEN** the anchor form is submitted with x 12, z 12, height 0 and heading East
- **THEN** a design with a single station at that anchor is created

### Requirement: The palette is a keyboard-operable list of what can be placed (Tier 1)

The palette SHALL list only the pieces the server offers, grouped, with a visible name, a small icon and an icon plus text for invalid entries stating the first error. It SHALL be one tab stop with arrow-key navigation (roving focus), Home and End, Enter or Space to append the focused piece, and the digits 1 to 9 to focus the first nine entries. Focusing an entry SHALL show its ghost.

#### Scenario: Keyboard placement

- **WHEN** the palette has focus, the Down arrow is pressed twice and Enter is pressed
- **THEN** the third entry is appended and focus stays in the palette on the new first entry

#### Scenario: Invalid entry

- **WHEN** an entry would cause a collision
- **THEN** it shows an error icon and the text "Collides with piece 12", and Enter on it announces that message and does not append

### Requirement: Shortcuts do not trap or surprise (Tier 1)

The editor SHALL support Ctrl+Z (undo), Ctrl+Y and Ctrl+Shift+Z (redo), Backspace (remove last piece), T (test run), V (ride-along), question mark (shortcut help) and Escape (cancel ghost, leave ride-along, close dialogs). Single-key shortcuts SHALL be ignored while focus is in a text input, select or dialog, and SHALL be listed in a help dialog reachable from a toolbar button. No shortcut SHALL replace a default browser shortcut that has no alternative.

#### Scenario: Typing is safe

- **WHEN** the name field has focus and the player types "tv"
- **THEN** no test run and no ride-along starts

#### Scenario: Undo

- **WHEN** Ctrl+Z is pressed after an append
- **THEN** the piece is removed and a message "Undid: added curve left radius 4" is announced

### Requirement: The track has a text equivalent (Tier 1)

The Pieces tab SHALL show the design as an ordered list or table: number, piece name, position in tiles, height in metres, and the finding badges for that piece. Selecting a row SHALL highlight the piece in the 3D view and selecting a piece in the view SHALL select its row. Every information that the 3D view conveys by position or colour, including markers, SHALL be available in text in this tab or the Checks tab.

#### Scenario: Row selection

- **WHEN** row 14 is selected
- **THEN** piece 14 is highlighted in the 3D view and the row has `aria-selected` true

### Requirement: Checks are listed with locations and suggestions (Tier 1)

The Checks tab SHALL list the findings in report order with severity icon and text, the piece number, the message and the suggestion. Selecting a finding SHALL move the selection to its piece. A design with errors SHALL disable test run and build with an explanation reachable by keyboard, not only by tooltip.

#### Scenario: Open circuit

- **WHEN** the circuit is open
- **THEN** the Checks tab shows the gap report in text and the Test run button is disabled with a described-by message

### Requirement: The test run panel shows result, ratings and a G chart (Tier 1)

After a test run the Test run tab SHALL show the outcome and verdict as text with an icon, lap time, maximum speed, the ratings as numbers with two decimals, the diagnosis list, and a chart of speed and of vertical, lateral and longitudinal G over time with the warn and hard limits drawn as lines and failure ranges shaded. The chart SHALL have a text summary and a data table alternative, and its cursor SHALL follow the replay position and be movable with the keyboard.

#### Scenario: Completed run

- **WHEN** the starter coaster is tested
- **THEN** the panel shows "Completed" with a pass icon, the lap time, the speed and three ratings

#### Scenario: Stall

- **WHEN** `starter-tall-hill` is tested
- **THEN** the panel shows the stall, the energy deficit in height steps and a marker row that selects the hill piece

### Requirement: Announcements use live regions (Tier 1)

Results of actions SHALL be announced in a polite live region: piece added or removed with the new error count, undo and redo, test started and finished with the outcome, conflict and reload. Errors that block the user SHALL use an assertive region or a focused alert. Announcements SHALL NOT repeat unchanged text.

#### Scenario: Append announced

- **WHEN** a piece is appended
- **THEN** the live region says "Added curve left radius 4 as piece 18. 0 errors, 1 warning."

### Requirement: The editor meets WCAG 2.2 AA in its main states (Tier 1)

The editor SHALL pass the axe-core accessibility specs with no violations in these states: empty start screen, design with errors, design with warnings, test run completed, test run failed, ride-along active, help dialog open. Interactive targets SHALL be at least 24 by 24 CSS pixels, focus SHALL be visible, text contrast SHALL be at least 4.5:1, and nothing SHALL flash more than three times per second.

#### Scenario: Axe

- **WHEN** the axe specs run on the seven states
- **THEN** each reports zero violations

### Requirement: Conflicts and failures are recoverable (Tier 1)

When the server answers 409 for a stale revision the editor SHALL reload the design, keep the palette selection when it is still offered and announce "The design changed; reloaded." When a request fails the editor SHALL keep the last good design on screen, show a message with a retry button and keep the keyboard focus where it was.

#### Scenario: Stale revision

- **WHEN** an append is answered with 409
- **THEN** the design is reloaded, the focus stays in the palette and the announcement is made

### Requirement: Cost and building (Tier 2)

The toolbar SHALL show the cost estimate when the server provides one, with the breakdown in the Pieces tab. Build SHALL open a confirmation dialog that states the cost and the test verdict, and its focus SHALL start on the cancel button.

#### Scenario: Confirm

- **WHEN** Build is pressed on a Tested design
- **THEN** a dialog names the cost, the verdict and the ratings, with focus on Cancel
