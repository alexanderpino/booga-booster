# Spec Delta

## Purpose

Defines marketing campaigns the operator can buy to attract more guests: their types, prices, durations, the arrival demand they add and the entrance discount one of them gives. Campaigns are data-driven and paid up front.

## ADDED Requirements

### Requirement: Campaign types are defined in data

Each campaign type SHALL define an id, a weekly price in cents, an added arrival demand in milli-guests per second and an entrance discount in permille. The shipped types SHALL be `park-advertising` (35 000 cents per week, 400 milli-guests per second, no discount) and `entry-discount` (12 000 cents per week, 300 milli-guests per second, 500 permille discount).

#### Scenario: Shipped types

- **WHEN** the campaign catalogue is read
- **THEN** it lists the two types with these values

#### Scenario: Invalid data

- **WHEN** a type has a negative price or a discount above 1 000 permille
- **THEN** start-up fails naming the type

### Requirement: Starting a campaign pays up front for 1 to 12 weeks

Starting a campaign for w weeks (1 to 12) SHALL charge w times the weekly price as `Marketing` at once and activate it with w weeks remaining. It SHALL fail without side effects if the balance cannot cover it, if w is out of range, or if two other campaigns are already active. Starting a type that is already active SHALL extend it, capping the remaining weeks at 12, and charge only for the added weeks.

#### Scenario: Start

- **WHEN** park advertising is started for 4 weeks with a balance of 200 000
- **THEN** 140 000 is charged and the campaign has 4 weeks remaining

#### Scenario: Extend

- **WHEN** 3 weeks remain and 4 more weeks are requested
- **THEN** 7 weeks remain and 4 weeks are charged

#### Scenario: Cap

- **WHEN** 10 weeks remain and 5 more are requested
- **THEN** only 2 weeks are added and charged

#### Scenario: Too many campaigns

- **WHEN** two different campaigns are active and a third type is requested
- **THEN** the request fails

### Requirement: Active campaigns add demand and may discount the fee

The sum of the added demand of all active campaigns SHALL be sent to the simulation as marketing demand (milli-guests per second converted to the simulation's unit), and the largest active discount SHALL reduce the effective entrance fee (see pricing). The entry discount demand SHALL be divided by 8 (integer, rounding down) while the effective fee after discount is 300 cents or less, because a discount on a nearly free entrance attracts little.

#### Scenario: Two campaigns

- **WHEN** both shipped types are active and the plan fee is 1 000
- **THEN** demand is 700 milli-guests per second and the effective fee is 500

#### Scenario: Pointless discount

- **WHEN** the entry discount is active and the plan fee is 500 (effective 250)
- **THEN** its demand counts as 37 milli-guests per second

### Requirement: Campaigns age at the week close

At each week close every active campaign SHALL lose one remaining week and end when none remain; its demand and discount SHALL stop at the same tick boundary. A campaign started mid-week SHALL count that partial week as one of its weeks.

#### Scenario: Last week

- **WHEN** a campaign has 1 week remaining at the close
- **THEN** it ends and the next demand sent to the simulation excludes it

#### Scenario: Partial week counts

- **WHEN** a 1-week campaign is started 10 seconds before the close
- **THEN** it ends at that close
