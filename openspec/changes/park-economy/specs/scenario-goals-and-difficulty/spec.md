# Spec Delta

## Purpose

Defines scenarios (goals with deadlines), the win and lose rules including bankruptcy, and the difficulty profiles that set starting money, loan terms, cost factors and tolerance. They give a session a purpose and let a workshop pick its pace.

## ADDED Requirements

### Requirement: Difficulty profiles are data

A difficulty profile SHALL define: starting balance, starting loan, maximum loan, annual interest in basis points, upkeep factor, wage factor and research cost factor (permille), rating offset, bankruptcy tolerance in weeks (0 disables bankruptcy) and whether unlock-by-cash is enabled. Four profiles ship (values in the first scenario, amounts in cents).

#### Scenario: Shipped profiles

- **WHEN** the profiles are read
- **THEN** Easy is start 2 500 000, loan 0, maximum loan 3 000 000, interest 800, factors 800 / 900 / 800, offset +50, tolerance 8 weeks, cash unlock on
- **AND** Normal is start 1 500 000, loan 1 000 000, maximum 2 500 000, interest 1 000, factors 1 000 / 1 000 / 1 000, offset 0, tolerance 4, cash unlock off
- **AND** Hard is start 800 000, loan 2 000 000, maximum 2 000 000, interest 1 400, factors 1 250 / 1 200 / 1 250, offset -50, tolerance 2, cash unlock off
- **AND** Sandbox is start 100 000 000, loan 0, maximum 0, interest 0, factors 1 000 / 1 000 / 1 000, offset 0, tolerance 0, cash unlock on

#### Scenario: Normal profile

- **WHEN** a scenario starts on Normal
- **THEN** the opening balance is 1 500 000, the loan 1 000 000 and the interest rate 1 000 basis points

#### Scenario: Invalid profile data

- **WHEN** a profile has a starting loan above its maximum loan
- **THEN** start-up fails naming the profile

### Requirement: Starting a scenario resets the economy

Starting a scenario SHALL replace all economy state: balance and loan from the profile, empty ledger, default price plan from the catalog, no staff, research reset to the scenario's initial unlocked set, no campaigns, no awards, rating at the scenario's initial value (default 500), calendar at week 1. It SHALL be the only way to change the profile, SHALL reset the simulation's economy inputs through logged commands, and SHALL publish a scenario status event.

#### Scenario: Reset

- **WHEN** a scenario is started while a previous one has a ledger of 80 weeks
- **THEN** the ledger is empty, the week is 1 and the status is Running

### Requirement: Goals are defined in data

A goal SHALL be one of `guests-and-rating` (minimum rating, minimum guests, hold weeks), `monthly-profit` (minimum operating profit per month, consecutive months) or `debt-free` (loan 0 and minimum balance). A scenario SHALL define its goals (all must be met), a deadline in months and a difficulty. The numeric thresholds are starting hypotheses calibrated by the balancing harness.

#### Scenario: Shipped scenarios

- **WHEN** the scenario list is read
- **THEN** `first-steps` is Easy with guests-and-rating (600, 300, hold 4) and a 24-month deadline
- **AND** `thrill-valley` is Normal with guests-and-rating (800, 800, hold 4) and debt-free with balance 5 000 000 and a 36-month deadline
- **AND** `cash-crunch` is Hard with monthly-profit 500 000 for 3 consecutive months and a 24-month deadline

### Requirement: Goal progress is evaluated at the week close

At every week close the economy SHALL update goal progress: for `guests-and-rating` a counter of consecutive weeks in which both the rating and the guest count meet the minimum (reset to 0 otherwise); for `monthly-profit` the count of consecutive finished months at or above the minimum, evaluated at month close; for `debt-free` whether the condition holds now. A goal is met when its condition is satisfied for its hold or consecutive length.

#### Scenario: Hold counter

- **WHEN** rating 650 and 320 guests are seen at three week closes, then rating 590 at the fourth
- **THEN** the counter is 3 then resets to 0

#### Scenario: Profit streak

- **WHEN** three consecutive months have operating profit 520 000, 510 000 and 505 000
- **THEN** the monthly-profit goal with minimum 500 000 and length 3 is met at the third month close

### Requirement: The scenario is won when all goals are met before the deadline

When every goal is met at a close, the status SHALL become Won. Won is terminal for goals: the park continues in free play, goals are no longer evaluated and the win SHALL be reported once.

#### Scenario: Win

- **WHEN** the last unmet goal becomes met at week 40 of a 24-month scenario (96 weeks)
- **THEN** the status becomes Won and one scenario status event is published

#### Scenario: Free play after winning

- **WHEN** the scenario is Won and the player changes prices
- **THEN** the change succeeds and the status stays Won

### Requirement: The scenario is lost on deadline or bankruptcy

The status SHALL become Lost with reason `DeadlinePassed` when the deadline month ends with a goal unmet, or `Bankrupt` when the balance has been negative at every week close for the profile's bankruptcy tolerance in a row. A tolerance of 0 SHALL disable bankruptcy. Lost is terminal: spending commands (construction, marketing, research funding increases, hiring, borrowing) SHALL be rejected; selling, repaying, saving and starting a new scenario SHALL remain possible.

#### Scenario: Bankruptcy on Hard

- **WHEN** the balance is negative at two consecutive week closes on Hard (tolerance 2)
- **THEN** the status becomes Lost with reason Bankrupt at the second close

#### Scenario: Recovery resets the streak

- **WHEN** the balance is negative at one close and non-negative at the next
- **THEN** the negative streak is 0

#### Scenario: Deadline

- **WHEN** the 24th month closes with the guests-and-rating goal unmet
- **THEN** the status is Lost with reason DeadlinePassed

#### Scenario: Sandbox

- **WHEN** the profile is Sandbox and the balance goes negative for 100 weeks
- **THEN** the scenario never becomes Lost

### Requirement: Negative balance blocks spending

While the balance is negative, spending commands (construction, marketing start, unlock-by-cash, loan repayment) SHALL fail with a message stating the balance; borrowing within the limit, price changes, firing staff and research funding to None SHALL stay possible.

#### Scenario: Blocked purchase

- **WHEN** the balance is -100 and the player starts a campaign
- **THEN** the request fails with the balance in the message

#### Scenario: Borrowing helps

- **WHEN** the balance is -100 and the player borrows 1 000 000 within the limit
- **THEN** the borrow succeeds
