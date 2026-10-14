## ADDED Requirements

### Requirement: Every scheduled job runs exactly once

The job system SHALL execute each scheduled job exactly once, no matter how many workers run it, how the work is stolen between them, or how many threads schedule jobs at the same time. A job SHALL never run twice and SHALL never be dropped while the scheduler is running.

#### Scenario: A thousand jobs each run once

- **WHEN** 1000 jobs are scheduled, each incrementing its own slot in a 1000-element array, with 4 workers
- **AND** the caller completes the combined handle
- **THEN** every slot holds exactly 1

#### Scenario: Jobs scheduled from many threads each run once

- **WHEN** 8 external threads each schedule 10,000 jobs concurrently, each job incrementing its own slot
- **AND** all handles are completed
- **THEN** all 80,000 slots hold exactly 1

#### Scenario: Jobs that spawn jobs each run once

- **WHEN** a root job schedules 10 child jobs and each child schedules 10 grandchild jobs, and the whole tree is completed
- **THEN** the count of executed jobs is exactly 111 (1 + 10 + 100)

### Requirement: Work is stolen, not statically assigned

Each worker SHALL own a double-ended queue. The owner SHALL take its newest job first (last in, first out) and other workers SHALL steal the oldest job (first in, first out). An external (non-worker) thread SHALL submit through a global injection queue. An idle worker SHALL steal from a busy worker so that an unbalanced load still finishes with every worker busy.

#### Scenario: Idle workers take over an unbalanced load

- **WHEN** one worker's queue receives 10,000 jobs of equal cost and the other 3 workers start with empty queues
- **THEN** each of the other 3 workers executes at least 1 job from that queue

#### Scenario: Owner takes the newest job, thief the oldest

- **WHEN** a worker queue holds jobs pushed in the order A, B, C
- **THEN** the owner's next take returns C
- **AND** a thief's next steal returns A

#### Scenario: Stealing is not required for a single worker

- **WHEN** the worker count is 1
- **THEN** no steal is ever attempted and every job runs on that worker or, for an external-thread submission, after being drained from the injection queue

### Requirement: Counters and dependencies order jobs

Every scheduled job SHALL return a handle backed by a counter. A job scheduled with a dependency handle SHALL NOT start until that handle's counter has reached zero. Combining several handles SHALL produce a handle that completes only after all of them have completed. Passing an already-completed handle as a dependency SHALL be valid and SHALL not delay the job.

#### Scenario: Dependent job waits for its dependency

- **WHEN** job B is scheduled with job A's handle as its dependency
- **THEN** job B starts only after job A has finished
- **AND** every write job A made is visible to job B

#### Scenario: Combined handle waits for all

- **WHEN** handles of jobs A, B and C are combined and a job D depends on the combined handle
- **THEN** job D starts after A, B and C have all finished

#### Scenario: Completed handle as dependency

- **WHEN** a job is scheduled with a dependency handle whose job has already completed
- **THEN** the job runs normally without waiting

#### Scenario: Handle used after its frame ended

- **WHEN** a handle is used after the frame that created it has ended
- **THEN** the operation throws a stale-handle error and does not touch any other job

### Requirement: A job can await a handle without blocking its worker

A job written as an async method SHALL be able to `await` a job handle. While awaiting, the worker SHALL be free to run other jobs, and the remainder of the method SHALL resume as a new job when the handle completes. The continuation SHALL resume on a job-system worker and SHALL NOT resume on the .NET ThreadPool or on a synchronization context.

#### Scenario: Continuation resumes as a job

- **WHEN** an async job awaits a handle for 100 child jobs and then records the managed thread id it resumes on
- **THEN** the recorded thread is one of the scheduler's workers
- **AND** all 100 child jobs had finished before the continuation ran

#### Scenario: Awaiting does not block the worker

- **WHEN** the worker count is 1 and an async job awaits a handle whose jobs are queued on that same worker
- **THEN** the awaited jobs run and the continuation resumes, with no deadlock

#### Scenario: Await of a completed handle continues synchronously

- **WHEN** an async job awaits a handle that is already complete
- **THEN** the method continues without being re-queued

### Requirement: Worker count is configurable, including a single deterministic worker

The scheduler SHALL accept a worker count between 1 and 256, or 0 for "automatic", which resolves to the processor count minus one with a minimum of 1. A value outside that range SHALL be rejected when options are validated. A mode of `Inline` SHALL run every job synchronously on the calling thread in submission order, as the serial reference.

#### Scenario: Explicit worker count

- **WHEN** the scheduler is created with a worker count of 4
- **THEN** it reports a worker count of 4 and starts exactly 4 worker threads

#### Scenario: Automatic worker count on an 8-core machine

- **WHEN** the worker count is 0 and the machine has 8 logical processors
- **THEN** the scheduler starts 7 workers

#### Scenario: Automatic worker count never drops below one

- **WHEN** the worker count is 0 and the machine has 1 logical processor
- **THEN** the scheduler starts 1 worker

#### Scenario: Invalid worker count

- **WHEN** the options hold a worker count of 257 or a negative value
- **THEN** options validation fails with an error naming the worker count

#### Scenario: Inline mode

- **WHEN** the mode is `Inline` and jobs A, B, C are scheduled in that order
- **THEN** A, B and C run on the calling thread in the order A, B, C
- **AND** no worker threads are started

### Requirement: Cancellation stops pending work cooperatively

The scheduler SHALL accept a cancellation token when a frame or graph is run. After cancellation, jobs that have not started SHALL NOT run, jobs that are running SHALL be allowed to finish, and every counter SHALL still reach zero so that no waiter hangs. Completing a cancelled handle SHALL throw `OperationCanceledException`.

#### Scenario: Pending jobs are skipped after cancel

- **WHEN** 1000 jobs are queued with 1 worker, and the token is cancelled while job 10 is running
- **THEN** jobs queued after the cancellation do not run
- **AND** completing the combined handle throws `OperationCanceledException`

#### Scenario: No waiter hangs after cancel

- **WHEN** a caller is blocked completing a handle and the token is cancelled
- **THEN** the caller is released within 1 second

#### Scenario: Running job observes the token

- **WHEN** a running job polls the cancellation token from its context
- **THEN** it observes the token as cancelled after cancellation

### Requirement: Exceptions propagate deterministically

An exception thrown inside a job SHALL be captured, SHALL mark the job's counter faulted, and SHALL surface to whoever completes that handle or any handle that depends on it. Jobs that depend on a faulted job SHALL NOT run. When several jobs fault, the surfaced exception SHALL be an aggregate ordered by job ordinal (the order of scheduling), not by wall-clock time. The scheduler SHALL stay usable after a fault.

#### Scenario: Exception surfaces to the caller

- **WHEN** a job throws `InvalidOperationException("boom")` and the caller completes its handle
- **THEN** the caller receives an exception that contains the `InvalidOperationException` with message "boom"

#### Scenario: Dependents of a faulted job are skipped

- **WHEN** job B depends on job A and job A throws
- **THEN** job B does not run
- **AND** completing job B's handle surfaces job A's exception

#### Scenario: Multiple faults are ordered by ordinal

- **WHEN** jobs with ordinals 3 and 7 both throw, with job 7 throwing first in wall-clock time
- **THEN** the aggregate lists the exception from job 3 before the one from job 7

#### Scenario: Scheduler is usable after a fault

- **WHEN** a frame has faulted and a new frame is started
- **THEN** jobs in the new frame run normally

### Requirement: Shutdown is graceful and bounded

The scheduler SHALL support shutdown in two modes: `Drain` finishes all already-queued jobs, `Cancel` skips jobs that have not started. In both modes new submissions SHALL be rejected after shutdown begins, workers SHALL be joined within the given timeout, and shutdown SHALL be idempotent. Disposing the scheduler SHALL perform a `Cancel` shutdown.

#### Scenario: Drain finishes queued jobs

- **WHEN** 500 jobs are queued and shutdown is requested in `Drain` mode
- **THEN** all 500 jobs run before shutdown completes

#### Scenario: Cancel skips unstarted jobs

- **WHEN** 500 jobs are queued behind a long-running job and shutdown is requested in `Cancel` mode
- **THEN** the jobs that had not started do not run
- **AND** shutdown completes within the timeout

#### Scenario: Submission after shutdown

- **WHEN** a job is scheduled after shutdown has begun
- **THEN** scheduling throws an `ObjectDisposedException`-family error and no job runs

#### Scenario: Shutdown twice

- **WHEN** shutdown is called a second time
- **THEN** it returns without error

#### Scenario: Timeout is reported

- **WHEN** a job ignores cancellation and runs longer than the shutdown timeout
- **THEN** shutdown reports a timeout error naming the number of workers that did not stop

### Requirement: Worker affinity and priority are best-effort options

The scheduler SHALL accept options to set the worker thread priority and to pin each worker to a processor. These options SHALL be best-effort: on a platform or under permissions where the request cannot be honoured, the scheduler SHALL continue to work, SHALL record what was applied for each worker, and SHALL NOT throw.

#### Scenario: Priority applied

- **WHEN** the options request `AboveNormal` priority on a platform that allows it
- **THEN** each worker thread reports `AboveNormal`

#### Scenario: Unsupported pinning degrades quietly

- **WHEN** the options request pinning on a platform where per-thread affinity is unavailable
- **THEN** the scheduler starts normally
- **AND** the per-worker report states pinning was not applied

#### Scenario: Workers are named

- **WHEN** the scheduler starts 4 workers
- **THEN** their threads are named `boogabooster-job-0` through `boogabooster-job-3`

### Requirement: Steady-state frames do not allocate

After a warm-up of at least 2 frames, running a frame of the same shape (same graph, same job count) with non-async jobs SHALL allocate 0 bytes on the managed heap on the worker threads and on the calling thread. Async jobs MAY allocate one state-machine box per async job.

#### Scenario: Zero allocation per job

- **WHEN** a warmed-up frame of 10,000 non-async struct jobs is run 100 times
- **THEN** the managed allocation measured around the runs is 0 bytes
