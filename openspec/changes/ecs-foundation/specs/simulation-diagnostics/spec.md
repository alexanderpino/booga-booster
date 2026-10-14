## ADDED Requirements

### Requirement: The simulation emits metrics on the shared meter

The module SHALL publish these instruments on `BoogaBoosterTelemetry.Meter` (name `FourDotnet.BoogaBooster`): counter `boogabooster.sim.ticks` (unit `{tick}`), histogram `boogabooster.sim.tick.duration` (unit `ms`), histogram `boogabooster.sim.system.duration` (unit `ms`, tag `system` = system name), observable gauge `boogabooster.sim.entities` (unit `{entity}`), observable gauge `boogabooster.sim.archetypes`, counter `boogabooster.sim.ticks.dropped`, counter `boogabooster.sim.commands.discarded`. Tag values SHALL be low-cardinality (system names only; never entity ids). The module SHALL NOT configure OpenTelemetry itself.

#### Scenario: Tick metrics

- **WHEN** 10 ticks run with two systems
- **THEN** `boogabooster.sim.ticks` increases by 10, `boogabooster.sim.tick.duration` records 10 values and `boogabooster.sim.system.duration` records 20 values tagged with the two names

#### Scenario: Gauges follow the world

- **WHEN** 5 entities are created
- **THEN** the entities gauge reports 5 on the next collection

#### Scenario: No listener, no cost

- **WHEN** no listener is subscribed to the meter
- **THEN** per-system timing takes no timestamps and allocates nothing

### Requirement: No per-tick trace spans

The module SHALL NOT start an `Activity` per tick or per system (120 Hz would flood tracing). It SHALL start an activity named `simulation.snapshot` for snapshot capture and for save/restore.

#### Scenario: Snapshot span

- **WHEN** the snapshot endpoint is called
- **THEN** one activity `simulation.snapshot` is recorded with tag `tick`

### Requirement: Read-only diagnostics endpoint

`GET /simulation/diagnostics` SHALL return tick, simulation time in seconds, entity count, archetype count, system names in plan order, last tick duration in milliseconds, number of dropped ticks and number of discarded commands. It SHALL be served by a query handler derived from the shared `QueryHandler<,>` base so it appears in handler metrics.

#### Scenario: Fresh host

- **WHEN** the endpoint is called on a freshly started host with the simulation disabled
- **THEN** it returns tick 0, entity count 0 and an empty system list with status 200
