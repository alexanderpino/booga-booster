# Spec Delta

## Purpose

Defines the park's income and expense ledger: the categories, how revenue flows in during a week and is closed into ledger lines, the conservation rule that keeps the balance honest, and the queries the UI and the balancing harness use.

## ADDED Requirements

### Requirement: Every posting has a category with a direction

Operating categories SHALL be: income (`EntranceFees`, `RideTickets`, `ShopSales`, `ConstructionRefund`) and expense (`Construction`, `RideUpkeep`, `StaffWages`, `Research`, `Marketing`, `LoanInterest`). Financing categories SHALL be `LoanDrawdown` and `LoanRepayment`. A posting whose sign contradicts its category direction SHALL be rejected. Zero amounts SHALL be ignored.

#### Scenario: Wrong sign

- **WHEN** an expense category is posted with a positive amount
- **THEN** the posting is rejected with a validation error

#### Scenario: Financing is not profit

- **WHEN** a week has income 5 000, expenses 3 000 and a loan drawdown of 100 000
- **THEN** the week's operating profit is 2 000

### Requirement: Balance equals opening plus all postings

At all times, balance SHALL equal the opening balance plus the sum of all closed ledger lines plus the net of the open week's postings. This conservation invariant SHALL be checked by a property test over random posting sequences and by every snapshot load.

#### Scenario: Invariant after random operations

- **WHEN** 10 000 seeded random postings and week closes are applied
- **THEN** the invariant holds after every operation

#### Scenario: Detects tampering

- **WHEN** a snapshot is loaded whose balance differs from opening plus lines
- **THEN** loading is rejected with the difference in the error

### Requirement: Revenue is posted when flushed and lines are created when the week closes

Each flush of simulation revenue (every 12 s of simulated time) SHALL update the balance at once and add to the open week's running totals per category and per ride. Closing the week SHALL turn the running totals and the week's expense postings into ledger lines, ordered by category order then subject id, and start a new open week.

#### Scenario: Balance moves at flush

- **WHEN** a flush brings 12 000 cents of ride tickets
- **THEN** the balance rises by 12 000 immediately and no ledger line exists yet for the open week

#### Scenario: Close creates ordered lines

- **WHEN** the week closes with income for two rides and upkeep for the same rides
- **THEN** lines appear in category order then ride id order, identically on every run

### Requirement: Ledger queries are paged and summarised

The economy SHALL answer: lines for a week range (at most 200 per page with a continuation token), weekly summaries (income, expense, operating profit, financing, closing balance), monthly summaries (four weeks) and the current open week's running totals.

#### Scenario: Weekly summary

- **WHEN** week 7 had income 9 000 and expenses 6 500
- **THEN** its summary shows operating profit 2 500 and the closing balance of that week

#### Scenario: Paging

- **WHEN** a range holds 450 lines
- **THEN** three pages of 200, 200 and 50 lines are returned

### Requirement: History is retained for 104 weeks in detail

Detailed lines SHALL be kept for the last 104 weeks. Weeks older than that SHALL be compacted, one whole month (four weeks) at a time, into monthly lines per category (subject dropped), preserving every total so the conservation invariant still holds.

#### Scenario: Compaction

- **WHEN** week 112 closes
- **THEN** weeks 1 to 8 (two whole months older than the 104 kept) are compacted into monthly lines and the sum of all lines is unchanged; a month that is only partly older than the window stays detailed
