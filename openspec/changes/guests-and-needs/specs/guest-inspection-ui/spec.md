# Spec Delta

## Purpose

Defines the guest inspection experience in the browser: selecting a guest, reading its needs and thoughts, a park-wide guest summary and mood feed, and the read endpoints behind them. It is built with PrimeNG and must meet WCAG AA.

## ADDED Requirements

### Requirement: Inspection data is served from a published projection

The simulation SHALL publish an immutable read projection of guest detail every fourth tick, swapped atomically, and all guest read endpoints SHALL read from the latest projection without waiting on the tick loop. The projection SHALL not be older than 100 ms in a running simulation.

#### Scenario: Reads do not stall the tick

- **WHEN** 100 clients poll the inspection endpoint at 1 Hz
- **THEN** no tick overruns because of them (tick time with polling is within 5 % of tick time without)

#### Scenario: Projection age

- **WHEN** an endpoint is read at a random time in a running simulation
- **THEN** the response carries the tick of its projection and that tick is at most 12 ticks old

### Requirement: A guest can be inspected by id

`GET` on the guest route with entity index and generation SHALL return the guest's detail (fields listed in the first scenario). An unknown or stale id SHALL return not found.

#### Scenario: Known guest

- **WHEN** an existing guest is requested
- **THEN** the response contains display name, guest number, seconds in the park, happiness, hunger, thirst, bladder, fatigue and nausea as percentages, cash in cents, intent kind with target ride id when any, party size, rides ridden count, held item and map flags, intensity window, nausea tolerance label, queue tolerance in seconds, and up to four thoughts with code, label, subject and seconds ago
- **AND** happiness equals the simulation value divided by 100

#### Scenario: Stale generation

- **WHEN** the guest left and its index was reused
- **THEN** the request with the old generation returns not found

#### Scenario: Camel-case JSON

- **WHEN** the response is serialised
- **THEN** it uses camel-case property names and no field is missing for a valid guest

### Requirement: A park summary and mood feed are available

The summary SHALL report guests in park, arrived, left, turned away, average happiness, share of guests happier than 50 percent, counts by intent kind, longest queue per ride, and old litter count. The feed SHALL return entries after a cursor as defined by the thoughts capability.

#### Scenario: Summary values

- **WHEN** 100 guests are in the park and 62 have happiness above 5 000
- **THEN** the happy share is 62 percent

#### Scenario: Feed paging

- **WHEN** the feed is read with cursor 40 and limit 10
- **THEN** it returns entries 41 to 50 oldest first

### Requirement: An accessible list lets users select guests without the canvas

A nearby-guests endpoint SHALL return up to 20 guests nearest to a given point (ties by lower entity index) with id, name, mood tier and intent. The UI SHALL offer a keyboard-operable list built from it, so every selection possible on the canvas is possible without a pointer.

#### Scenario: Nearest first

- **WHEN** guests are at 3, 5 and 9 m from the point
- **THEN** the list is ordered 3, 5, 9

#### Scenario: Keyboard selection

- **WHEN** the user tabs to the list and presses Enter on an entry
- **THEN** that guest becomes selected and the inspection panel shows it

### Requirement: The inspection panel uses PrimeNG and refreshes politely

The panel SHALL be built with PrimeNG components (consulting the `primeng` MCP server for component choice), standalone, signal-driven, OnPush. While a guest is selected it SHALL refresh once per second. Needs SHALL be shown as labelled progress bars with the percentage in text. Refresh SHALL not move keyboard focus and SHALL announce nothing except when the guest leaves (a polite status "Guest left the park").

#### Scenario: Values render with text

- **WHEN** hunger is 72 percent
- **THEN** the hunger bar has an accessible name "Hunger", exposes the value 72 and shows "72 %" as text

#### Scenario: Guest leaves while selected

- **WHEN** the selected guest leaves the park
- **THEN** the panel shows "Guest left the park" in a polite live region and offers to clear the selection

#### Scenario: Focus is kept

- **WHEN** the panel refreshes while the user is focused on a control inside it
- **THEN** focus remains on that control

### Requirement: The guest UI passes accessibility checks

All new guest UI SHALL pass axe-core with no violations at the WCAG AA level the repository already requires in the Vitest a11y specs, SHALL be fully keyboard operable, SHALL meet 4.5:1 text contrast and 3:1 non-text contrast, and SHALL respect reduced-motion preferences by disabling the walking bob.

#### Scenario: Axe

- **WHEN** the a11y spec renders the panel, list, summary, feed and meter
- **THEN** axe-core reports zero violations

#### Scenario: Reduced motion

- **WHEN** the user's preference is reduced motion
- **THEN** the walking bob is not applied
