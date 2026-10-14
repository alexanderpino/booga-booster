# Spec Delta

## Purpose

Defines the Angular catalog picker that lets a player choose what to build and the ride inspector that shows and controls a placed ride, both accessible and built with PrimeNG.

## ADDED Requirements

### Requirement: Catalog picker

The build panel SHALL show a catalog picker with one group per category (gentle rides, thrill rides, coasters, shops, facilities, path furniture, staff), a search box that filters by name and serve tag, and a list of entries. Each entry SHALL show name, footprint size, cost formatted as currency (`12000.00`), and for rides capacity and the default-level ratings as three numbers (`1.05`). A locked entry SHALL show a lock indicator and its reason in text and SHALL not be selectable. Selecting an entry SHALL activate the park's Place object tool with that definition.

#### Scenario: Group and entry

- **WHEN** the catalog is loaded
- **THEN** the thrill group lists `octopus`, `corsair-swing` and `whirlwind-twirl` with cost and capacity

#### Scenario: Search

- **WHEN** the player types `drink`
- **THEN** only entries whose name or serve tags match, such as `fizz-kiosk`, remain

#### Scenario: Locked entry

- **WHEN** an entry is locked with reason `needs research:thrill-1`
- **THEN** it shows the lock and the reason, and activating it does nothing

#### Scenario: Select

- **WHEN** the player selects `starlight-carousel`
- **THEN** the Place object tool is active with that definition and the ghost uses its footprint

### Requirement: Picker accessibility

The picker SHALL be operable by keyboard alone: one tab stop for the group list, arrow keys moving between entries, Home and End to the first and last, Enter or Space to select, `/` focusing the search box, and Escape clearing the search. Groups SHALL be exposed as labelled regions, selection as a pressed or selected state, and a result count SHALL be announced politely after filtering. The picker SHALL pass axe-core with zero violations in the states loaded, filtered, empty result, locked entry and loading. Colour SHALL never be the only carrier of locked or selected state.

#### Scenario: Keyboard selection

- **WHEN** the focus is on the list, the player presses Down twice and Enter
- **THEN** the third entry becomes selected

#### Scenario: Filter announcement

- **WHEN** the filter leaves 2 entries
- **THEN** a polite live region says that 2 entries are shown

#### Scenario: Empty result

- **WHEN** the filter matches nothing
- **THEN** a visible text says nothing matches and axe reports no violation

### Requirement: Placeholder rendering for catalog objects

For every placed object that has no loaded art, the park view SHALL draw the definition's placeholder (the tinted box with a rotation stripe) through the park renderer's placeholder mechanism, sized from the definition. Scripted rides SHALL animate their `joint_motion` node from the scene snapshots so spin and swing are visible.

#### Scenario: Box size

- **WHEN** a `starlight-carousel` is placed
- **THEN** a box 12 m by 12 m by 5 m in its tint is drawn at the footprint centre

#### Scenario: Spinning joint

- **WHEN** the carousel is Started
- **THEN** the box's joint node rotates about the vertical axis following the server angle

### Requirement: Ride inspector

Selecting a placed ride with the Inspect tool SHALL open an inspector showing: name, status (Closed, Open, Broken), lifecycle state, operating level with a control, ratings (excitement, intensity, nausea) with their source, reliability as a percentage, ticket price, value, throughput, queue length and queue time. It SHALL offer Open, Close, Emergency stop and Repair buttons enabled only when allowed, show the server's refusal reason as text, and update from polling `GET /rides/{id}` at 2 Hz while open. Values SHALL be announced politely only when status or lifecycle state changes.

#### Scenario: Buttons follow status

- **WHEN** the ride is Closed
- **THEN** Open is enabled and Close and Emergency stop are disabled

#### Scenario: Refusal text

- **WHEN** Open is refused with `QueueEntryNotConnected`
- **THEN** the inspector shows that the queue entry is not connected to a queue path

#### Scenario: Polling stops on close

- **WHEN** the inspector is closed
- **THEN** polling stops

### Requirement: Inspector accessibility and states

The inspector SHALL pass axe-core with zero violations in these states: Closed ride, Open ride running, Broken ride, refusal shown. Values SHALL use text labels, not colour alone; the level control SHALL be keyboard operable with an accessible name and value text.

#### Scenario: Axe states

- **WHEN** the a11y spec renders the four states
- **THEN** axe reports no violations

#### Scenario: Level control by keyboard

- **WHEN** the level slider has focus and Right arrow is pressed
- **THEN** the level increases by one and the new value is exposed as text
