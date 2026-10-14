## ADDED Requirements

### Requirement: Range partitioning is a pure function of count and grain

A parallel-for over `count` items with grain size `grain` SHALL split the range `[0, count)` into `ceil(count / grain)` contiguous chunks. Chunk `i` SHALL cover `[i * grain, min((i + 1) * grain, count))`. The partitioning SHALL NOT depend on the worker count, on the machine, or on timing. A grain size below 1 or a negative count SHALL be rejected.

#### Scenario: Partitioning of 10 items with grain 4

- **WHEN** a parallel-for runs over 10 items with grain 4
- **THEN** exactly 3 chunks are produced
- **AND** they cover `[0,4)`, `[4,8)` and `[8,10)`

#### Scenario: Same partitioning for every worker count

- **WHEN** the same parallel-for (1,000,003 items, grain 1024) runs with 1, 2, 4 and 8 workers
- **THEN** the set of chunk ranges is identical for all four runs

#### Scenario: Empty range

- **WHEN** a parallel-for runs over 0 items
- **THEN** no chunk runs and the returned handle is already complete

#### Scenario: Invalid grain

- **WHEN** a parallel-for is requested with grain 0
- **THEN** it throws an argument error naming the grain

### Requirement: Every item is visited exactly once

A parallel-for SHALL call the body for every index in `[0, count)` exactly once, with no index skipped or repeated.

#### Scenario: Full coverage

- **WHEN** a parallel-for over 1,000,000 items runs with 4 workers and marks each visited index
- **THEN** all 1,000,000 marks are set exactly once

#### Scenario: Chunks do not overlap

- **WHEN** two different chunks of the same parallel-for are executed at the same time
- **THEN** their index ranges are disjoint

### Requirement: Reduction order is fixed

A parallel-reduce SHALL compute one partial result per chunk and SHALL combine the partials in ascending chunk order, independent of which worker finished first. For a non-associative operation such as floating-point addition, the result SHALL therefore be bit-identical for every worker count.

#### Scenario: Floating-point sum is bit-identical across worker counts

- **WHEN** a parallel-reduce sums 1,000,000 `double` values generated from seed 42, with grain 4096, on 1, 2, 4 and 8 workers
- **THEN** the four results have identical bit patterns

#### Scenario: Result equals the serial chunked reference

- **WHEN** the same sum runs in `Inline` mode
- **THEN** its bit pattern equals the threaded results

#### Scenario: Reduction differs from naive sequential order only by design

- **WHEN** the grain is changed from 4096 to 1024
- **THEN** the result MAY differ in the last bits from the grain-4096 result
- **AND** the documentation states that the grain is part of the result's definition

### Requirement: Grain size is explicit and has no worker-dependent default

The parallel-for and parallel-reduce operations SHALL require the caller to pass a grain size or SHALL use a documented constant default that does not depend on the worker count or the processor count.

#### Scenario: Default grain is a constant

- **WHEN** the default grain is read on machines with 2 and 64 logical processors
- **THEN** both report the same value

### Requirement: Imbalanced chunks still finish balanced

When chunk costs differ widely, the system SHALL still keep all workers busy by stealing chunks, without changing the chunk boundaries.

#### Scenario: Triangular workload

- **WHEN** chunk `i` costs time proportional to `i` and the loop runs on 4 workers
- **THEN** each worker executes at least one chunk
- **AND** the chunk boundaries equal those of the 1-worker run

### Requirement: A fault or cancellation in one chunk stops the rest

If a chunk throws, or the token is cancelled, chunks that have not started SHALL NOT run, and the exception or cancellation SHALL surface through the loop's handle with the same rules as other jobs.

#### Scenario: Chunk throws

- **WHEN** chunk 5 of 100 throws
- **THEN** completing the loop's handle throws an aggregate containing that exception
- **AND** chunks not yet started do not run
