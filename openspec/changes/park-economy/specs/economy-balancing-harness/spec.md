# Spec Delta

## Purpose

Defines the headless balancing harness: seeded multi-month runs of the whole park with scripted player policies, the metrics collected, and the acceptance bands that detect runaway inflation, money creation, unbounded debt and dead feedback loops. It is how guest and economy numbers are tuned and kept honest.

## ADDED Requirements

### Requirement: The harness runs a park headlessly and deterministically

The harness SHALL build a park from a named fixture (grid, rides, facilities, prices, scenario) and seeded guests, run the simulation and economy for N in-game months as fast as possible with the manual tick driver, and produce a result per seed. The same fixture, policy, seed and months SHALL always produce identical results.

#### Scenario: Reproducible

- **WHEN** the harness runs fixture `starter-park`, policy `reference`, seed 7, 3 months, twice
- **THEN** the result files are byte-identical

#### Scenario: Months to ticks

- **WHEN** 3 months are requested
- **THEN** the run lasts 3 x 4 x 11 520 = 138 240 ticks

### Requirement: Scripted policies stand in for the player

Policies SHALL be deterministic scripts (four ship: `reference`, `greedy`, `free`, `negligent`, defined in the first scenario). A policy SHALL act only through the same commands the API offers.

#### Scenario: Policy definitions

- **WHEN** the policies are read
- **THEN** `reference` sets entrance fee 1 000, prices each ride at half its catalog value rounded down to 10 cents, employs two handymen and one mechanic, funds research at Normal, borrows nothing extra and repays 1 000 000 whenever the balance exceeds 3 000 000
- **AND** `greedy` sets fee 5 000 and ride prices 2 000
- **AND** `free` sets fee 0 and ride prices 0
- **AND** `negligent` employs no staff, funds no research, borrows the maximum loan at once, prices as `reference` and never repays

#### Scenario: Policies use public commands

- **WHEN** a policy runs
- **THEN** every action appears in the economy command log

### Requirement: Each run records a fixed set of metrics

A result SHALL record, per week: balance, loan, operating income, operating expense, rating, guests in park, arrivals, turned away, lost, average queue seconds, old litter; and overall: bankrupt flag and week, minimum and maximum balance, final scenario status, cumulative income and expense, and the economy state hash.

#### Scenario: Columns

- **WHEN** a result is written for a 2-month run
- **THEN** it has 8 weekly rows with all listed columns

### Requirement: Structural bands hold for every build

For fixture `starter-park` the harness SHALL check seven bands that do not depend on calibrated numbers (listed in the first scenario) and SHALL report each as pass or fail with the offending seeds.

#### Scenario: Band definitions

- **WHEN** the structural bands are read
- **THEN** B1 conservation: final balance equals opening plus all income minus all expenses plus drawdowns minus repayments, exactly, in every run
- **AND** B2 valid values: rating in [0, 999] at every period and every amount inside the money limits
- **AND** B3 pricing responds: with `free` cumulative income is lower than with `reference` in at least 95 percent of seeds, and with `greedy` the mean guests in park over months 4 to 12 is lower than with `reference` in at least 90 percent of seeds
- **AND** B4 neglect is punished: `negligent` ends bankrupt or with rating below 300 in at least 90 percent of seeds over 24 months
- **AND** B5 `reference` survives: not bankrupt over 24 months in at least 90 percent of 20 seeds
- **AND** B6 stability: the coefficient of variation of the month-24 balance across seeds for `reference` is at most 0.35
- **AND** B7 no runaway: for `reference`, mean weekly income over the last 8 weeks is at most 2.0 times the mean over weeks 27 to 34 (after a 26-week warm-up)

#### Scenario: Inflation detected

- **WHEN** a defect doubles every guest payment each month
- **THEN** B7 fails and the report names the ratio and the seeds

#### Scenario: Money creation detected

- **WHEN** a defect posts income without a payment
- **THEN** B1 fails with the difference

#### Scenario: Dead feedback loop detected

- **WHEN** guest arrivals ignore the entrance fee
- **THEN** B3 fails for `greedy`

### Requirement: Calibrated bands are frozen in a committed file

A committed file `balance-bands.json` SHALL hold numeric bands for the fixture and policy `reference`: minimum and maximum of balance at months 6, 12 and 24, of guests in park at months 12 and 24 and of rating at months 12 and 24, derived by running 20 seeds and widening the observed [min, max] by 15 percent. A run SHALL fail when any value leaves its band. Changing parameters that affect results SHALL require re-running the calibration and committing the new file with the reason.

#### Scenario: Out of band

- **WHEN** the month-12 balance of one seed is below the band minimum
- **THEN** the harness exits non-zero and prints seed, metric, value and band

#### Scenario: Recalibration

- **WHEN** a guest parameter changes
- **THEN** the calibration command regenerates the file and the diff shows the new bands

### Requirement: Smoke and nightly profiles differ only in size

The PR smoke profile SHALL run 5 seeds for 3 months of the `reference` policy, check B1 and B2 and the pinned numbers of a golden result (exact), and take at most 5 minutes on the CI reference machine. The nightly profile SHALL run 20 seeds for 24 months for all four policies and check all bands. The profiles SHALL be selectable by command-line argument and test trait.

#### Scenario: Nightly cost

- **WHEN** the nightly profile runs on a runner with at least 16 cores
- **THEN** it completes within 90 minutes, because runs are independent, each uses one simulation worker and they execute in parallel; `--seeds` scales the profile down on smaller machines

#### Scenario: Leaner world

- **WHEN** the harness builds its world
- **THEN** it omits systems that do not influence the economy (stream encoding, inspection projection)

#### Scenario: Smoke golden

- **WHEN** the smoke profile runs on an unmodified tree
- **THEN** its result equals the committed golden result exactly

#### Scenario: Profile selection

- **WHEN** the harness is started with `--profile nightly`
- **THEN** 20 seeds, 24 months and four policies are run

### Requirement: The harness reports in machine and human form

The harness SHALL write a JSON result per run, a CSV of weekly rows for all runs, and a short text summary listing each band with pass or fail and the offending seeds, and SHALL exit with code 0 only when every band holds.

#### Scenario: Exit code

- **WHEN** one band fails
- **THEN** the exit code is non-zero and the summary lists the failure

### Requirement: Goal thresholds are calibrated with the harness

The shipped scenario thresholds SHALL be calibrated so that the `reference` policy wins `first-steps` in at least 70 percent of 20 seeds and `negligent` wins it in at most 10 percent; the same two conditions SHALL hold for the other scenarios on their fixtures. The calibrated values SHALL be committed to the scenario data with the harness report.

#### Scenario: Calibration check

- **WHEN** the scenario calibration runs
- **THEN** the report shows the win rates per scenario and policy and the committed thresholds satisfy the conditions
