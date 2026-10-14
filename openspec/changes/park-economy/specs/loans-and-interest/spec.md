# Spec Delta

## Purpose

Defines borrowing and repaying park loans, the weekly interest rule and the limits that come from the difficulty profile. A loan lets the player invest early at a measurable cost.

## ADDED Requirements

### Requirement: Loans move in fixed steps within a limit

Borrowing and repaying SHALL be in multiples of 100 000 cents. The outstanding loan SHALL never exceed the difficulty's maximum loan. Borrowing SHALL add the amount to the balance and the loan; repaying SHALL require the balance to cover the amount and the amount not to exceed the loan.

#### Scenario: Borrow within the limit

- **WHEN** the loan is 1 000 000, the maximum is 2 500 000 and the player borrows 1 000 000
- **THEN** the loan is 2 000 000 and the balance rises by 1 000 000

#### Scenario: Over the limit

- **WHEN** the player borrows 2 000 000 with a loan of 1 000 000 and a maximum of 2 500 000
- **THEN** the request fails and nothing changes

#### Scenario: Repay more than the balance

- **WHEN** the balance is 300 000 and the player repays 500 000
- **THEN** the request fails and nothing changes

#### Scenario: Off-step amount

- **WHEN** the player borrows 150 000
- **THEN** the request fails with a validation error naming the step

### Requirement: Interest is charged weekly on the outstanding loan

At each week close the economy SHALL post an expense `LoanInterest` equal to `loan * annual basis points / (10 000 * 48)` rounded half away from zero (48 weeks per year: 12 months of 4 weeks). Interest SHALL NOT be added to the loan. A loan of 0 SHALL post nothing.

#### Scenario: Normal difficulty

- **WHEN** the loan is 1 000 000 cents at 1 000 basis points
- **THEN** the weekly interest is 2 083 cents (2 083.33 rounded)

#### Scenario: Rate zero

- **WHEN** the rate is 0
- **THEN** no interest line is posted

#### Scenario: Repaid loan stops interest

- **WHEN** the loan is repaid fully during a week
- **THEN** that week's close posts no interest

### Requirement: Loans are blocked in terminal states

Borrowing SHALL be rejected once the scenario is lost. Repaying SHALL remain allowed in every state when the balance covers it.

#### Scenario: Lost scenario

- **WHEN** the scenario status is Lost and the player borrows
- **THEN** the request fails with a message naming the status

### Requirement: Loan state is visible

The loan, the maximum, the rate in basis points and the next week's expected interest SHALL be available from the summary query.

#### Scenario: Expected interest

- **WHEN** the loan is 2 000 000 at 1 000 basis points
- **THEN** the expected interest shown is 4 167 cents (4 166.67 rounded)
