# Spec Delta

## Purpose

Defines park awards: a small set of positive and negative recognitions with clear criteria, evaluated monthly, lasting a quarter-year of game time, and shifting how many guests arrive. Awards reward consistent quality and punish neglect.

## ADDED Requirements

### Requirement: Awards are defined in data with criteria and effect

Each award SHALL define an id, a polarity, a criteria expression over the park metrics and an arrival effect in permille. Seven awards ship, listed in the first scenario.

#### Scenario: Shipped awards

- **WHEN** the award definitions are read
- **THEN** `clean-park` is met by old litter at most 5 and guests at least 200 (+100)
- **AND** `happy-guests` by happy share at least 800 permille and guests at least 200 (+100)
- **AND** `best-value` by effective entrance fee at most half the value for money and guests at least 200 (+80)
- **AND** `reliable-rides` by uptime at least 980 permille and at least 4 open rides (+60)
- **AND** `dirty-park` by old litter at least 80 (-100)
- **AND** `long-queues` by average queue at least 300 s and guests at least 200 (-80)
- **AND** `unhappy-guests` by happy share at most 350 permille and guests at least 200 (-100)

#### Scenario: Opposite awards are mutually exclusive

- **WHEN** the data validation test compares each positive award with each negative award on the same metric (litter, happy share)
- **THEN** their criteria are disjoint (old litter at most 5 versus at least 80; happy share at least 800 versus at most 350), so both can never be met at the same evaluation

### Requirement: Awards are evaluated at each month close

At every month close (every fourth week close) the economy SHALL evaluate each award on the latest metrics. A met award SHALL be granted if not held, or have its expiry renewed if held, for 12 weeks. A held award SHALL end when its 12 weeks pass without renewal; failing the criteria SHALL NOT end it early.

#### Scenario: Grant

- **WHEN** at a month close the park has 300 guests and old litter 3
- **THEN** `clean-park` is granted with 12 weeks remaining

#### Scenario: Renewal

- **WHEN** `clean-park` has 4 weeks remaining and the next month close still meets the criteria
- **THEN** it has 12 weeks remaining again

#### Scenario: Expiry

- **WHEN** an award's 12 weeks pass without a month close that meets the criteria
- **THEN** it ends at the week close on which its remaining weeks reach 0

### Requirement: Awards shift arrivals through a clamped sum

The arrival effect sent to the simulation SHALL be the sum of the effects of held awards clamped to [-250, +250] permille. A change SHALL be sent as one logged command at the next tick boundary.

#### Scenario: Sum

- **WHEN** `clean-park` (+100) and `happy-guests` (+100) are held
- **THEN** the effect sent is +200

#### Scenario: Clamp

- **WHEN** held effects sum to +340
- **THEN** +250 is sent

#### Scenario: Net negative

- **WHEN** `dirty-park` (-100) and `reliable-rides` (+60) are held
- **THEN** -40 is sent

### Requirement: Awards are visible

The economy SHALL list held awards with polarity, name, remaining weeks and the metric values that earned them, and SHALL record a short history of granted and expired awards (last 50).

#### Scenario: History bound

- **WHEN** 60 award events occur
- **THEN** the last 50 are kept in order
