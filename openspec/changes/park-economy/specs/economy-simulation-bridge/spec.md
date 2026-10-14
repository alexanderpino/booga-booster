# Spec Delta

## Purpose

Defines how the economy stays in step with the simulation: the in-game calendar, the 12-second revenue flush, the weekly and monthly close, how operator commands are applied, and how economy inputs reach the simulation. Everything is tick-aligned so runs are reproducible.

## ADDED Requirements

### Requirement: The calendar is derived from the tick count

Simulation time SHALL be divided into weeks of 11 520 ticks (96 s at 120 Hz, eight flush periods), months of 4 weeks and years of 12 months, numbered from week 1. The week length SHALL be configurable (`Economy:Calendar:TicksPerWeek`), validated to be a positive multiple of 1 440 so the flush and the week close stay aligned. The calendar SHALL depend only on the tick, never on the wall clock.

#### Scenario: Boundaries

- **WHEN** ticks 11 519 and 11 520 are completed
- **THEN** week 1 closes after tick 11 520 and week 2 begins

#### Scenario: Month and year

- **WHEN** week 4 closes
- **THEN** month 1 closes at the same moment; when week 48 closes, year 1 closes

#### Scenario: Invalid configuration

- **WHEN** `TicksPerWeek` is 10 000
- **THEN** options validation fails because 10 000 is not a multiple of 1 440

### Requirement: Economy work runs after the tick on the simulation thread

The economy SHALL do its periodic work in a hook invoked after each completed tick, on the same thread, before the next tick starts. The hook SHALL do in-memory work only; file and network input and output SHALL happen elsewhere. The work SHALL complete within 2 ms for a flush and 5 ms for a week close in the reference scenario (targets recorded by a benchmark; more than twice a target stops the change for review).

#### Scenario: Order of work

- **WHEN** tick 11 520 completes
- **THEN** the flush for that tick runs first, then the week close, then (if the week is the fourth) the month close, and the next tick starts only afterwards

#### Scenario: No I/O in the hook

- **WHEN** the hook runs with a storage fake that throws on any call
- **THEN** the hook completes without calling it

### Requirement: Revenue is flushed every 12 simulated seconds

Every 1 440 ticks the economy SHALL read and reset the simulation's revenue accumulators through one logged command, post the income to the open week (entrance fees, ride tickets per ride, shop sales), read the park metrics, compute the rating and send changed inputs to the simulation. A flush with zero revenue SHALL still compute the rating.

#### Scenario: Flush posts income

- **WHEN** the accumulators hold entrance 3 000, rides 5 400 (two rides) and shops 1 200
- **THEN** the balance rises by 9 600 and the accumulators read zero afterwards

#### Scenario: Idle park

- **WHEN** no revenue arrived in the period
- **THEN** the rating step still runs

### Requirement: The week close posts expenses and advances time-based state

At every week close the economy SHALL, in this order: post upkeep, wages, research funding and loan interest; progress research and complete items; age campaigns; evaluate scenario goals and bankruptcy; compose the ledger lines; emit events. At every month close it SHALL then evaluate awards and compose monthly summaries.

#### Scenario: Expense order is fixed

- **WHEN** a week closes with upkeep, wages, research and interest all non-zero
- **THEN** the ledger lines list them in that category order, identically on every run

#### Scenario: Goals see the closed week

- **WHEN** rating and guests meet a goal at the close
- **THEN** the goal counter increments in the same close, after expenses are posted

### Requirement: Operator commands are applied at tick boundaries in order

Operator commands (prices, loans, staff, research, marketing, scenario start) SHALL be placed in an inbox and applied by the hook in arrival order at the next tick boundary, each completing its caller with the result or error. Every economy state change SHALL therefore occur on the simulation thread, and the command SHALL be written to the input log with the tick at which it applied. Queries SHALL read an immutable projection published after every flush and every applied command.

#### Scenario: Latency

- **WHEN** a price is set through the API while the simulation runs
- **THEN** the response arrives after the next tick boundary, at most one tick plus request handling later

#### Scenario: Order

- **WHEN** a borrow and a purchase arrive in this order within one tick
- **THEN** they apply in this order

#### Scenario: Projection read

- **WHEN** a summary is requested during a tick
- **THEN** it returns the last published projection without waiting for the tick

### Requirement: Economy inputs to the simulation are logged commands

Economy outputs to the simulation (effective entrance fee, ride prices, rating, award permille, marketing demand, staff levels, ride type unlocks, revenue read-and-reset) SHALL each be a simulation gateway command recorded in the simulation's input log, sent only on change, so replaying the log reproduces the run.

#### Scenario: Only on change

- **WHEN** two successive flushes yield the same award and demand values
- **THEN** no command is sent for them on the second flush

#### Scenario: Replay

- **WHEN** a recorded 12-month run is replayed from the economy command log and the simulation input log
- **THEN** the final economy state hash and the final world hash equal the recorded ones

### Requirement: The economy requires the park mode

With `Economy:Enabled=true` the host SHALL require `Simulation:Guests:Mode=Park`; any other combination SHALL fail start-up with a message naming both keys. With the economy disabled the simulation runs with the documented default inputs.

#### Scenario: Conflicting configuration

- **WHEN** the economy is enabled and the guest mode is `LegacyQueue`
- **THEN** start-up fails naming both keys

#### Scenario: Economy off

- **WHEN** `Economy:Enabled` is false
- **THEN** no economy endpoint is mapped and the simulation uses its default entrance fee 1 000, rating 500 and zero modifiers

### Requirement: The economy state has a canonical hash

The economy SHALL provide a canonical hash over its aggregates (balance, loan, ledger, prices, staff, research, campaigns, awards, scenario, rating) computed in a fixed order from integer values only, for determinism tests and replay.

#### Scenario: Stable hash

- **WHEN** the same sequence of commands and flushes runs twice
- **THEN** the hashes are equal

#### Scenario: Sensitivity

- **WHEN** a single cent differs in any ledger line
- **THEN** the hashes differ
