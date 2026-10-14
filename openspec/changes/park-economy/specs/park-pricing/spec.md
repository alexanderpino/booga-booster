# Spec Delta

## Purpose

Defines how the operator sets the entrance fee and the price of each ride, the validation of those prices, the advice shown against ride value, and how prices reach the simulation so guests react to them.

## ADDED Requirements

### Requirement: The entrance fee is a validated plan value

The park SHALL have one entrance fee between 0 and 10 000 cents inclusive, in steps of 10 cents, defaulting to 1 000 cents. Setting a value outside the range or off the step SHALL fail with a validation error (HTTP 400) and leave the fee unchanged. Setting the current value SHALL be a no-op that publishes nothing.

#### Scenario: Valid change

- **WHEN** the operator sets the fee to 1 250
- **THEN** the fee is 1 250 and the change is recorded in the price history with the current week

#### Scenario: Off-step value

- **WHEN** the operator sets the fee to 1 255
- **THEN** the request fails with a validation error naming the step and the fee stays unchanged

#### Scenario: No-op

- **WHEN** the operator sets the fee to its current value
- **THEN** no history entry, no simulation command and no event result

### Requirement: Each ride has a validated price

Each ride SHALL have a price between 0 and 5 000 cents inclusive in steps of 10 cents. A newly placed ride SHALL receive the default price of its catalog definition. Setting the price of an unknown ride SHALL fail with not found (HTTP 404). Other rules are as for the entrance fee.

#### Scenario: Default from the catalog

- **WHEN** a ride whose definition has default price 300 is placed
- **THEN** its price is 300

#### Scenario: Unknown ride

- **WHEN** the price of a ride id that does not exist is set
- **THEN** the response is not found and nothing changes

#### Scenario: Demolished ride

- **WHEN** a ride is demolished
- **THEN** its price entry is removed

### Requirement: Prices reach the simulation at the next tick boundary

A changed fee or ride price SHALL be sent to the simulation as one logged command and take effect at the next tick boundary. The fee sent SHALL be the effective fee: the plan fee reduced by any active entry discount (see marketing), rounded by the money rounding rule. Unchanged prices SHALL NOT be re-sent.

#### Scenario: Effective fee with discount

- **WHEN** the plan fee is 1 000 and an entry discount of 500 permille is active
- **THEN** the simulation receives an effective entrance fee of 500

#### Scenario: Price applied once

- **WHEN** a ride price is set to 400 and nothing else changes for 100 ticks
- **THEN** exactly one price command is recorded in the input log

### Requirement: Price advice compares price with ride value

For each ride the economy SHALL report advice from the ratio of price to ride value (value supplied by the catalog): `Low` when price is at most half the value, `Fair` when at most the value, `High` when at most 1.5 times the value, otherwise `TooHigh`. A value of 0 SHALL give `TooHigh` for any price above 0 and `Fair` for a price of 0.

#### Scenario: Thresholds

- **WHEN** the value is 400 and prices are 200, 400, 600 and 610
- **THEN** advice is Low, Fair, High and TooHigh

#### Scenario: Zero value

- **WHEN** the value is 0 and the price is 10
- **THEN** advice is TooHigh

### Requirement: Price history is bounded

The economy SHALL keep the last 50 price changes (fee and rides together) with week index and previous and new value, oldest dropped first.

#### Scenario: Bound

- **WHEN** 60 price changes are made
- **THEN** the history holds the latest 50 in order
