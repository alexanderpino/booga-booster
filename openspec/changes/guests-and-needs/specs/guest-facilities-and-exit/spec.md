# Spec Delta

## Purpose

Defines how guests satisfy needs at food stalls, drink stalls, toilets, first aid and information kiosks, how purchases are paid and recorded, and the rules by which a guest decides to leave the park.

## ADDED Requirements

### Requirement: Urgent needs send a guest to the nearest matching facility

When hungry (7 000) and holding nothing, a guest SHALL head for the nearest food facility by walking distance; when thirsty, the nearest drink facility; when it needs the toilet (7 500), the nearest toilet; when very sick (8 500), the nearest first aid. Urgent needs SHALL be checked every 240 ticks (cohort) and SHALL pre-empt wandering and ride-seeking but not a ride in progress or a queue the guest has been in for less than 60 s.

#### Scenario: Hungry guest picks nearest food

- **WHEN** a hungry wandering guest has food stalls at 40 m and 90 m walking distance
- **THEN** it sets intent to seek food with the 40 m stall as target

#### Scenario: Needs do not pre-empt riding

- **WHEN** a guest becomes needs-toilet while riding
- **THEN** it finishes the ride, then heads for the toilet

#### Scenario: No facility of that kind

- **WHEN** no toilet is reachable
- **THEN** the guest records a "no toilet" thought once, its happiness target falls by 10 per second while the need is unmet, and it keeps wandering

### Requirement: Purchases are gated by cash, value and weather

A guest SHALL buy an item only if its price is at most the guest's cash, at most `thrift ratio` times the item's value, and the item's weather class allows it (cold items refused at or below 8 degrees Celsius, open-air stalls refused in precipitation). A refused purchase SHALL record a thought and set the guest wandering again.

#### Scenario: Price above thrift limit

- **WHEN** a drink costs 400 cents, its value is 200 and the thrift ratio is 1.5
- **THEN** the guest refuses and records "too expensive"

#### Scenario: Cold item in cold weather

- **WHEN** the temperature is 5 degrees Celsius and the item is an ice cream
- **THEN** the guest refuses to buy it

### Requirement: Consumption changes needs and may create litter

Eating SHALL take 8 s, reduce hunger by 6 000 and thirst by 1 000, and add 1 000 to bladder. Drinking SHALL take 5 s, reduce thirst by 6 000 and add 1 500 to bladder. Buying SHALL deduct the price from cash and produce a purchase record for the facility. After consumption the guest SHALL hold an empty container; within 60 s while on a path it drops it with probability 50 %, producing litter.

#### Scenario: Eating

- **WHEN** a guest with hunger 7 400, thirst 3 000, bladder 2 000 and cash 1 500 buys a snack for 300 and eats it
- **THEN** after 8 s hunger is 1 400, thirst is 2 000, bladder is 3 000, cash is 1 200 and a purchase record of 300 exists for the stall

#### Scenario: Container dropped

- **WHEN** the guest's draw for the 50 % drop succeeds
- **THEN** one litter item appears at the guest's node and the guest holds nothing afterwards

### Requirement: Toilets and first aid reset their need

Using a toilet SHALL take 10 s, set bladder to 0 and raise the happiness target by 300; its price, if any, SHALL be at most 20 % of the guest's cash. First aid SHALL take 10 s and cap nausea at 1 000.

#### Scenario: Toilet use

- **WHEN** a guest with bladder 8 000 uses a free toilet
- **THEN** after 10 s bladder is 0 and the happiness target is 300 higher

#### Scenario: Toilet too expensive

- **WHEN** the toilet costs 500 cents and the guest has 1 800 cents
- **THEN** the guest refuses (500 is more than 20 % of 1 800) and records a thought

### Requirement: An information kiosk sells a park map

Buying a map SHALL set the guest's has-map flag, which widens ride search to the whole park and raises the search probability to 60 %. A guest with a map SHALL NOT buy another.

#### Scenario: Map widens the search

- **WHEN** a guest holds a map and the nearest ride by walking is 300 m away
- **THEN** that ride can be a candidate, whereas without a map it is not

### Requirement: Guests leave when the visit is over

A guest SHALL change intent to leaving, head for the park exit and never accept another ride or purchase when any of the following first holds: fatigue at least 9 000; visible happiness below 1 500 for 30 consecutive updates; cash below 300 cents while wandering (checked with probability 10 % per update); no successful ride or purchase for 240 s (the "lost" rule, which also lowers the happiness target by 3 000). The first matching rule SHALL be recorded as the leave reason.

#### Scenario: Exhausted

- **WHEN** a wandering guest's fatigue reaches 9 000
- **THEN** its intent becomes leaving with reason `Tired`

#### Scenario: Lost

- **WHEN** a guest has been wandering for 240 s without finding an acceptable ride or purchase
- **THEN** it leaves with reason `Lost`, its happiness target drops by 3 000, and the lost-guest count of the park metrics increases

#### Scenario: Unhappiness needs persistence

- **WHEN** happiness is below 1 500 for 29 consecutive updates and then rises
- **THEN** the guest does not leave for that reason

### Requirement: Leaving guests do not queue or consume

A guest whose intent is leaving SHALL leave any queue it is in, release its ticket and refuse all rides and purchases.

#### Scenario: Leaving from a queue

- **WHEN** a queuing guest meets a leave rule
- **THEN** the queue length of its ride decreases by one in the next ranking pass and the guest heads for the exit
