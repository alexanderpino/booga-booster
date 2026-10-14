# Spec Delta

## Purpose

Defines how guests join a ride's queue, keep order and party contiguity, wait and abandon, are handed to the ride for boarding, pay, and receive the ride outcome. It also keeps the existing `Queue` module contract working so the ride loading logic is unchanged.

## ADDED Requirements

### Requirement: Joining a queue assigns a ticket in arrival order

When a guest bound for ride R reaches the start of R's queue path and the queue is not full, its intent SHALL become queuing and it SHALL receive a ticket from R's counter, which only increases. Members of a party joining in the same tick SHALL receive consecutive tickets ordered by member index. A full queue (all queue path slots occupied, or the ride's maximum length reached) SHALL turn the guest away with a "queue too long" thought.

#### Scenario: Order of arrival

- **WHEN** guests A, B and C reach the queue start in ticks 100, 101 and 103
- **THEN** their tickets are increasing in that order and A is ahead of B ahead of C

#### Scenario: Party stays contiguous

- **WHEN** a party of three joins in one tick between two single guests
- **THEN** the party members hold three consecutive tickets and no other guest's ticket lies between them

#### Scenario: Queue full

- **WHEN** a queue path has 40 slots and 40 guests wait
- **THEN** a further guest is turned away, its target is cleared and a thought is recorded

### Requirement: Queue rank and length are derived every 30 ticks

Every 30 ticks, for each ride, the queuing guests SHALL be ranked by ascending ticket; rank 0 is the front. Each guest SHALL walk to the queue slot node of its rank (the last slot for ranks beyond the path length). The ride's queue length and the longest wait SHALL be written to the ride's queue statistics. Ranking SHALL not depend on storage order or thread count.

#### Scenario: Shuffle forward

- **WHEN** the front guest boards
- **THEN** within one ranking pass every other guest's rank decreases by one and the length decreases by one

#### Scenario: Statistics

- **WHEN** 12 guests wait, the longest for 95 s
- **THEN** the ride reports length 12 and longest wait 95 s after the next pass

### Requirement: Waiting is bounded by tolerance and abandonment

A guest whose wait exceeds twice its queue tolerance and whose happiness is below 3 500 SHALL leave the queue with probability 30 permille per update. An abandoning guest SHALL release its ticket, record "waited for ages" and return to wandering. A ride that closes or breaks SHALL release every ticket within one ranking pass.

#### Scenario: Ride breaks down

- **WHEN** a ride with 20 queuing guests goes out of service
- **THEN** after the next ranking pass all 20 are wandering with a "ride closed" thought

#### Scenario: Abandonment threshold

- **WHEN** tolerance is 200 s, the wait is 399 s and happiness is 2 000
- **THEN** the guest does not abandon; at 401 s it may abandon

### Requirement: Boarding is a handshake through the ride's boarding port

A ride that is ready to load SHALL publish an offer of N seats on its boarding port. The boarding step SHALL take, in rank order, every party that fully fits the remaining seats (skipping parties too large), mark each taken guest as riding with the ride cycle, and reduce the offer. Seat units and split rules belong to the ride.

#### Scenario: First fitting party

- **WHEN** 6 seats are offered and the queue is a party of 8, then a party of 3, then a party of 2
- **THEN** the party of 3 and the party of 2 board (5 seats), the party of 8 stays and the offer drops to 1 seat

#### Scenario: Nothing fits

- **WHEN** 2 seats are offered and the first party in the look-ahead window of 10 parties has 3 members
- **THEN** nobody boards and the offer remains 2

#### Scenario: Offer is single use per tick

- **WHEN** the boarding step has consumed an offer
- **THEN** the same offer is not consumed again in the next tick

### Requirement: Payment happens on boarding

A boarding guest SHALL pay the ride's current price once, on boarding. If the price has risen above the guest's cash since it joined, the guest SHALL not board, SHALL release its ticket, and SHALL record "cannot afford". Income SHALL be added to the ride's traffic statistics and to the period purchase records.

#### Scenario: Price raised while waiting

- **WHEN** a guest with 450 cents is at the front when the price changes from 400 to 500
- **THEN** it does not board, keeps 450 cents and records a thought

#### Scenario: Income recorded

- **WHEN** 20 guests board a ride priced 300
- **THEN** the ride's income total rises by 6 000 and its rider count by 20

### Requirement: Ride outcome sets satisfaction when the ride cycle completes

When the ride completes the cycle a guest boarded, the guest SHALL walk to the ride exit node and its happiness target SHALL change by the sum of five terms (values in the first scenario), clamped to [-4 000, +5 000]. Visible happiness SHALL then snap to the target, nausea SHALL be added by the tolerance rule and the ride SHALL enter the guest's history.

#### Scenario: Term values

- **WHEN** the five terms are evaluated
- **THEN** intensity fit is +2 200 inside the window, +1 000 inside window plus slack, -1 500 outside
- **AND** nausea fit is +800 at most half the class limit, 0 at most the limit, -1 800 above it
- **AND** value is +900 when price is at most half the value, 0 at most the value, -1 000 above it
- **AND** novelty is +500 on a first ride and 0 otherwise
- **AND** wait is +600 up to 90 s, -1 500 beyond the guest's tolerance and 0 in between

#### Scenario: A great ride

- **WHEN** a first-time guest rides a good-value ride, intensity inside its window, low nausea, waited 60 s
- **THEN** the satisfaction is 2 200 + 800 + 900 + 500 + 600 = 5 000 (the clamp) and visible happiness equals the new target

#### Scenario: A bad ride

- **WHEN** intensity is outside the window and slack, nausea above the limit, price above value, repeated ride, waited past tolerance
- **THEN** the sum is -1 500 - 1 800 - 1 000 + 0 - 1 500 = -5 800, clamped to -4 000

### Requirement: The Queue module contract is served in park mode

In park mode `IRideQueueService` SHALL be implemented over the simulation's queue state with the same observable contract as the Queue module's implementation: status lists groups in rank order, taking a group removes and returns it (null if gone), and enqueuing creates a party at the queue start subject to caps. A taken group SHALL board through the same path as the generic handshake. The existing Queue service contract tests SHALL pass against it.

#### Scenario: Status mirrors the queue

- **WHEN** three parties wait in ranks 0 to 2
- **THEN** `GetStatus` lists three groups in that order with matching sizes and total weights

#### Scenario: Take returns once

- **WHEN** `TakeGroupAsync` is called twice for the same group
- **THEN** the first returns the group and the second returns null

#### Scenario: Weight comes from the guest

- **WHEN** a guest of weight 83 kg is in a taken group
- **THEN** the returned person has weight 83

### Requirement: Group-queued events are published per joining party

When a party joins a ride queue the simulation SHALL emit one group-queued integration event (ride id, group id, people count, time) through the integration event publisher, outside the tick, without blocking it.

#### Scenario: Event after the tick

- **WHEN** a party of two joins during tick 500
- **THEN** exactly one event with people count 2 is published after tick 500 completes
