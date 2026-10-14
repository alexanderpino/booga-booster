# Spec Delta

## Purpose

Defines how the catalog plugs into the park: it supplies placeable definitions, creates ride runtime when something is placed, gates definitions behind an unlock hook, binds art by asset id with a placeholder fallback, publishes a read API, and lets a new definition be added without changing code.

## ADDED Requirements

### Requirement: The catalog supplies the park's placeable definitions

Every catalog definition SHALL be placeable through the park's build commands with its footprint, rotations, clearance, build cost, anchor, host, scene-node flag, per-park limit, attach points, version and compatibility floor. A definition SHALL be refused with `DefinitionLocked` while the unlock hook reports it locked. The built-in `path`, `queue-path` and `park-entrance` definitions SHALL remain provided by the park grid; a catalog definition with one of those ids SHALL be an `IdDuplicate` load error.

#### Scenario: Place a carousel

- **WHEN** a `starlight-carousel` is placed on a free flat 6 x 6 area
- **THEN** the placement succeeds with cost 1200000 and the area is occupied

#### Scenario: Locked definition

- **WHEN** the unlock hook reports `looper-coaster` locked and it is placed
- **THEN** the placement is refused with `DefinitionLocked`

#### Scenario: Reserved id

- **WHEN** a catalog file declares id `park-entrance`
- **THEN** loading reports `IdDuplicate`

### Requirement: Placement creates ride runtime

When a ride-kind definition is placed, the catalog SHALL cause the ride runtime to be created for that placement: lifecycle Idle, status Closed, level set to the default, occupancy 0 of capacity, reliability 10000, ratings per the definition, and for scripted motion a child scene node named `joint_motion` under the placement node. Stalls, facilities, path additions and staff buildings SHALL get only the shop, furniture or staff data they declare. The ride instance id SHALL be the placement serial. Removing the object SHALL remove its runtime and joint node; removal SHALL be refused as busy while the ride is not Idle or has riders aboard.

#### Scenario: Runtime appears with the object

- **WHEN** a carousel is placed with serial 5
- **THEN** ride 5 exists, is Closed and Idle, and has a child node `joint_motion` under `starlight_carousel_5`

#### Scenario: Busy removal

- **WHEN** ride 5 is in Loading with riders aboard and its removal is requested
- **THEN** removal is refused with `ObjectBusy`

#### Scenario: Idle removal

- **WHEN** ride 5 is Idle and removed
- **THEN** ride 5 no longer exists and its scene nodes are gone

### Requirement: Unlock and prerequisite hook

Each definition SHALL carry `unlock.initiallyUnlocked` (boolean) and `unlock.prerequisites` (a list of opaque strings matching `^[a-z][a-z0-9-]*:[a-z0-9-]+$`, such as `research:thrill-1`). The catalog SHALL ask an unlock provider whether a definition is unlocked and why not. The default provider SHALL report every definition unlocked. The catalog SHALL not interpret prerequisite strings.

#### Scenario: Default provider

- **WHEN** no provider is registered by another module
- **THEN** every definition is unlocked

#### Scenario: Reason is reported

- **WHEN** a provider reports `looper-coaster` locked with reason `needs research:thrill-1`
- **THEN** the read API shows locked true and that reason

#### Scenario: Bad prerequisite string

- **WHEN** a prerequisite is `Research 1`
- **THEN** loading reports `UnlockInvalid`

### Requirement: Art binding with placeholder fallback

Each definition SHALL carry `art` with `assets` (a list of asset manifest ids matching `^[a-z][a-z0-9-]*$`, possibly empty), `placeholder` (a key of a registered procedural builder, default `catalog-box`) and `tint` (a `#rrggbb` colour). When all listed assets are ready the client SHALL show them; when the list is empty, an asset is missing from the manifest, or loading fails, the client SHALL show the placeholder. The `catalog-box` placeholder SHALL be a box with the footprint's width and depth and the clearance as height, in the tint, with a visible stripe so rotation can be seen. No definition SHALL fail to appear for lack of art.

#### Scenario: No art

- **WHEN** a definition has an empty `assets` list
- **THEN** a tinted box of its footprint size is shown

#### Scenario: Asset missing from the manifest

- **WHEN** a definition lists `octopus-arm` and the manifest has no such id
- **THEN** the placeholder is shown and the failure is logged once

#### Scenario: Asset ready

- **WHEN** every listed asset loads
- **THEN** the assets are shown and the placeholder is not

#### Scenario: Bad tint

- **WHEN** `tint` is `red`
- **THEN** loading reports `ArtInvalid`

### Requirement: Read API

The system SHALL expose `GET /catalog/definitions` (all definitions with computed statistics: ratings at the default level, value, throughput, locked flag and reason, catalog hash) and `GET /catalog/definitions/{id}`. Responses SHALL carry an `ETag` equal to the catalog hash plus the unlock state revision, and SHALL honour `If-None-Match`. An unknown id SHALL return 404, a malformed id 400.

#### Scenario: List

- **WHEN** `GET /catalog/definitions` is requested
- **THEN** 20 definitions are returned with the catalog hash

#### Scenario: Not modified

- **WHEN** the request repeats with the returned ETag
- **THEN** the response is 304

#### Scenario: Unknown id

- **WHEN** `GET /catalog/definitions/no-such` is requested
- **THEN** the response is 404

### Requirement: Adding a ride needs no engine code

Placing a new `scripted-cycle` flat ride, stall, facility, path addition or staff building SHALL require only a valid definition file (and optionally art assets), with no change to C# or TypeScript code. After restarting the server with the file in the extra directory, the definition SHALL appear in the picker, be placeable, and for a ride be operable with ratings and motion from its data.

#### Scenario: Agent adds a ride

- **WHEN** a valid `scripted-cycle` definition `moon-hopper.json` is added to the extra directory and the server restarted
- **THEN** `moon-hopper` appears in `GET /catalog/definitions`, can be placed, opened and dispatched, and shows a placeholder box

#### Scenario: Invalid file blocks start-up with a clear report

- **WHEN** the new file has a negative cost
- **THEN** start-up fails and the report names `moon-hopper.json`, `economy.buildCostCents` and `ValueOutOfRange`

### Requirement: Saved placements keep working across catalog edits

A park saved with definition versions SHALL load when each saved version is at least the definition's `compatibleFrom`, and SHALL otherwise be refused by the grid's load rule. A ride placed from an older compatible version SHALL use the current definition's data.

#### Scenario: Compatible older version

- **WHEN** a park saved with `starlight-carousel` version 1 is loaded and the definition is now version 2 with `compatibleFrom` 1
- **THEN** the park loads and the carousel uses the version 2 data

#### Scenario: Incompatible older version

- **WHEN** the definition now has `compatibleFrom` 2
- **THEN** the load is refused as incompatible
