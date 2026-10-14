# Proposal

## Why

After `guests-and-needs` thousands of guests walk, queue, ride and spend, but their money goes nowhere, the park has no balance to protect, no rating to improve, no goal to reach and no reason to price a ticket. Prices cannot yet shape queues, upkeep cannot yet punish a bad park and nothing can be won or lost. The economy closes the tycoon loop: income and costs flow through a ledger, the park rating computed from what guests feel feeds back into who arrives, and scenario goals give a session a shape. It is also the roadmap's first DDD aggregate fed by simulation events, which is the boundary ADR `docs/adr/game-0001-ecs-for-simulation-core.md` prescribes. The design is informed by the sourced research `docs/research/rct-gameplay.md` (sections 2, 3, 8, 9, 14, 17); every rule is re-specified in our own words with our own numbers.

## What Changes

- New DDD module `Economy` (`src/Economy/FourDotnet.BoogaBooster.Economy` + `.Abstractions`, tests in `src/Tests/FourDotnet.BoogaBooster.Economy.Tests`): aggregates for park finance (balance, ledger, loan), price plan, staff roster, research programme, marketing plan, awards and scenario, with CQRS handlers, endpoints and `Economy` DI wiring. The simulation stays ECS; only DTOs and events cross the boundary (ADR game-0001).
- **Money model**: a `Money` value type over integer cents (never `double`, `decimal` only at the API edge), checked arithmetic, defined rounding, integer JSON.
- **Pricing**: entrance fee and per-ride prices as validated plans pushed to the simulation.
- **Ledger**: income and expense categories, a weekly-closed ledger with a conservation invariant, weekly lines per category and per ride.
- **Loans and interest**: borrow and repay in steps, weekly interest from an annual basis-point rate, per-difficulty limits.
- **Upkeep and staff wages**: ride and facility upkeep by status, staff roster (handymen, mechanics enabled; levels, not entities) with weekly wages and effects through the simulation's staff levels.
- **Research and development**: funding levels, weekly progress points, an ordered item list unlocking ride types for the catalog, focus category, optional unlock-by-cash mode.
- **Marketing campaigns**: data-driven campaigns that add arrival demand and may discount the entrance fee, paid up front.
- **Park rating**: our own simplified formula (eight terms) with smoothing, computed every 12 s from guest metrics, written back to the simulation; the breakdown is exposed for teaching.
- **Awards**: seven data-driven awards (four positive, three negative) with criteria, duration and an arrival effect.
- **Scenario goals and difficulty**: goals (guests and rating held for weeks, monthly profit, debt-free), deadline, win and lose (including bankruptcy), three difficulty profiles plus sandbox.
- **Simulation bridge**: a deterministic calendar (week 96 s, month 4 weeks, year 12 months), a 12 s revenue flush and weekly close executed after the tick, all inputs to the simulation applied as logged commands.
- **Integration events** through the existing Dapr pub/sub conventions: `money-changed`, `rating-changed`, `scenario-status-changed`, with ordering numbers for idempotency.
- **Finance UI** with PrimeNG (charts with data-table alternatives, tables, forms), accessible, including a rating breakdown and a live cashflow view.
- **Persistence**: versioned, checksummed JSON snapshots with atomic writes behind an `IEconomyStore` seam, paired with the world save by a manifest.
- **Economy balancing harness**: headless runs of N in-game months over seeded guests with scripted player policies, structural acceptance bands that detect runaway inflation, money creation and bankruptcy, calibrated numeric bands frozen in a committed file, and a PR smoke versus nightly split.
- **Workshop simplifications**, explicit and listed: flat build costs, no land purchase or taxes, linear research list, staff as levels, a single currency, no interest on cash.
- **No breaking change**: the simulation works without the economy (guest inputs default to the values `guests-and-needs` documents); the economy is enabled by `Economy:Enabled`.

## Capabilities

### New Capabilities
- `money-model`: The `Money` type, arithmetic, rounding, limits and wire format.
- `park-pricing`: Entrance fee and ride price plans, validation, propagation to the simulation.
- `park-ledger`: Categories, the weekly close, conservation invariant and queries.
- `loans-and-interest`: Borrowing, repaying, interest and limits.
- `upkeep-and-staff`: Ride and facility upkeep, staff hiring, wages and effects.
- `research-and-unlocks`: Funding, progress, item order, focus, unlocks and unlock-by-cash.
- `marketing-campaigns`: Campaign types, costs, durations, demand and discount effects.
- `park-rating`: The rating formula, smoothing, bands and breakdown.
- `park-awards`: Award criteria, durations and arrival effects.
- `scenario-goals-and-difficulty`: Goals, win and lose, bankruptcy, difficulty profiles and sandbox.
- `economy-simulation-bridge`: Calendar, flush and close protocol, commands to and reads from the simulation, determinism.
- `economy-integration-events`: The three integration events, ordering, outbox and delivery rules.
- `economy-persistence`: Snapshot format, versioning, validation, atomic save and pairing with the world save.
- `finance-ui`: The PrimeNG finance, pricing, research, marketing, rating and scenario views and their accessibility.
- `economy-balancing-harness`: Headless seeded runs, policies, metrics, structural and calibrated acceptance bands.

### Modified Capabilities
<!-- None. No published capability changes its requirements. -->

## Impact

- **New code**: `src/Economy/**`, `src/Tests/FourDotnet.BoogaBooster.Economy.Tests/**`, `src/Tools/FourDotnet.BoogaBooster.Economy.BalanceHarness/**` (console runner over the same library code), `src/Shared/FourDotnet.BoogaBooster.IntegrationMessages/Events/Economy/**` (three events), `src/FourDotnet.BoogaBooster.App/src/app/economy/**`, `docs/economy/01-park-economy.md`, `02-park-rating.md`, `03-balancing.md`, data files `Economy/Data/research-items.json`, `awards.json`, `scenarios.json`, `difficulty.json`, `balance-bands.json`.
- **Edited code (small)**: `src/BoogaBooster.slnx` (new projects), `Api` host (`AddEconomyModule()` / `MapEconomyEndpoints()`), `Simulation.Abstractions` (gateway commands and report DTOs for economy inputs and outputs, if not already provided by `guests-and-needs`/`ride-on-ecs`), the placement flow of `park-world-grid`/`ride-catalog` gains a charge hook (reconciled in task group 1), the Aspire AppHost only if a state store is added later (not in this change), `CLAUDE.md` and `.github/copilot-instructions.md` (module row; `ai-tooling-sync` check).
- **Reads from** `Simulation.Abstractions` only: the gateway, revenue and metrics DTOs, the post-tick observer. **Writes to** the simulation only through logged gateway commands.
- **Dependencies**: no new runtime NuGet package in the backend. Frontend charting through PrimeNG's chart component needs the Chart.js package, a candidate that must pass the currency check at apply time (fallback: a small accessible SVG chart component without dependency).
- **Not affected**: `DigitalTwin`, `Weather`, `Queue`, existing endpoints and published specs.
- **Later changes**: `coaster-editor` reuses `TrySpend` for costs and the catalog price defaults; asset pipeline unaffected.

> **Quality bar:** this change is for a workshop demo, not a production release. See `docs/demo-scope.md` for what may be deferred and what may not.
