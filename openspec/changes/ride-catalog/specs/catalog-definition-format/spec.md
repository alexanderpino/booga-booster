# Spec Delta

## Purpose

Defines the data format in which every buildable thing is described, the rules a definition must satisfy, how definitions are loaded and versioned, and the stable error codes reported for invalid ones.

## ADDED Requirements

### Requirement: One JSON file per definition

Each definition SHALL be one UTF-8 JSON file named `<id>.json`. The file SHALL carry `schemaVersion` (integer, currently 1), `id`, `version` (integer at least 1), `kind`, `category`, `name` and `description`. The id SHALL match `^[a-z][a-z0-9-]{0,63}$` and SHALL equal the file name without extension. Unknown properties at any level SHALL be errors. A `$schema` property pointing at the published schema file SHALL be allowed for editor support.

#### Scenario: Valid minimal stall

- **WHEN** a file `frost-cart.json` has all required fields of a stall and id `frost-cart`
- **THEN** it loads without errors

#### Scenario: File name differs from id

- **WHEN** `frost-cart.json` declares id `frost-kart`
- **THEN** loading reports `FileNameMismatch` for that file

#### Scenario: Unknown property

- **WHEN** a definition contains a property `colour`
- **THEN** loading reports `FieldUnknown` naming `colour`

#### Scenario: Unsupported schema version

- **WHEN** a definition declares `schemaVersion` 2
- **THEN** loading reports `SchemaVersionUnsupported`

### Requirement: Kinds and categories

`kind` SHALL be one of `flat-ride`, `coaster`, `stall`, `facility`, `path-addition`, `staff-building`. The allowed `category` per kind SHALL be: flat-ride `gentle` or `thrill`; coaster `roller-coaster`; stall `shop`; facility `facility`; path-addition `path-furniture`; staff-building `staff`. A kind or category outside the list SHALL be reported as `KindInvalid` or `FieldUnknown` respectively, and a valid category on the wrong kind as `KindFieldMismatch`.

#### Scenario: Wrong category for kind

- **WHEN** a `stall` declares category `thrill`
- **THEN** loading reports `KindFieldMismatch`

#### Scenario: Unknown kind

- **WHEN** a definition declares kind `submarine`
- **THEN** loading reports `KindInvalid`

### Requirement: Footprint, placement and attach fields

Every definition SHALL declare `footprint` with `width` and `depth` (tiles, 1 to 16), `rotations` (a non-empty list of distinct values from 0 to 3) and `clearanceLevels` (1 to 63). It SHALL declare `placement` with `anchor` (`ground`), `host` (`ground` or `path`), `sceneNode` (boolean) and `maxPerPark` (integer 1 to 99 or null). It MAY declare `attach`, a list of `{kind, cell, facing}` with kind in `QueueEntry`, `RideExit`, `ShopFront`, `ParkGate`, cell `[u, v]` inside the footprint, facing one of `North`, `East`, `South`, `West`. Two attach points SHALL NOT share a cell and facing. A path addition SHALL have host `path`, footprint 1 x 1 and no attach points; every other kind SHALL have host `ground`.

#### Scenario: Attach outside the footprint

- **WHEN** a 2 x 2 definition has an attach cell `[2, 0]`
- **THEN** loading reports `AttachOutOfFootprint`

#### Scenario: Duplicate rotation

- **WHEN** `rotations` is `[0, 1, 1]`
- **THEN** loading reports `RotationsInvalid`

#### Scenario: Footprint too large

- **WHEN** `width` is 17
- **THEN** loading reports `ValueOutOfRange` naming `footprint.width`

### Requirement: Kind-specific sections

A `flat-ride` SHALL declare `economy`, `operation` (behaviour `scripted-cycle` or `digital-twin`), `ratings`, `breakdown`, `unlock`, `art`, and at least one `QueueEntry` and one `RideExit` attach point. A `coaster` SHALL declare the same except `operation.behaviour` is `coaster-delegated`, and SHALL declare `track` with `delegate` equal to `coaster-editor` and a `pieceSet` id. A `stall` and a `facility` SHALL declare `economy`, `shop` (`serves` list of tags, `serviceSeconds`, `concurrent`), `unlock`, `art` and one `ShopFront` attach point. A `path-addition` SHALL declare `economy`, `furniture` (`serves` tag), `unlock`, `art`. A `staff-building` SHALL declare `economy`, `staff` (`hosts` list of roles, `capacity`), `unlock`, `art` and one `ShopFront` attach point. A section that does not belong to the kind SHALL be `KindFieldMismatch`; a missing one `FieldMissing`.

#### Scenario: Ride without exit

- **WHEN** a flat ride has a `QueueEntry` but no `RideExit`
- **THEN** loading reports `AttachMissing` naming `RideExit`

#### Scenario: Stall with an operation section

- **WHEN** a stall contains an `operation` section
- **THEN** loading reports `KindFieldMismatch`

#### Scenario: Coaster without track

- **WHEN** a coaster has no `track` section
- **THEN** loading reports `FieldMissing` naming `track`

### Requirement: Numeric ranges and units

Money SHALL be integer cents: `buildCostCents` 0 to 100000000, `upkeepPerMinuteCents` 0 to 100000, `defaultPriceCents` 0 to 100000. Time SHALL be seconds, integer 1 to 600 (the simulation converts to ticks at 120 per second). Ratings SHALL be integer hundredths 0 to 2000. Capacity SHALL be 1 to 64. Operating `levels.count` SHALL be 1 to 9 and `levels.default` between 1 and the count. Any value outside its range SHALL be `ValueOutOfRange` naming the field; a wrong JSON type SHALL be `TypeMismatch`.

#### Scenario: Negative cost

- **WHEN** `buildCostCents` is -1
- **THEN** loading reports `ValueOutOfRange` naming `economy.buildCostCents`

#### Scenario: String where a number is required

- **WHEN** `capacity` is `"24"`
- **THEN** loading reports `TypeMismatch` naming `operation.capacity`

#### Scenario: Default level above count

- **WHEN** `levels.count` is 3 and `levels.default` is 4
- **THEN** loading reports `LevelsInvalid`

### Requirement: Fail-fast aggregated loading

At start-up the system SHALL read all embedded definitions and, when configured, all `*.json` files of one extra directory (not recursive beyond the category sub-folders). It SHALL validate every file, SHALL collect all errors with file, definition id (when known), JSON path and error code, and SHALL fail start-up with one report when any error exists. It SHALL NOT start with a partial catalog. An id present twice, across embedded and extra files or within one directory, SHALL be `IdDuplicate`; an extra file cannot override an embedded definition.

#### Scenario: Two errors in two files

- **WHEN** one file has a negative cost and another an unknown property
- **THEN** one start-up failure lists both errors

#### Scenario: Extra directory adds a definition

- **WHEN** the extra directory contains a valid `new-ride.json`
- **THEN** the catalog contains it after start-up

#### Scenario: Override is refused

- **WHEN** the extra directory contains a valid file with the id of an embedded definition
- **THEN** start-up fails with `IdDuplicate`

### Requirement: Versioning and compatibility

`version` SHALL be raised by the author whenever a change could alter behaviour of already placed objects. A definition MAY declare `compatibleFrom` (integer between 1 and `version`, default 1): placements saved with a lower version are incompatible. The catalog SHALL expose a catalog hash computed from the definitions' content (line endings normalised) in id order, so clients and tests detect a changed catalog.

#### Scenario: Incompatible saved placement

- **WHEN** a definition has `version` 3 and `compatibleFrom` 2 and a park saved with version 1 is loaded
- **THEN** the load is refused as incompatible by the grid's load rule

#### Scenario: Hash changes with content

- **WHEN** one definition's `buildCostCents` is edited
- **THEN** the catalog hash differs

#### Scenario: Hash ignores line endings

- **WHEN** the same files are checked out with CRLF line endings
- **THEN** the catalog hash is unchanged

### Requirement: Error reports are stable and readable

Every validation error SHALL have a stable code from this set: `SchemaVersionUnsupported`, `JsonInvalid`, `FieldMissing`, `FieldUnknown`, `TypeMismatch`, `IdInvalid`, `IdDuplicate`, `FileNameMismatch`, `KindInvalid`, `KindFieldMismatch`, `ValueOutOfRange`, `FootprintInvalid`, `RotationsInvalid`, `AttachOutOfFootprint`, `AttachMissing`, `OperationInvalid`, `MotionInvalid`, `LevelsInvalid`, `RatingsInvalid`, `UnlockInvalid`, `ArtInvalid`, `NameDenied`. A message SHALL name the JSON path and the offending value without echoing more than 80 characters of it.

#### Scenario: Malformed JSON

- **WHEN** a file is not valid JSON
- **THEN** loading reports `JsonInvalid` with the line and column

#### Scenario: Long value is truncated

- **WHEN** a description of 5000 characters has the wrong type
- **THEN** the error message contains at most 80 characters of the value
