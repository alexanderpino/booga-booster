## ADDED Requirements

### Requirement: The simulation advances in fixed steps of 1/120 s

The tick driver SHALL advance the simulation in whole fixed steps of exactly 1/120 s (`FixedStepHz = 120`). The simulation time of tick `n` SHALL be computed as `n / 120.0` seconds, never accumulated by repeated addition. The step passed to systems SHALL be the constant `1.0 / 120.0` regardless of wall-clock jitter.

#### Scenario: One second of wall time

- **WHEN** the driver is started and `TimeProvider` advances by exactly 1 s, then `Advance()` is called
- **THEN** exactly 120 ticks run and the tick counter is 120

#### Scenario: Less than one step

- **WHEN** only 8 ms (less than 1/120 s = 8.333 ms) elapsed since start
- **THEN** `Advance()` runs 0 ticks

#### Scenario: No drift over ten minutes

- **WHEN** wall time advances in 1 000 irregular increments summing to 600 s
- **THEN** after the last `Advance()` the tick counter is exactly 72 000

### Requirement: Time comes only from TimeProvider

The driver SHALL read time exclusively from an injected `TimeProvider` (`GetTimestamp`, `GetElapsedTime`, timers). It SHALL NOT use `DateTime.UtcNow`, `Stopwatch` statics or `Task.Delay` without a provider. Tick counts SHALL be derived with integer arithmetic from the timestamp frequency so that no rounding error accumulates.

#### Scenario: Fake clock

- **WHEN** a `FakeTimeProvider` is injected and advanced by 25 ms
- **THEN** `Advance()` runs 3 ticks

### Requirement: Catch-up is capped

If more than `MaxCatchUpSteps` (default 8, approx. 66.7 ms) steps are due in one `Advance()`, the driver SHALL run exactly `MaxCatchUpSteps` ticks, drop the remaining due steps by moving its baseline forward, and report the number dropped through the `boogabooster.sim.ticks.dropped` counter.

#### Scenario: Long stall

- **WHEN** 1 s elapsed without calling `Advance()` and the cap is 8
- **THEN** `Advance()` runs 8 ticks, 112 steps are dropped, and the next 1/120 s of wall time yields exactly 1 further tick

### Requirement: Start, stop and pause do not cause a burst

Starting the driver SHALL capture its baseline. Stopping and starting again SHALL reset the baseline without running ticks for the stopped interval. The tick counter SHALL continue from its previous value.

#### Scenario: Pause

- **WHEN** the driver ticks to 10, is stopped, 5 s pass, and it is started again
- **THEN** `Advance()` immediately afterwards runs 0 ticks and the counter is 10

### Requirement: Hosted loop

A hosted background service SHALL call `Advance()` repeatedly using a `TimeProvider`-based periodic timer with a period of 1/120 s, and SHALL stop promptly on cancellation. It SHALL only run when `Simulation:Enabled` is true (default false).

#### Scenario: Disabled by default

- **WHEN** the host starts with default options
- **THEN** no tick runs and diagnostics report tick 0

#### Scenario: Cancellation

- **WHEN** the host stops while the loop is running
- **THEN** the loop exits within one period and no tick is cut in half
