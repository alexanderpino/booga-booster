# Spec Delta

## Purpose

Defines the integration events the economy publishes over the existing Dapr pub/sub conventions so other modules and services can follow money, rating and scenario changes without referencing the economy: topics, payloads, ordering numbers, coalescing and delivery rules.

## ADDED Requirements

### Requirement: Three events are declared in the shared messages library

The shared library SHALL declare `MoneyChangedIntegrationEvent` (topic `money-changed`), `RatingChangedIntegrationEvent` (topic `rating-changed`) and `ScenarioStatusChangedIntegrationEvent` (topic `scenario-status-changed`) as sealed records under `Events/Economy/`, each implementing the integration event marker and carrying a topic name attribute, following the existing conventions.

#### Scenario: Topic names

- **WHEN** the topic resolver reads each event type
- **THEN** it returns `money-changed`, `rating-changed` and `scenario-status-changed`

#### Scenario: Conventions test

- **WHEN** the messaging conventions test scans the events assembly
- **THEN** every economy event has the suffix `IntegrationEvent`, the marker interface and a unique topic name

### Requirement: Payloads are small and carry ordering numbers

Each event SHALL carry the park id, a sequence number that increases by 1 per event type per park, the simulation tick and an occurred-at time, plus the type-specific fields listed in the first scenario.

#### Scenario: Field lists

- **WHEN** the three event contracts are read
- **THEN** money-changed adds balance in cents, delta in cents since the previous event, reason (`Flush`, `WeekClose`, `Purchase`, `Loan`, `Refund`, `ScenarioStart`) and week index
- **AND** rating-changed adds rating, previous rating and band
- **AND** scenario-status-changed adds scenario id, status (`Running`, `Won`, `Lost`) and reason

#### Scenario: Money payload

- **WHEN** a week close changes the balance from 120 000 to 131 500
- **THEN** the event has balanceCents 131 500, deltaCents 11 500, reason WeekClose and the week index

#### Scenario: Sequences

- **WHEN** three money events are published
- **THEN** their sequence numbers are n, n+1 and n+2

### Requirement: Events are coalesced to stay quiet

Money events SHALL be published for each week close, each operator transaction and each loan or refund movement, and flush-only changes SHALL be coalesced into at most one event per 12 simulated seconds. Rating events SHALL be published only when the rating changes by at least 5 points since the last published rating, or the band changes. A scenario status event SHALL be published on every status change.

#### Scenario: Small rating drift

- **WHEN** the rating moves from 500 to 503 and then to 506
- **THEN** one event is published at 506 (change of 6 since the last published value)

#### Scenario: Band change

- **WHEN** the rating moves from 549 to 551
- **THEN** an event is published because the band changes from Fair to Good

#### Scenario: Flush coalescing

- **WHEN** two flushes occur within the same 12 s with no transaction
- **THEN** at most one money event results

### Requirement: Publishing never blocks the simulation and is at least once

Events SHALL be placed in a bounded in-memory outbox (capacity 1 000, drop oldest with a metric) and published by a background service through the integration event publisher after the tick. A failed publish SHALL be retried three times after 200 ms, 1 s and 5 s, then dropped with an error log and a metric. Consumers SHALL treat events with a sequence not above the last seen as duplicates.

#### Scenario: Slow broker

- **WHEN** the publisher blocks for 5 s
- **THEN** simulation tick time is unchanged and events queue in the outbox

#### Scenario: Overflow

- **WHEN** 1 100 events are queued while publishing is blocked
- **THEN** the oldest 100 are dropped and the dropped counter is 100

#### Scenario: Retry

- **WHEN** the first publish attempt throws and the second succeeds
- **THEN** the event is delivered once

### Requirement: A subscriber example documents consumption

The messages library README SHALL describe the three topics and the idempotency rule, and a test-only subscriber SHALL show the minimal-API pattern with `WithTopic` for one of them.

#### Scenario: README

- **WHEN** the README is read
- **THEN** it lists the three topics with their payload fields and the duplicate rule
