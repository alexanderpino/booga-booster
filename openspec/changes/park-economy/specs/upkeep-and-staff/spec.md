# Spec Delta

## Purpose

Defines the running costs of the park: weekly upkeep of rides and facilities by status, the staff roster with weekly wages, and how staff levels reach the simulation. Staff are headcounts, not entities, which keeps the model teachable.

## ADDED Requirements

### Requirement: Rides and facilities cost upkeep every week

At each week close the economy SHALL post `RideUpkeep` per ride and facility equal to its catalog weekly upkeep multiplied by the difficulty upkeep factor (permille) and by a status factor: 1 000 permille when open, testing or broken, 250 when closed. The status used SHALL be the status at the moment of the close. A demolished ride SHALL cost nothing from the moment of demolition.

#### Scenario: Open ride on normal difficulty

- **WHEN** a ride has weekly upkeep 15 000 cents, difficulty factor 1 000, and is open at the close
- **THEN** 15 000 cents of upkeep is posted for that ride

#### Scenario: Closed ride

- **WHEN** the same ride is closed at the close
- **THEN** 3 750 cents are posted

#### Scenario: Hard difficulty

- **WHEN** the factor is 1 250 and the ride is open
- **THEN** 18 750 cents are posted

#### Scenario: Demolished mid-week

- **WHEN** a ride is demolished before the close
- **THEN** no upkeep is posted for it that week

### Requirement: Staff roles, caps and wages

The roster SHALL define roles with enabled flag, weekly wage and cap: handyman (enabled, 3 000 cents, cap 50), mechanic (enabled, 5 000 cents, cap 50), security guard (disabled, 4 000) and entertainer (disabled, 3 500). Hiring or firing an enabled role SHALL change its count by the requested number within 0 and the cap. Hiring a disabled role SHALL fail with a message naming the role as not yet available.

#### Scenario: Hire handymen

- **WHEN** the player hires 3 handymen
- **THEN** the roster shows 3 handymen

#### Scenario: Over the cap

- **WHEN** the roster has 49 handymen and 2 more are requested
- **THEN** the request fails and the count stays 49

#### Scenario: Disabled role

- **WHEN** the player hires a security guard
- **THEN** the request fails and the roster is unchanged

#### Scenario: Fire more than employed

- **WHEN** 5 are fired with 3 employed
- **THEN** the request fails and the count stays 3

### Requirement: Wages are posted weekly by headcount at the close

At each week close the economy SHALL post `StaffWages` equal to the sum over roles of count times weekly wage times the difficulty wage factor (permille), per role. Staff hired during the week are paid for the whole week and staff fired during the week are not paid for it. Wages SHALL be posted even when the balance is insufficient.

#### Scenario: Wages

- **WHEN** 4 handymen and 2 mechanics are employed at factor 1 000
- **THEN** wages of 4 x 3 000 + 2 x 5 000 = 22 000 cents are posted

#### Scenario: Insufficient balance

- **WHEN** the balance is 1 000 and wages are 22 000
- **THEN** the wages are posted and the balance is -21 000 (bankruptcy rules apply separately)

### Requirement: Staff levels are sent to the simulation

A change in headcount SHALL be sent to the simulation as one logged command carrying all role counts, applied at the next tick boundary. The handyman count drives litter cleaning; the mechanic count is passed to the ride catalog for repair delay.

#### Scenario: Handymen clean

- **WHEN** 2 handymen are hired
- **THEN** the simulation's staff levels show 2 handymen at the next tick and litter cleaning throughput becomes 12 items per simulated minute

### Requirement: Construction is charged when placing and partly refunded on demolition

Placing a ride or facility SHALL charge its catalog build cost as `Construction` and fail without side effects when the balance cannot cover it. Demolition SHALL refund 50 percent of the build cost (rounded) as `ConstructionRefund`.

#### Scenario: Cannot afford

- **WHEN** the balance is 20 000 and the build cost is 30 000
- **THEN** placement fails, the ride is not created and the balance is unchanged

#### Scenario: Refund

- **WHEN** a ride built for 30 000 is demolished
- **THEN** 15 000 is refunded
