# Spec Delta

## Purpose

Defines what a simulated park guest is: the data every guest carries, how its traits are drawn from a seeded sampler, how parties group guests, and when guests are created and removed. Everything else about guests builds on these facts.

## ADDED Requirements

### Requirement: A guest is an entity with a fixed set of runtime data

Every guest SHALL be one entity in the simulation world carrying: position, path motion, needs and mood, wallet, preferences, intent with its current target, queue ticket, party membership, ride history, held item and a thought history. Hot per-tick toggles (intent, queuing) SHALL be fields of these data, not added or removed data, so a guest never changes storage layout while it lives. A guest SHALL NOT store text.

#### Scenario: A new guest has every datum

- **WHEN** a guest is spawned
- **THEN** the world reports all listed data on that entity and no guest is ever present with a subset of them

#### Scenario: Changing intent does not move the entity between storage groups

- **WHEN** a guest changes intent from wandering to queuing to riding
- **THEN** the number of distinct storage groups in the world is unchanged and the guest's entity id is unchanged

#### Scenario: Names are derived, not stored

- **WHEN** the display name of guest number 1042 is requested twice, including after a world save and restore
- **THEN** both requests return the identical name and the saved world holds no text for that guest

### Requirement: Guest numbers are unique and ever-increasing

Each guest SHALL receive a guest number from a single counter that only increases, starts at 1 and is part of the saved world state. The number SHALL identify the guest in logs, names and sampling and SHALL never be reused, even after the guest leaves.

#### Scenario: Numbers increase across despawns

- **WHEN** guests numbered 1, 2 and 3 are spawned, guest 2 leaves, and a new guest is spawned
- **THEN** the new guest has number 4

#### Scenario: The counter survives save and restore

- **WHEN** a world with 500 spawned guests is saved and restored and one more guest is spawned
- **THEN** the new guest has number 501

### Requirement: Guest traits come from a seeded sampler

All random traits of a guest (starting needs, cash, intensity window, nausea tolerance class, queue tolerance, thrift ratio, walking speed, fatigue scale, party role) SHALL be produced by a sampler that is a pure function of the world seed and the guest number. The same seed and number SHALL always yield the same traits, independent of call order or thread.

#### Scenario: Same seed and number give the same traits

- **WHEN** the sampler is asked twice for seed 42, guest number 7, in different call orders
- **THEN** both results are field-for-field identical

#### Scenario: Traits respect their documented ranges

- **WHEN** 100 000 guests are sampled with seed 42
- **THEN** every starting hunger and thirst lies in [1000, 3500], every starting bladder in [0, 2000], every starting fatigue in [500, 1500], every cash amount in [2500, 6500] cents, every walking speed in [1.10, 1.50] m/s, every queue tolerance in [120, 420] s, and no guest has an intensity window minimum above its maximum

#### Scenario: Class proportions follow the configured weights

- **WHEN** 100 000 guests are sampled with seed 42 with default weights (nausea tolerance none 10 %, low 20 %, average 35 %, high 35 %; thrill class gentle 30 %, moderate 40 %, thrill 30 %)
- **THEN** each observed proportion is within 1.5 percentage points of its weight

#### Scenario: The sampler is replaceable in tests

- **WHEN** a test supplies a fake sampler that returns fixed traits
- **THEN** every spawned guest carries exactly those traits

### Requirement: Guests arrive in parties and parties stay together while all members are present

A party SHALL be 1 to 5 guests created in the same tick under one party id with member indices 0 to size minus 1. Members SHALL share a walking speed equal to the slowest member and SHALL choose destinations together (the lowest member index decides). A member whose own need forces a different destination SHALL leave the party and continue alone; the party id of the remaining members is unchanged.

#### Scenario: A party of three is created together

- **WHEN** the spawner creates a party of three
- **THEN** three guests exist with consecutive guest numbers, the same party id, member indices 0, 1, 2 and the same walking speed

#### Scenario: A member detaches for a toilet visit

- **WHEN** member 1 of a party of three becomes urgent for a toilet while the others are wandering
- **THEN** member 1 becomes a party of one and members 0 and 2 keep their party id and continue together

### Requirement: The number of guests is capped

The world SHALL hold at most `MaxGuests` guests (default 6 000). A spawn that would exceed the cap SHALL be skipped and counted in a metric; it SHALL NOT fail the tick.

#### Scenario: Cap reached

- **WHEN** 6 000 guests are in the park and the arrival process requests a party of two
- **THEN** no guest is created, the skipped-spawn counter increases by two and the tick completes normally

### Requirement: Guests are removed only at the park exit or by a documented fault

A guest SHALL be removed from the world only when it reaches the park exit while leaving, or when its position becomes unreachable after a graph change and it has been stranded for more than 60 s. Removal SHALL be done through the deferred command mechanism, release any ride-queue ticket the guest holds and record a leave reason.

#### Scenario: Guest exits the park

- **WHEN** a leaving guest reaches the exit node
- **THEN** its entity is destroyed after the tick, the queue ticket (if any) is released and a leave event with the reason is recorded

#### Scenario: Stranded guest is removed after the grace period

- **WHEN** a path edit leaves a guest with no route to any node for 60 s
- **THEN** the guest is removed with reason `Stranded` and the stranded counter increases by one
