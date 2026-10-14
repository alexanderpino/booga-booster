# Spec Delta

## Purpose

Defines how the economy state is saved and loaded: a versioned, checksummed snapshot written atomically, validation of everything on load, migration of older versions, and pairing with the world save so a restored park and its economy belong together.

## ADDED Requirements

### Requirement: A snapshot captures the whole economy state

A snapshot SHALL contain a schema version, the park id, the simulation tick, the calendar position, scenario and profile ids, balance, opening balance, loan, ledger lines, the open week's running totals, price plan and history, staff roster, research state, campaigns, awards and history, rating and rating history ring, outbox sequence counters and a SHA-256 checksum of the payload. All amounts SHALL be integer cents.

#### Scenario: Round trip

- **WHEN** a snapshot of a state is saved and loaded into a fresh economy
- **THEN** the economy state hash of the loaded state equals the original

#### Scenario: Integer amounts only

- **WHEN** the saved document is inspected
- **THEN** every amount is a JSON integer

### Requirement: Saving is atomic and rotated

The store SHALL write to a temporary file in the target directory and replace the target only after the write is flushed, keeping the previous three versions as backups. A failure at any point SHALL leave the previous valid file readable. Saving SHALL work from an immutable projection and SHALL NOT run on the simulation thread.

#### Scenario: Crash before replace

- **WHEN** the process is stopped after the temporary file is written but before it replaces the target
- **THEN** loading reads the previous valid file

#### Scenario: Backup rotation

- **WHEN** five saves are made
- **THEN** the current file and the three most recent backups exist and older ones are gone

#### Scenario: Off the loop

- **WHEN** a save is requested during a run
- **THEN** no tick overruns because of it (tick time within 5 % of baseline in a manual-driver test)

### Requirement: Loading validates before it applies

Loading SHALL reject a snapshot, leaving the current state untouched, when: the schema version is newer than supported; the checksum does not match; the balance differs from opening plus all lines plus the open week net; the loan exceeds the profile maximum; the rating is outside [0, 999]; research or award ids are unknown; or any amount is outside the money limits. The error SHALL list every failed check.

#### Scenario: Tampered amount

- **WHEN** one ledger amount is edited in the file
- **THEN** the checksum check fails and the error says so

#### Scenario: Newer version

- **WHEN** the file's schema version is higher than the code supports
- **THEN** loading is refused with the two versions in the message

#### Scenario: Several failures

- **WHEN** the checksum is wrong and the rating is 1 200
- **THEN** both failures are listed

### Requirement: Older versions are migrated step by step

For each supported older schema version a migration SHALL convert the document to the next version; loading SHALL apply them in order and then validate. A committed fixture per supported version SHALL be loaded by a test.

#### Scenario: Migration chain

- **WHEN** a version 1 fixture is loaded by code at version 2
- **THEN** the loaded state has all version 2 fields with documented defaults and validates

### Requirement: The economy save is paired with the world save

A save SHALL record a manifest with the simulation tick and the world hash at the time of the snapshot. Loading the economy SHALL require that the world has been restored to the same tick and hash; otherwise loading SHALL be refused with both values, unless the operator explicitly requests an unpaired load, which SHALL be logged and reported in the response.

#### Scenario: Paired load

- **WHEN** the world and economy snapshots come from the same save
- **THEN** both load and the run continues, and the next 600 ticks equal an uninterrupted run (hash compared every 60 ticks)

#### Scenario: Mismatch

- **WHEN** the economy snapshot is from tick 50 000 and the world is at tick 49 000
- **THEN** loading is refused with both ticks in the message

### Requirement: Autosave and manual save

The economy SHALL autosave at every month close when a storage directory is configured, and SHALL save on request through the API. Autosave failures SHALL be logged and counted without affecting the simulation.

#### Scenario: Monthly autosave

- **WHEN** month 3 closes with storage configured
- **THEN** a snapshot with the calendar at the end of month 3 exists

#### Scenario: Failing disk

- **WHEN** the directory is read-only
- **THEN** autosave fails, a metric increments and the simulation continues
