# Spec Delta

## Purpose

Defines research and development: funding levels, weekly progress, the ordered list of researchable items, the focus category, and how finished items unlock ride types in the catalog. A cash shortcut supports short workshops.

## ADDED Requirements

### Requirement: Research items form an ordered list defined in data

The research programme SHALL load an ordered list of items from data, each with an id, a category (gentle, thrill, water, transport, shop, coaster), a point cost and the catalog ride type ids it unlocks. Item ids SHALL be unique and unlocked type ids SHALL exist in the catalog; otherwise start-up fails with a message naming the item.

#### Scenario: Valid data

- **WHEN** the data lists 12 items with unique ids and known type ids
- **THEN** the programme loads and reports 12 items, none completed

#### Scenario: Unknown ride type

- **WHEN** an item unlocks a type id that the catalog does not define
- **THEN** start-up fails and the message names the item and the type

### Requirement: Funding level sets weekly cost and progress

The funding levels SHALL be None (0 cents, 0 points per week), Basic (10 000 cents, 40 points), Normal (25 000 cents, 100 points) and Maximum (50 000 cents, 220 points), the cost multiplied by the difficulty research factor (permille). At each week close the cost SHALL be posted as `Research` and the points added to the current item. A funding change SHALL apply from the next close.

#### Scenario: Normal funding

- **WHEN** funding is Normal at factor 1 000 for one week close
- **THEN** 25 000 cents of research expense is posted and the current item gains 100 points

#### Scenario: None

- **WHEN** funding is None
- **THEN** nothing is posted and no progress is made

#### Scenario: Hard difficulty cost

- **WHEN** the factor is 1 250 and funding is Normal
- **THEN** 31 250 cents are posted

### Requirement: Items complete in order with carry-over and focus

The current item SHALL be the first uncompleted item of the focus category if a focus is set and such an item exists, otherwise the first uncompleted item in list order. When progress reaches the item's cost the item SHALL complete, its ride types SHALL be unlocked, surplus points SHALL carry over to the next current item, and several items MAY complete in one close.

#### Scenario: Carry-over

- **WHEN** the current item costs 100 points, has 90, and the week adds 100
- **THEN** it completes and the next item starts with 90 points

#### Scenario: Focus

- **WHEN** the focus is thrill and the first uncompleted item is gentle while a later item is thrill
- **THEN** the thrill item becomes the current item

#### Scenario: Several in one week

- **WHEN** 220 points arrive and the next two items cost 80 and 100
- **THEN** both complete and 40 points carry over

### Requirement: Unlocking changes catalog availability

A completed item SHALL send one logged command per unlocked ride type to the simulation/catalog, making the type placeable. Unlocking an already available type SHALL be a no-op. Items unlocked by the scenario at start SHALL be marked complete.

#### Scenario: Unlock

- **WHEN** an item that unlocks type `swing-ship` completes
- **THEN** one unlock command for `swing-ship` is recorded and the type becomes placeable

#### Scenario: Double unlock

- **WHEN** the type is already available
- **THEN** no error occurs and no extra command is recorded

### Requirement: The cash shortcut unlocks an item immediately

When the difficulty profile enables unlock-by-cash, the player SHALL be able to unlock any uncompleted item at once for its remaining points times 500 cents, charged as `Research`, failing when the balance cannot cover it. The shortcut SHALL be disabled on profiles that do not enable it.

#### Scenario: Pay to unlock

- **WHEN** an item has 60 points remaining and the balance is 100 000
- **THEN** 30 000 cents are charged and the item completes

#### Scenario: Disabled profile

- **WHEN** the profile does not enable the shortcut
- **THEN** the request fails with a message naming the profile

### Requirement: Research ends cleanly

When all items are complete, funding SHALL become None automatically and the status SHALL report all researched.

#### Scenario: Exhausted list

- **WHEN** the last item completes
- **THEN** funding is None and the status is AllResearched
