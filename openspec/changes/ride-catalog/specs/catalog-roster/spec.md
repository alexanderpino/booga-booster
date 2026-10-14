# Spec Delta

## Purpose

Fixes the set of definitions that ships with the game and the values each must carry, so the catalog is useful from the first run and the numbers are reviewable in one place. All names and numbers are original and are tuning starting points, not copies of any other game.

## ADDED Requirements

### Requirement: The shipped roster

The catalog SHALL ship exactly these 20 definitions, all with `version` 1, `compatibleFrom` 1 and initially unlocked: flat rides `octopus`, `starlight-carousel`, `corsair-swing`, `whirlwind-twirl`, `skyline-wheel`, `bumper-arena`, `haunted-manor`; coasters `family-coaster`, `looper-coaster`; stalls `sausage-grill`, `fizz-kiosk`, `frost-cart`, `trinket-shop`, `info-kiosk`; facility `restroom`; path additions `bench`, `litter-bin`, `lamp-post`; staff buildings `staff-lodge`, `repair-workshop`. Every one SHALL pass the validator.

#### Scenario: Roster loads

- **WHEN** the catalog loads with no extra directory
- **THEN** it contains exactly these 20 ids and no validation error

#### Scenario: Names are original

- **WHEN** the roster names and descriptions are checked against the denylist file `docs/park/denylist-names.txt`
- **THEN** none matches (the denylist holds third-party ride and product names that must not be used)

### Requirement: Flat ride values

The flat rides SHALL carry these values (footprint width x depth in tiles; clearance in levels of 0.25 m; capacity in guests; seconds; money in cents): 

| id | category | footprint | clearance | capacity | load min/max | ride | ramp up/down | unload | build | upkeep/min | price |
|---|---|---|---|---|---|---|---|---|---|---|---|
| octopus | thrill | 10 x 10 | 40 | 32 | n/a | 60 | n/a | n/a | 4500000 | 600 | 300 |
| starlight-carousel | gentle | 6 x 6 | 20 | 24 | 15/45 | 75 | 5/6 | 15 | 1200000 | 150 | 200 |
| corsair-swing | thrill | 3 x 7 | 36 | 20 | 20/50 | 60 | 6/8 | 15 | 950000 | 220 | 300 |
| whirlwind-twirl | thrill | 5 x 5 | 24 | 16 | 15/40 | 50 | 5/5 | 12 | 800000 | 180 | 250 |
| skyline-wheel | gentle | 7 x 2 | 56 | 24 | 20/60 | 120 | 8/10 | 20 | 1400000 | 160 | 250 |
| bumper-arena | gentle | 6 x 5 | 16 | 12 | 20/50 | 90 | 0/0 | 15 | 700000 | 120 | 200 |
| haunted-manor | gentle | 4 x 4 | 24 | 10 | 20/60 | 100 | 0/0 | 15 | 900000 | 140 | 250 |

The `octopus` clearance of 40 levels (10 m) and footprint of 10 x 10 are derived from `RideParameters` at apply time (main arm 6 m, hub arm 2 m, seat offset 0.5 m, margin 1.0 m gives a swept radius of 9.5 m; maximum height is tower plus arm rise plus gondola) and SHALL be recomputed and recorded in the docs; the table value is the default if that derivation yields no larger value. The `octopus` has `maxPerPark` 1; all other rides have `maxPerPark` null.

#### Scenario: Table values present

- **WHEN** the `starlight-carousel` definition is read
- **THEN** its footprint is 6 x 6, capacity 24, ride time 75 s and build cost 1200000

#### Scenario: Octopus is limited to one

- **WHEN** the `octopus` definition is read
- **THEN** its `maxPerPark` is 1 and its behaviour is `digital-twin`

### Requirement: Flat ride attach points, motion and levels

Every flat ride SHALL have its `QueueEntry` and `RideExit` on its West side at rotation 0 (facing West) except `haunted-manor`, whose `RideExit` faces East. Attach cells (u, v): octopus `(0,4)` and `(0,5)`; starlight-carousel `(0,2)` and `(0,3)`; corsair-swing `(0,3)` and `(0,4)`; whirlwind-twirl `(0,1)` and `(0,2)`; skyline-wheel `(0,0)` and `(0,1)`; bumper-arena `(0,1)` and `(0,3)`; haunted-manor entry `(0,1)` and exit `(3,1)`. Motion and levels: starlight-carousel spin about Y at 0.6 rad/s at the top level, 3 levels, default 2; corsair-swing swing about X, amplitude 1.1 rad at the top level, period 6 s, 5 levels, default 3; whirlwind-twirl spin about Y at 1.8 rad/s, 3 levels, default 2; skyline-wheel spin about Z at 0.2 rad/s, 1 level; bumper-arena and haunted-manor motion none, 1 level; octopus has no scripted motion (its motion is the digital twin's).

#### Scenario: Rotated attach

- **WHEN** a `starlight-carousel` is placed at anchor `(10, 10)` with rotation 0
- **THEN** its queue entry cell is `(10, 12)` facing West and its exit cell is `(10, 13)` facing West

#### Scenario: Carousel speed at default level

- **WHEN** the carousel runs at its default level 2 of 3
- **THEN** its target spin rate is 0.4 rad/s

### Requirement: Rating bases

Rating bases (excitement, intensity, nausea in hundredths) and per-level steps SHALL be: starlight-carousel base 95/20/25, step 10/5/5; corsair-swing base 170/230/160, step 25/40/30; whirlwind-twirl base 140/160/210, step 30/25/40; skyline-wheel base 130/30/20, step 0/0/0; bumper-arena base 110/90/40, step 0/0/0; haunted-manor base 160/60/50, step 0/0/0. The octopus SHALL use the simulation-derived model with base 250/220/180 and a provisional baseline recorded from a measured reference run at apply time (placeholder 300/320/250 until then). Coasters: `family-coaster` provisional 200/180/100, `looper-coaster` provisional 320/380/240, delegated model. Value weights (excitement/intensity/nausea) SHALL be 40/20/10 for all rides.

#### Scenario: Swing at default level

- **WHEN** `corsair-swing` runs at its default level 3
- **THEN** its ratings are excitement 220, intensity 310, nausea 220

#### Scenario: Wheel ignores levels

- **WHEN** `skyline-wheel` is read at any level
- **THEN** its ratings are 130, 30, 20

### Requirement: Coaster entries

`family-coaster` SHALL have a 6 x 3 station footprint, clearance 16, capacity 16, build cost 1500000, piece set `family-v1`, trackPieceCostCents 3000 and supportCostCents 250; `looper-coaster` an 8 x 3 station, clearance 20, capacity 24, build cost 3000000, piece set `looper-v1`, trackPieceCostCents 4500 and supportCostCents 300. Both SHALL have `QueueEntry` at `(0,1)` facing West and `RideExit` at the far end `(5,1)` and `(7,1)` facing East, ride price 300 and 400, upkeep 250 and 400 cents per minute.

#### Scenario: Coaster is a type entry

- **WHEN** `looper-coaster` is read
- **THEN** its kind is `coaster`, its track section delegates to `coaster-editor` with piece set `looper-v1`, and its operation behaviour is `coaster-delegated`

### Requirement: Stalls, facility, furniture and staff buildings

Values (footprint; clearance; build; upkeep per minute; price; serves; serviceSeconds; concurrent): `sausage-grill` 1 x 1; 12; 35000; 30; 250; food; 8; 2. `fizz-kiosk` 1 x 1; 12; 30000; 25; 150; drink; 5; 2. `frost-cart` 1 x 1; 12; 28000; 20; 200; treat; 6; 2. `trinket-shop` 2 x 1; 12; 60000; 40; 600; souvenir; 10; 3. `info-kiosk` 1 x 1; 12; 25000; 15; 100; information; 4; 2. `restroom` 2 x 1; 12; 22500; 35; 20; toilet; 20; 2. Path additions (1 x 1, host path): `bench` 1500 cents, serves rest; `litter-bin` 1000, litter; `lamp-post` 1200, light; upkeep 0. Staff: `staff-lodge` 3 x 2, clearance 12, build 80000, upkeep 50, hosts handyman, mechanic, entertainer, capacity 6; `repair-workshop` 2 x 2, clearance 12, build 55000, upkeep 40, hosts mechanic, capacity 3. Every stall, the restroom and both staff buildings SHALL have a `ShopFront` at cell `(0,0)` facing South. Serve tags SHALL be meaning-free to this change; `guests-and-needs` maps them to needs.

#### Scenario: Shop front facing

- **WHEN** a `sausage-grill` is placed at anchor `(8, 8)` with rotation 1
- **THEN** its shop front faces East after rotation (South maps to East under one quarter turn) from cell `(8, 8)`, so the path tile to connect is `(9, 8)`

#### Scenario: Bench

- **WHEN** `bench` is read
- **THEN** its host is `path`, its footprint 1 x 1 and its build cost 1500
