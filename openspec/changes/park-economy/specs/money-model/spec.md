# Spec Delta

## Purpose

Defines how money is represented and computed in the economy: exact integer cents with checked arithmetic, one rounding rule, safe parsing and formatting, and a wire format that cannot lose precision. Every other economy capability relies on it.

## ADDED Requirements

### Requirement: Money is an exact integer count of cents

The economy SHALL represent every amount as a signed 64-bit count of cents (1/100 of the park currency unit). Binary floating point (`float`, `double`) SHALL NOT appear in economy domain types, DTOs or JSON for amounts. Amounts SHALL be limited to the range -10^15 to +10^15 cents inclusive.

#### Scenario: Exact representation

- **WHEN** 0.10 is added ten times as ten amounts of 10 cents
- **THEN** the total is exactly 100 cents

#### Scenario: Limit enforced

- **WHEN** a value above 10^15 cents is constructed
- **THEN** construction fails with a validation error naming the limit

#### Scenario: No floating point in the domain

- **WHEN** the architecture test scans the Economy domain and abstractions assemblies for `double` and `float` members and parameters
- **THEN** it finds none (conversion helpers at the API edge are excluded and listed in the test)

### Requirement: Arithmetic is checked

Addition, subtraction and negation SHALL fail with a money overflow error instead of wrapping when the result leaves the allowed range. Comparison, sign and absolute value SHALL be total on the allowed range.

#### Scenario: Overflow detected

- **WHEN** 9.9 x 10^14 cents are added to 9.9 x 10^14 cents
- **THEN** the operation fails with a money overflow error and no value is produced

#### Scenario: Subtraction below zero is allowed

- **WHEN** 500 cents are subtracted from 200 cents
- **THEN** the result is -300 cents

### Requirement: Proportional amounts use one rounding rule

Multiplying an amount by a ratio expressed in permille or basis points SHALL round half away from zero to the nearest cent, using integer arithmetic only. Splitting an amount into n parts SHALL give every part the floor share and distribute the remainder one cent at a time to the earliest parts, so the parts sum to the original.

#### Scenario: Round half away from zero

- **WHEN** 1 001 cents are multiplied by 505 permille (505.505 cents) and -1 001 cents likewise
- **THEN** the results are 506 and -506

#### Scenario: Exact half

- **WHEN** 1 cent is multiplied by 500 permille
- **THEN** the result is 1 cent; -1 cent gives -1 cent

#### Scenario: Split preserves the sum

- **WHEN** 1 000 cents are split into 3 parts
- **THEN** the parts are 334, 333 and 333

### Requirement: Text conversion is exact and culture independent

Parsing SHALL accept an optional sign, digits and at most two decimal digits separated by a period (for example `12`, `12.5`, `-0.05`) and SHALL reject anything else, including exponents, thousands separators and more than two decimals. Formatting SHALL always emit a period decimal separator and exactly two decimals, regardless of culture.

#### Scenario: Valid inputs

- **WHEN** `12.5`, `-0.05` and `7` are parsed
- **THEN** the amounts are 1 250, -5 and 700 cents

#### Scenario: Invalid inputs

- **WHEN** `1e3`, `1,000.00`, `1.234` and the empty string are parsed
- **THEN** each is rejected with a validation error

#### Scenario: Culture independence

- **WHEN** 123 456 cents are formatted under a culture that uses a comma decimal separator
- **THEN** the text is `1234.56`

### Requirement: The wire format carries cents as integers

Every DTO and integration event that carries an amount SHALL do so as a JSON integer count of cents in a property whose name ends in `Cents`. A value outside the 53-bit safe integer range of JSON consumers SHALL NOT occur because of the limit above; clients SHALL format for display only.

#### Scenario: Serialisation

- **WHEN** a balance of 1 234 567 cents is serialised
- **THEN** the JSON contains `"balanceCents":1234567`
