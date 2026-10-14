# Spec Delta

## Purpose

Defines the evidence that the coaster simulation is deterministic and correct and that it is fast enough for a park: golden masters on known layouts, analytic reference cases, property tests, thread-count independence, observability and measured performance budgets.

## ADDED Requirements

### Requirement: Known layouts reproduce checked-in golden records (Tier 1)

For each shipped fixture (`starter-coaster`, `starter-tall-hill`, `starter-tight-curves`) a checked-in golden record SHALL hold the outcome, verdict, lap duration in steps, maximum speed, the ratings, the windowed G peaks, the drop count and the hash of the train state sampled every 120 steps. A test run SHALL reproduce the record exactly on the reference platform and within a relative 1e-9 on other platforms. Changing a golden record SHALL need a deliberate commit that says why.

#### Scenario: Exact repeat

- **WHEN** the starter coaster is run twice in one process
- **THEN** the state hashes and all record values are identical

#### Scenario: The record is not vacuous

- **WHEN** one physics constant is changed by one unit in the last place in a test
- **THEN** the comparison with the golden record fails

#### Scenario: Bracket independent of the record

- **WHEN** the starter completes
- **THEN** the lap lasts between 35 s and 60 s and the maximum speed is between 11.5 m/s and 14.9 m/s, whatever the golden record says

### Requirement: Analytic cases pin the physics to closed forms (Tier 1)

The test suite SHALL contain the cases of the dynamics, geometry and metrics specs that have closed-form answers: frictionless descent (14.007 m/s after 3.193 s), valley symmetry, drag-only decay (16.609 m/s), rolling-only stopping distance (127.4 m), lift ramp (200 steps), trim brake exit speeds (3.0 and 6.93 m/s), station stop within 0.05 m, flat-curve lateral G (1.000), valley vertex G (2.000) and crest vertex G (0.000), each with the stated tolerance.

#### Scenario: All closed forms

- **WHEN** the closed-form suite runs
- **THEN** every case is within its stated tolerance

### Requirement: Properties hold on random inputs (Tier 1)

With plain seeded random generators and the seed printed on failure, the suite SHALL check: energy never increases outside drive zones; with drag off trajectories are independent of mass; joins are continuous; rotating a design by a multiple of 90 degrees and translating it by lattice vectors leaves metrics and ratings unchanged and positions exactly rotated; mirroring a design (left curves become right curves) leaves the longitudinal and vertical metrics unchanged and keeps lateral G magnitudes; write-read-write of files is identical; undo of any operation restores the previous hash; and the palette always offers at least one piece for any reachable design that has fewer than 256 pieces.

#### Scenario: Seeds

- **WHEN** a property fails
- **THEN** the failure message contains the seed and the step so that a single run reproduces it

#### Scenario: Mirror

- **WHEN** the starter coaster is mirrored
- **THEN** maximum speed, lap duration, highest drop and ratings equal the original within a relative 1e-9, and the lateral G magnitudes equal the original within the same tolerance

### Requirement: Results are independent of worker count (Tier 2)

A world with at least 50 coasters and 150 trains, run for 1,200 steps with 1, 2, 4, 8 and the processor count as workers and with the inline reference scheduler, SHALL give the same state hash at every 100th step. A soak of 100 runs with random worker counts SHALL give no difference.

#### Scenario: Thread-count independence

- **WHEN** the scenario runs under all worker settings
- **THEN** every hash sequence is identical to the inline one

### Requirement: Dynamics code uses only exactly rounded operations (Tier 1)

Source files of the dynamics and the compiled-track tables SHALL use only addition, subtraction, multiplication, division and square root on doubles, no fused multiply-add intrinsic, no vector intrinsic with machine-dependent width and no `Math` function other than `Sqrt`, `Abs`, `Min`, `Max`, `Floor` and `Round` with an explicit mode. A test SHALL scan these files and fail on any other `Math` call, `Interlocked` on a floating-point type or `Parallel` call.

#### Scenario: Scan

- **WHEN** a dynamics file calls `Math.Sin`
- **THEN** the scan test fails and names the file and line

### Requirement: The module is observable without per-step cost (Tier 1)

The module SHALL record on the shared meter: the test-run duration histogram, a test-run counter by outcome, the validation duration histogram, a counter of seat repairs, a counter of incidents by kind and a gauge of live designs. It SHALL NOT create a trace span per step or per train; handlers get spans through the standard handler base. Instrument names SHALL start with `boogabooster.coaster.`.

#### Scenario: Outcome counter

- **WHEN** a test run ends with `StalledOnClimb`
- **THEN** the counter `boogabooster.coaster.test_run.completed` is incremented with the tag `outcome` set to `StalledOnClimb`

### Requirement: Performance budgets are measured (Tier 1)

The benchmark project SHALL measure and record the following targets on a stated machine: C1 one train dynamics step at most 0.30 microseconds with 0 bytes allocated; C2 a step with metrics at most 0.80 microseconds; C3 compile of a 200-piece design at most 15 ms and an appended piece at most 1.5 ms; C4 validation of 200 pieces at most 5 ms excluding park queries; C5 a test run of a 60 s lap on 1,200 m of track at most 30 ms; C6 write, hash and read of a 200-piece file at most 3 ms; C9 undo or redo with revalidation at most 5 ms. A result above twice its target SHALL stop the change for review. Timing SHALL NOT be asserted in unit tests; zero allocation (C1) SHALL be.

#### Scenario: Allocation guard

- **WHEN** 1,000,000 dynamics steps run after warm-up
- **THEN** the allocated bytes are unchanged

#### Scenario: Recorded results

- **WHEN** the benchmarks have run
- **THEN** `docs/coaster/` holds the CPU model, runtime version and every result next to its target

### Requirement: Many coasters fit the tick budget (Tier 2)

C7: 2,000 trains on 500 coasters SHALL be stepped in at most 1.0 ms per tick with one worker, which is 12 percent of the 8.33 ms tick. The benchmark SHALL also report 2, 4 and 8 workers; speed-up is reported, not promised, because the per-coaster work is small. The test SHALL run through the job system with declared access sets.

#### Scenario: Scale run

- **WHEN** the scale benchmark runs
- **THEN** it records the tick time for 1, 2, 4 and 8 workers and the state hash, and the hashes are identical

### Requirement: The client meets its own budgets (Tier 1)

C8: building the mesh of a 100-piece design SHALL take at most 40 ms and appending a piece at most 5 ms in the reference test environment, recorded not asserted; draw calls and triangle counts SHALL be asserted as in the rendering capability. Client specs SHALL cover the composer, mesh builders, camera maths, dead reckoning and store logic with Vitest, and the editor states with axe-core specs.

#### Scenario: Specs exist

- **WHEN** the client test suite runs
- **THEN** it includes specs for composition, meshing, camera, dead reckoning, store and the seven accessibility states

### Requirement: Coverage and boundaries are enforced (Tier 1)

The `Coaster` module library SHALL have at least 80 percent line coverage. An architecture test SHALL fail if `Coaster` references any module project other than through `.Abstractions` projects, if any type has a public field, or if an ECS engine type appears in the `Coaster.Abstractions` project.

#### Scenario: Boundary

- **WHEN** a project reference from `Coaster` to the `DigitalTwin` module project is added
- **THEN** the architecture test fails
