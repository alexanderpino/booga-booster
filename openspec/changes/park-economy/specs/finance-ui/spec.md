# Spec Delta

## Purpose

Defines the finance user interface built with PrimeNG: balance and loan, ledger table and chart, live cashflow, pricing, staff, research, marketing, rating breakdown, awards and scenario progress. It must be usable by keyboard and screen reader and never rely on colour or a canvas alone.

## ADDED Requirements

### Requirement: A finance overview shows the state at a glance

The overview SHALL show balance, loan, interest due next week, operating profit of the last closed week, the scenario status with its deadline and goal progress, the rating with band, and the calendar position (year, month, week). Values SHALL be formatted from integer cents with two decimals and the configured currency label; negative values SHALL show a minus sign and the word "loss" or "overdrawn" in text.

#### Scenario: Negative balance

- **WHEN** the balance is -12 345 cents
- **THEN** the overview shows "-123.45" with the text "overdrawn"

#### Scenario: Goal progress

- **WHEN** a guests-and-rating goal has a hold counter of 2 of 4
- **THEN** a labelled progress bar shows 2 of 4 weeks with its value in text

### Requirement: The ledger is a table with a chart and a table alternative

The ledger view SHALL show weekly income, expenses and operating profit for the last 24 weeks as a table (PrimeNG table, sortable, with category columns) and as a line chart of income against expenses. The chart SHALL have an accessible name, a text summary of trend and range, and the table SHALL be the equivalent data alternative reachable next to it. Series SHALL differ by line style as well as colour.

#### Scenario: Table matches chart

- **WHEN** 24 weeks of data are shown
- **THEN** the table has 24 rows and the chart has 24 points per series with the same values

#### Scenario: Not colour alone

- **WHEN** the chart is rendered in forced-colours or greyscale
- **THEN** the series remain distinguishable by line style and a text legend

### Requirement: A live cashflow view shows the last 12 minutes

A live view SHALL show income and expense per 12-second flush for the last 60 flushes (12 simulated minutes), refreshed by polling every 2 seconds, so a price change is visible within one flush.

#### Scenario: Price change shows up

- **WHEN** the entrance fee is raised
- **THEN** within two flush periods the live view shows the changed income per period

#### Scenario: Polling stops

- **WHEN** the view is closed
- **THEN** polling stops and in-flight requests are cancelled

### Requirement: Pricing, loans, staff, research and marketing are operable forms

Each area SHALL offer a form with labelled inputs honouring the allowed ranges and steps (fee and ride prices with min, max, step and the price advice text; loan borrow and repay in steps; staff counts within caps; research funding and focus; campaign type and weeks), inline validation messages tied to inputs, and a confirmation dialog for borrowing, repaying and campaigns above 100 000 cents. Failed server validation SHALL be shown next to the input and announced politely.

#### Scenario: Out of range price

- **WHEN** the user types 60.00 in a ride price field with a maximum of 50.00
- **THEN** the field shows an error "Maximum is 50.00" associated with the input and the value is not sent

#### Scenario: Server rejection

- **WHEN** the server rejects a hire because of the cap
- **THEN** the message appears next to the staff input and is announced in a polite live region

#### Scenario: Confirmation focus

- **WHEN** the borrow confirmation dialog opens
- **THEN** focus moves into it, is trapped, and returns to the borrow button on close

### Requirement: The rating view teaches the formula

The rating view SHALL show the current rating, the target, the band and the nine terms with their inputs and values as a table, and SHALL highlight the largest positive and negative terms in text.

#### Scenario: Breakdown rows

- **WHEN** the breakdown has nine terms
- **THEN** nine rows are shown with input and value, and the text names the largest penalty

### Requirement: Research, marketing and awards views show progress and expiry

The research view SHALL show the current item with a progress bar, the queue, funding and focus; the marketing view SHALL list campaigns with remaining weeks; the awards view SHALL list held awards with polarity text and remaining weeks.

#### Scenario: Research progress

- **WHEN** the current item has 60 of 100 points
- **THEN** a progress bar with the value 60 percent and text "60 of 100 points" is shown

### Requirement: The finance UI meets accessibility and performance rules

All finance views SHALL pass axe-core with no violations at the WCAG AA level the repository requires, be keyboard operable with visible focus, use native control flow, signals, OnPush and standalone components, announce transactions through a polite live region and honour reduced motion. The chart library choice SHALL pass the currency check; if it does not, an accessible SVG chart without dependency SHALL be used.

#### Scenario: Axe

- **WHEN** the a11y spec renders every finance view
- **THEN** axe-core reports zero violations

#### Scenario: Chart library fallback

- **WHEN** the currency check of the chart package fails
- **THEN** the SVG fallback component renders the same series and passes the same specs
