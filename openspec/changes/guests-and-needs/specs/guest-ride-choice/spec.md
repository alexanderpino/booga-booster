# Spec Delta

## Purpose

Defines how a wandering guest decides which ride to walk to: a set of hard gates a ride must pass for that guest, and a score that picks the best ride among those that pass. The rules use ride ratings in hundredths as supplied by the ride catalog.

## ADDED Requirements

### Requirement: Guests decide in cohorts with a probabilistic trigger

A wandering guest SHALL make a ride decision once every 480 ticks (cohort `(tick + entity index) mod 480 == 0`). At each decision it SHALL look for a ride with probability 25 %, or 60 % if it holds a park map, drawn from its counter-based stream. A guest holding food or a drink, a guest leaving the park, and a guest in a party whose leader is not deciding SHALL NOT look for a ride.

#### Scenario: Trigger rate

- **WHEN** 10 000 wandering guests reach their decision tick
- **THEN** between 2 350 and 2 650 of them look for a ride (seeded, repeatable)

#### Scenario: Holding food

- **WHEN** a guest holds an unfinished snack at its decision tick
- **THEN** it does not look for a ride

### Requirement: A ride must pass every gate to be a candidate

A ride SHALL be a candidate for a guest only if it passes every gate: status, queue, reach, previous ride, price against cash, price against value (below), weather, intensity, nausea and expected wait (each defined in this capability). A failed gate SHALL record its refusing reason for the thought system.

#### Scenario: Status, queue and reach gates

- **WHEN** a ride is closed, broken, has a full queue, or is 70 m away by path (60 m limit) and the guest holds no map
- **THEN** the ride is not a candidate; with a map the 70 m ride can be a candidate

#### Scenario: Too expensive

- **WHEN** a ride costs 600 cents against a value of 300 and the guest's thrift ratio (a trait between 1.2 and 2.0) is 1.5
- **THEN** the ride is refused for price and the guest records a "too expensive" thought naming the ride

#### Scenario: Cannot afford

- **WHEN** a ride costs 500 cents and the guest has 450
- **THEN** the ride is refused and the guest records a "cannot afford" thought

#### Scenario: Previous ride excluded

- **WHEN** the guest has just left ride A
- **THEN** ride A is not a candidate at its next decision, and is a candidate again after any other ride, shop or toilet visit

### Requirement: The intensity gate uses a window widened by happiness

The guest's intensity window is [min, max] in hundredths. The slack SHALL be `3 * happiness percent` hundredths (0 to 300). A ride SHALL pass if its intensity is in [min - slack, max + slack]; below SHALL record "not thrilling enough", above SHALL record "too intense".

#### Scenario: Window and slack

- **WHEN** a guest's window is [200, 600] and happiness is 50 percent (slack 150, accepted range 50 to 750)
- **THEN** intensity 460 passes, 760 is refused as too intense and 49 is refused as not thrilling; at happiness 60 percent (slack 180, range 20 to 780) intensity 760 passes

#### Scenario: Boundary

- **WHEN** intensity equals max + slack exactly
- **THEN** the ride passes

### Requirement: The nausea gate uses the tolerance class limit

A ride SHALL pass the nausea gate if its nausea rating is at most `limit + slack` with limits 250, 450, 700 and 1100 for none, low, average and high. A guest whose current nausea is at least the sick threshold SHALL additionally refuse every ride with nausea rating 200 or more.

#### Scenario: Class limits

- **WHEN** a ride has nausea rating 600 and slack is 0
- **THEN** guests of class none and low refuse it and guests of class average and high accept it

#### Scenario: Sick guest

- **WHEN** a high-tolerance guest has nausea 7 000 and the ride's nausea rating is 250
- **THEN** the ride is refused

### Requirement: Weather gate refuses unsheltered rides in precipitation

While precipitation is anything other than none, a ride that is not sheltered SHALL be refused with "not in the rain" and the guest's happiness target SHALL drop by 300 once per decision at which this happens.

#### Scenario: Rain

- **WHEN** it is raining and ride R is not sheltered while ride S is
- **THEN** R is refused and S may be chosen

### Requirement: The queue tolerance gate uses the expected wait

The expected wait of a ride SHALL be queue length times seconds per rider, where seconds per rider is the ride's cycle seconds divided by its riders per cycle, both from the ride contract. A ride SHALL be refused when the expected wait exceeds the guest's queue tolerance.

#### Scenario: Long queue

- **WHEN** a ride carries 32 riders per 90 s cycle and 150 guests wait, and the guest's tolerance is 300 s
- **THEN** expected wait is 150 * 90 / 32 = 421.9 s and the ride is refused

### Requirement: The best-scoring candidate wins deterministically

Among candidates, the guest SHALL choose the ride with the highest integer score `2 * excitement + novelty - 4 * walking metres - 2 * expected wait seconds`, where novelty is 150 if the guest has never ridden it and 0 otherwise. Equal scores SHALL be won by the lower ride ordinal. If no candidate exists the guest continues wandering and the idle clock keeps running.

#### Scenario: Excitement versus distance

- **WHEN** ride A has excitement 700, 50 m away, wait 20 s, ridden before; ride B has excitement 500, 10 m away, wait 10 s, new
- **THEN** A scores 1 400 - 200 - 40 = 1 160 and B scores 1 000 + 150 - 40 - 20 = 1 090 and the guest chooses A

#### Scenario: Tie

- **WHEN** two candidates score identically
- **THEN** the one with the lower ride ordinal is chosen

#### Scenario: No candidate

- **WHEN** every ride fails a gate
- **THEN** the guest stays wandering and no target is set

### Requirement: Choosing a ride starts a walk and parties follow the leader

On choosing a ride the guest SHALL set its intent to seeking that ride, record the target and start following that ride's field. Followers in the same party SHALL adopt the same target in the same tick; a follower for whom a gate would fail (for example cannot afford) SHALL leave the party.

#### Scenario: Party chooses together

- **WHEN** the leader of a party of three chooses ride R and member 2 cannot afford it
- **THEN** members 0 and 1 seek R and member 2 becomes a party of one that keeps wandering
