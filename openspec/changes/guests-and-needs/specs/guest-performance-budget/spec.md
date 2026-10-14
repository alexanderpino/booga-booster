# Spec Delta

## Purpose

Defines the measurable performance budgets for the guest simulation: population targets, per-tick time, memory, allocation, stream cost, the benchmark scenarios that measure them, and what happens when a budget is missed. Budgets are targets to be measured, not facts.

## ADDED Requirements

### Requirement: The population target is 5 000 guests with a 10 000 stress ceiling

The simulation SHALL sustain 5 000 concurrent guests within the tick budget on the reference machine, and SHALL survive 10 000 without failing a tick or exceeding its stress budget. The design records why 5 000 (ADR game-0005 and the roadmap demo, park capacity arithmetic) and how it is measured.

#### Scenario: Stress scenario

- **WHEN** the benchmark scenario "park-5000" (5 000 pre-placed guests, 60 destinations, 20 000-node graph, seed 42) runs 1 200 ticks
- **THEN** it completes, the mean guest-system time per tick is recorded, and no tick exceeds the 8.33 ms fixed-step period for guest systems alone

#### Scenario: Ceiling scenario

- **WHEN** "park-10000" runs 1 200 ticks
- **THEN** it completes without exceptions and mean guest-system time per tick is at most 3 ms on one worker

### Requirement: Per-tick guest cost stays within the budget table

At 5 000 guests on one worker the mean time per tick SHALL be at most: movement 0.25 ms, needs cohort 0.10 ms, decisions 0.10 ms, queue ranking and boarding 0.10 ms, spawn and despawn 0.10 ms, stream encode for one client 0.30 ms, projection publish 0.20 ms amortised, in total at most 1.5 ms (18 % of the 8.33 ms tick). Exceeding any line by more than a factor of 2 SHALL stop the change for review. Timing SHALL NOT be asserted in unit tests.

#### Scenario: Benchmark report

- **WHEN** the guest benchmark suite runs
- **THEN** it reports one mean per line of the table and the doc records CPU model, runtime version and results

#### Scenario: Review trigger

- **WHEN** the measured movement cost is 0.6 ms
- **THEN** the review trigger (more than twice 0.25 ms) fires and is recorded in the apply notes

### Requirement: Iteration allocates nothing in steady state

After warm-up, guest systems SHALL allocate zero bytes on the managed heap per tick, verified by a machine-independent allocation test over 1 200 ticks of the 600-guest scenario.

#### Scenario: Allocation test

- **WHEN** the allocation test runs
- **THEN** allocated bytes before and after the measured ticks are equal

### Requirement: Memory per guest is bounded

The hot per-guest components SHALL total at most 192 bytes per guest, so 10 000 guests need at most 1.92 MB of hot component storage plus path fields (60 fields x 20 000 nodes x 8 bytes = 9.6 MB).

#### Scenario: Size test

- **WHEN** the size test sums the component struct sizes
- **THEN** the total is at most 192 bytes

### Requirement: Budgets scale with workers without changing results

With four workers the parallel movement and needs systems SHALL show a speed-up of at least 2.0 over one worker at 5 000 guests (target; recorded), and results SHALL be identical to the one-worker run. A speed-up below 1.3 SHALL be recorded as a finding for the job-system owner.

#### Scenario: Scaling report

- **WHEN** the scaling benchmark runs at 1, 2, 4 and 8 workers
- **THEN** the speed-ups are recorded and the world hashes are equal across worker counts

### Requirement: Server cost of streaming is bounded

With 20 connected clients each viewing 300 guests, the encode cost SHALL stay under 2 ms per published frame in total (shared work is encoded once per tier cadence), and the discarded-frame counter SHALL stay at 0 on a healthy loopback connection.

#### Scenario: Many clients

- **WHEN** 20 loopback clients are connected for 60 s
- **THEN** encode time per frame is recorded, is below 2 ms at the 95th percentile, and no frame is discarded
