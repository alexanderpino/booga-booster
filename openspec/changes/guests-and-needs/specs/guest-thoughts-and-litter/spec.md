# Spec Delta

## Purpose

Defines how guests express their state as short thoughts, how a park-wide mood feed is built from them, and how litter and vomit appear, affect nearby guests and the park metrics, and are cleaned up.

## ADDED Requirements

### Requirement: Each guest keeps its four most recent thoughts

A guest SHALL hold up to four thoughts, newest first, each with a kind, an optional subject entity index and the tick it was recorded. Recording a fifth SHALL drop the oldest. A thought of the same kind and subject as the newest SHALL be suppressed for 60 s.

#### Scenario: Ring behaviour

- **WHEN** five different thoughts are recorded in order
- **THEN** the guest holds the last four, newest first

#### Scenario: Duplicate suppression

- **WHEN** "hungry" is recorded and again 30 s later
- **THEN** only one entry exists; recorded again after 61 s it is added

### Requirement: The thought catalogue is closed and named

Thought kinds SHALL come from a closed list with stable numeric codes: hungry, thirsty, needs toilet, tired, sick, very sick, too intense, not thrilling, too expensive, cannot afford, good value, not in the rain, queue too long, waited for ages, enjoyed the ride, felt ill after the ride, disgusted by litter, enjoying the park, lost, no toilet, leaving unhappy. A code SHALL never be reused for a different meaning once released.

#### Scenario: Stable codes

- **WHEN** the catalogue is serialised in the inspection response
- **THEN** each kind has a stable code and an English label and a test pins the code table

### Requirement: Significant thoughts feed a bounded park mood feed

Thoughts of the kinds enjoyed the ride, felt ill after the ride, waited for ages, disgusted by litter, too expensive, good value, leaving unhappy and lost SHALL be copied into a global feed holding the last 200 entries with a rising cursor. Reads SHALL return entries after a cursor, oldest first, at most 50 per call. No more than 20 entries per simulated second SHALL be admitted; excess SHALL be counted as dropped, chosen deterministically (lowest guest number first within a second).

#### Scenario: Cursor paging

- **WHEN** 120 entries exist and a client reads after cursor 0 with limit 50
- **THEN** it receives entries with cursors 1 to 50 and a next cursor of 50

#### Scenario: Rate cap

- **WHEN** 100 admissible thoughts occur within one simulated second
- **THEN** 20 are admitted, 80 are counted as dropped, and the same 20 are admitted on every run

### Requirement: Litter comes from containers and vomit from nausea

Litter items SHALL be created only when a guest drops an empty container or vomits. Each item has a kind (litter or vomit), a node and a creation tick. The world SHALL hold at most 3 000 items; creation beyond that SHALL be skipped and counted.

#### Scenario: Cap

- **WHEN** 3 000 items exist and another is dropped
- **THEN** no item is created and the skipped counter increases by one

### Requirement: Nearby litter lowers guest mood

At each needs update a guest SHALL count litter items within 8 m (straight line, from a spatial hash rebuilt every 30 ticks) and subtract `6 * min(count, 5)` from its happiness target; with three or more items it records "disgusted by litter" at most once per 120 s.

#### Scenario: Five items

- **WHEN** seven items are within 8 m
- **THEN** the target falls by 30 for that update

#### Scenario: Clean surroundings

- **WHEN** no item is within 8 m
- **THEN** the litter term is 0

### Requirement: Old litter is counted for the park

Items older than 180 s SHALL be counted as old litter in the park metrics, recomputed every 12 s through a fixed-order reduction.

#### Scenario: Aging

- **WHEN** an item is created at t = 0 and the metrics are computed at 179 s and 181 s
- **THEN** it is not counted at 179 s and is counted at 181 s

### Requirement: Cleaning is a throughput supplied by staff levels

A singleton input SHALL hold the number of handymen (default 0). Each simulated minute, `handymen * 6` items SHALL be removed, oldest first, ties by lowest entity index. With zero handymen nothing is removed.

#### Scenario: Two handymen

- **WHEN** 50 items exist and two handymen are on duty for one minute
- **THEN** the 12 oldest items are removed

#### Scenario: No staff

- **WHEN** the handyman count is 0
- **THEN** no item is ever removed by cleaning
