# Design

## Context

See `proposal.md` for motivation. Current state that shapes the approach:

- **No economy exists.** Guests (after `guests-and-needs`) pay entrance fees, ride prices and shop prices out of their own wallets; the simulation accumulates those payments in singleton counters (`RevenueAccumulator`, per-ride `RideTraffic`) and exposes `ParkMetrics`, `GuestCounters` and writable inputs `ParkRating`, `ParkPricing`, `SpawnModifiers`, `StaffLevels` (design of `guests-and-needs`, D15). Nothing consumes them yet.
- **Boundary rule** (ADR `docs/adr/game-0001-ecs-for-simulation-core.md`): aggregates with invariants and transactions stay in DDD modules; the World holds per-tick runtime state; commands flow aggregate to World through a gateway, events flow World to aggregate. The economy is the first such aggregate set. No ECS type may appear in `Economy.Abstractions`.
- **Repo conventions**: modular monolith with module and `.Abstractions` projects, `Features/<Feature>/` command or query plus handler with `ExecuteAsync`, endpoints without logic, `DomainModel` base with lifecycle state (ADR-0003), `DomainValidationException` mapped to HTTP 400, integration events under `Shared/IntegrationMessages/Events/<Module>/` with `[TopicName]` and `IIntegrationEventPublisher`, `TimeProvider` for time, xUnit v3 + Moq + Bogus, no FluentAssertions, 80 percent line coverage on module libraries. There is **no persistence layer anywhere in the repo** (all stores are in memory) and the Aspire AppHost declares only RabbitMQ and the Dapr pub/sub component.
- **Research** (`docs/research/rct-gameplay.md`, honoured with its VERIFIED, INFERRED and UNKNOWN labels in "Research traceability"): rating terms (s2), spawn inputs (s3), staff wages (s9), loans and interest, marketing and research funding (s14), workshop simplifications (s17). Mechanisms inform the design; all numbers are ours.
- **Frontend**: Angular 22 zoneless, PrimeNG 22 installed; `package.json` lists no charting library.

## Goals / Non-Goals

**Goals:**
- A money model that cannot lose a cent, a ledger that cannot lose track of one, and a conservation test that proves it.
- A rating and a spawn feedback loop that is stable, explainable and tunable, with a harness that proves it stays that way.
- Deterministic, replayable economy tied to the tick, so a golden master and the balancing harness are possible.
- Teachable code and UI: every rule has a named constant, a spec scenario and a test; the rating breakdown is on screen.
- Persistence that survives crashes and detects tampering without introducing a database.

**Non-Goals:**
- No database, no Dapr state store, no multi-park or multi-user economy (one park per host, `ParkId` reserved).
- No land purchase, taxes, per-piece construction pricing, scenery value, ride age depreciation, interest on cash, multiple currencies.
- No staff entities, shifts or patrol areas (levels only).
- No new ride content: unlockable types come from `ride-catalog`.
- No economy-driven changes to guest behaviour beyond the documented inputs (fee, rating, award and marketing modifiers, staff levels).
- No client-side economy logic; the UI only presents server state and sends commands.

## Decisions

### D1 - Legal posture and originality

Same posture as `guests-and-needs` D1: OpenRCT2 is GPL-3.0 and the game name is a trademark. Rules are re-specified in our words; every number (prices, wages, rates, awards, thresholds, rating weights) is our own and justified by its in-game meaning or by the harness; award and campaign names are original; no code, strings or assets are copied. `docs/economy/01-park-economy.md` carries the "Provenance and licence" section; task 15.6 reviews the diff for verbatim research constants.

### D2 - `Money`: an `int64` of cents, never floating point

`Money` is a `readonly record struct` (properties only) over cents with a hard range of plus or minus 10^15 (below JSON's 2^53 safe-integer limit of about 9.0 x 10^15, so no consumer loses precision). Arithmetic is checked. Proportional maths takes a numerator and denominator (permille, basis points) and rounds half away from zero in `long` with an `Int128` intermediate when needed. Splits give the remainder cents to the earliest parts. The only `decimal` is in API-edge parsing helpers, which are excluded by name in the architecture test. DTOs carry `...Cents` as `long`. Alternative `decimal` rejected: it is exact but slower, bigger on the wire and invites `double` conversions; integer cents also hash and serialise canonically.

### D3 - Aggregates and the single writer

Aggregates (each a `DomainModel` with lifecycle state): `ParkFinance` (balance, opening balance, loan, ledger, open-week totals; the only aggregate that moves money), `PricePlan`, `StaffRoster`, `ResearchProgramme`, `MarketingPlan`, `AwardBook`, `Scenario` (profile, goals, status, calendar anchor). Domain services (pure): `ParkRatingCalculator`, `AwardEvaluator`, `InterestCalculator`, `GoalEvaluator`. An application `EconomyCoordinator` orchestrates a command across aggregates (for example "start campaign" asks `ParkFinance` to charge and `MarketingPlan` to activate, both or neither, validated before either mutates). There is exactly one writer thread: the simulation thread (D4), so aggregates need no locks and the coordinator needs no transactions beyond validate-then-apply.

### D4 - The bridge: a post-tick hook on the simulation thread, with an inbox

Options for tying the economy to the simulation:

| | In-thread post-tick hook (chosen) | Async consumer of events from a queue |
|---|---|---|
| Determinism | Economy state changes at exact ticks; replay with two logs reproduces the hash | Consumer timing varies; ordering with ticks is lost unless events carry ticks and are applied at fixed points anyway |
| Latency of feedback | Rating and prices applied at the next tick boundary | Variable |
| Cost to the tick | Flush at most 2 ms and week close at most 5 ms (targets), in memory only | None on the tick, but then the harness cannot run it deterministically |
| Complexity | One writer, no locks | Concurrency on aggregates |
| Failure isolation | A bug in the hook can break a tick | Isolated |

The hook is invoked by the simulation host after each completed tick (the post-tick observer that `ride-on-ecs` provides for its event drain; if its shape differs, adapt at apply, the semantics are fixed). It does, in order: apply inbox commands, run the flush if `tick mod 1 440 == 0`, run the week close if `tick mod TicksPerWeek == 0`, run the month close every fourth week, publish a new immutable projection. File and network I/O never runs in the hook: saves run from the projection on another thread and events leave through the outbox. HTTP commands enter an inbox (a `Channel` with a completion source per command) and are applied in arrival order at the next tick boundary, with the tick written to the economy command log. Queries read the projection and never wait for the tick.

### D5 - Calendar and revenue flow

Week = 11 520 ticks (96 s; eight 12-second flush periods), month = 4 weeks (6.4 minutes), year = 12 months (76.8 minutes; 48 weeks, used by the interest formula). The week length is configurable but must be a multiple of the 1 440-tick flush period. A flush reads and resets the simulation's accumulators through one logged command and posts income to the balance at once (the player sees cash rise every 12 s) while ledger lines for the week are composed at the close (one line per category and ride, keeping the ledger small: about 60 lines per week for 20 rides). Expenses (upkeep, wages, research, interest) are posted at the close because they are weekly by definition; construction, marketing and loans post immediately as player transactions. Conservation: `balance = opening + sum(closed lines) + net(open week)` always.

### D6 - Park rating: our own formula, smoothed

Nine integer terms (spec `park-rating`) sum to at most 999: base 99, happy guests 350, uptime 150, quality 200, variety 150, crowd 50; penalties litter up to -200, lost guests up to -150, queues up to -100. Weights follow the research's structural insight (happiness and uptime dominate, litter and downtime hurt, moderate ride intensity is rewarded) but with a different, simpler structure, different numbers and a points budget that adds to exactly 999 so a perfect park is reachable and testable. **Ideal ride mix** is excitement 6.00 and intensity 5.00 (hundredths 600 and 500), our choice. **Smoothing**: the rating moves at most 25 points per 12 s toward the target. Why: spawn rate depends on rating, and rating depends on guest happiness, which depends on queues, which depend on guest count; without damping a transient (a ride breaking) could swing arrivals sharply and excite an oscillation. With a 25-point step the full range takes about 8 minutes, longer than a guest's visit stages, which makes the loop over-damped and easy to reason about. The crowd term is deliberately small (50) so more guests cannot by themselves raise the rating enough to attract yet more guests: positive feedback is bounded.

Worked example (also the spec scenario): 500 guests, happy share 700 permille, uptime 950, excitement 620 and intensity 520, 6 open rides, 30 old litter, 12 lost guests, average queue 240 s gives 99 + 245 + 142 + 192 + 112 + 12 - 60 - 10 - 14 = 718. At rating 718 the guest factor of spawning is `0.02 + 0.98 * (718 - 200) / 799`, about 0.655, so 1.96 guests per second before the other factors.

### D7 - Spawn feedback inputs

The economy writes four simulation inputs and nothing else about guests: `ParkRating`, `ParkPricing.EntranceFeeCents` (the effective fee after any discount), `SpawnModifiers` (award permille, marketing demand) and `StaffLevels`. All are written only on change, as logged gateway commands. Ride prices are written to per-ride price components. The guest module reads them as documented in its spawning spec, so the economy is replaceable (a test double writes the same four inputs).

### D8 - Numbers: expected magnitudes (hypotheses for the harness)

A back-of-envelope reference park, to show the numbers are in a plausible relation and to give the harness a first target (not facts, calibrated in group 13):

| Item | Estimate per week (96 s) |
|---|---|
| Arrivals at rating 700 | 1.9 guests/s x 96 s, about 180 guests |
| Entrance (fee 10.00) | about 1 800.00 |
| Ride tickets (about 6 rides per guest at about 2.00) | about 2 200.00 |
| Shop sales (about 3 items at about 3.00) | about 1 600.00 |
| Upkeep (8 rides x 150.00, 6 stalls x 40.00) | about 1 440.00 |
| Wages (2 handymen, 1 mechanic) | 110.00 |
| Research Normal | 250.00 |
| Interest on a 10 000.00 loan at 10 percent | 20.83 |
| **Operating profit** | **about 3 800.00 per week, 15 000.00 per month** |

Construction at 30 000.00 to 100 000.00 per large ride makes the early game tight (opening balance 15 000.00 plus a 10 000.00 loan on Normal) and payback takes weeks to months, which is the intended pace. Guest cash (25.00 to 65.00 per guest from `guests-and-needs`) bounds what a guest can spend, a natural cap on greedy pricing.

### D9 - Loans, upkeep, wages

Loan steps of 1 000.00, interest `loan * bp / (10 000 * 48)` per week on the outstanding principal, no compounding, no early-repayment fee. Ride upkeep is a catalog weekly value times difficulty factor times status factor (closed rides cost a quarter: a closed ride still ages, but you cannot starve a park of maintenance by closing everything). Wages by headcount at the close; hired staff are paid for the whole week, fired staff are not, a rounding simplification that avoids per-day tracking. Only handymen and mechanics are enabled because only they have an effect (litter cleaning here; ride repair delay in the catalog); security and entertainers are data rows with `Enabled=false` so the roster is complete and the next change can switch them on. Placebo staff would be a trap for teaching.

### D10 - Research as data

An ordered list in `Data/research-items.json` (id, category, points, unlocked catalog type ids), a focus category, four funding levels. Linear, deterministic, carry-over of surplus points, several completions per close. Cash shortcut at 5.00 per remaining point on profiles that enable it (Easy, Sandbox) so a 45-minute workshop can reach every ride without waiting. Item order and costs are authored with the catalog types of `ride-catalog`; a validation test fails start-up on unknown types. First item costs 100 points (one week at Normal) so the first unlock is visible within two minutes of play.

### D11 - Marketing and awards

Marketing: two data-driven types (`park-advertising`, `entry-discount`) with up-front payment, additive demand and a discount that the economy folds into the effective fee, so the simulation needs no discount concept. At most two active campaigns; extension rather than stacking. Awards: seven data-driven awards evaluated monthly, 12-week duration, effects summing (clamped to plus or minus 250 permille, the range the guest spawning spec accepts). Opposite awards on one metric have disjoint criteria by data validation. Awards are the research's "17 types" reduced to a teachable seven; their thresholds are assumptions (the research marks award thresholds UNKNOWN) and are calibrated with the harness.

### D12 - Scenarios and difficulty

Difficulty and scenario are data (`difficulty.json`, `scenarios.json`), validated at start-up. Goal types are an open set behind an evaluator interface (three shipped). Win is checked at the week close, deadline at the month close, bankruptcy at the week close. The sandbox profile disables bankruptcy and goals so a workshop can explore. Thresholds in the shipped scenarios are hypotheses; the harness calibrates them so a competent scripted player usually wins and a negligent one rarely does (spec `economy-balancing-harness`).

### D13 - Persistence: a JSON document behind a seam

`IEconomyStore` with one shipped implementation, `JsonFileEconomyStore`: a `System.Text.Json` document with a schema version, SHA-256 checksum (platform type, no dependency), atomic temp-file-then-replace write, three rotated backups, validation of every invariant on load (so a hand-edited or damaged file is rejected, not silently trusted), stepwise migrations, and a manifest tying the snapshot to the world save (tick and world hash). Why not a database: the repo has none, adding one is an architectural decision beyond this change, and the style guide's database rules gate execution. Why not Dapr state: the AppHost declares no state store, and adding one is an infrastructure change (candidate for a later change; the seam makes it a drop-in). Autosave runs from the projection on a background thread at each month close. Judgement call recorded: file persistence is local to one host, which is acceptable for a workshop and wrong for production; stated in Risks.

### D14 - Events and the outbox

Three events under `Events/Economy/` (spec `economy-integration-events`). The hook appends to a bounded in-memory outbox; a hosted service publishes through `IIntegrationEventPublisher` with three retries (200 ms, 1 s, 5 s). Semantics are at least once and unordered across types; sequence numbers per type let consumers drop duplicates. Coalescing keeps chatter to a few events a minute. There is no consumer in this change; a test-only subscriber documents the minimal-API `WithTopic` pattern as the messages README prescribes. The topic names `money-changed` and `rating-changed` are used verbatim from the brief.

### D15 - Finance UI

PrimeNG components after consulting the `primeng` MCP server: candidates `p-table` (ledger, breakdown, nearby lists), `p-chart` (line chart; needs the Chart.js package which is not installed: a new third-party dependency that must pass the currency check, else an own accessible SVG component), `p-progressbar` (goals, research), `p-inputnumber`, `p-select`, `p-confirmdialog` or `p-dialog`, `p-tag`, `p-message`, `p-tabs` (candidate list, names verified at apply). Structure: a lazy route `economy` with a shell and child views (overview, ledger, cashflow, pricing, staff, research, marketing, rating, awards, scenario). Data via a signal-based service polling every 2 s while visible, commands via HTTP POST with typed errors. Accessibility rules in the spec: charts always have a table alternative and a text summary; values never rely on colour; a polite live region announces transactions; dialogs trap and restore focus. Currency display: integer cents formatted with two decimals and a configurable label (`coins`), never a floating-point division for arithmetic, only for display.

### D16 - Balancing harness: a library, a console runner and a test profile

`Economy.BalanceHarness` is a library with `ParkFixtures` (`starter-park`, `mid-park`, `thrill-park`; each a deterministic builder over `ParkScenarioBuilder` from `guests-and-needs`), `PlayerPolicy` scripts (`reference`, `greedy`, `free`, `negligent`), `HarnessRunner` (manual tick driver, no stream or projection systems, one worker per run, runs in parallel across seeds), `BandChecker` (structural bands B1 to B7 and calibrated bands from `balance-bands.json`), `Calibrator`, and report writers (JSON, CSV, text). A console project `src/Tools/FourDotnet.BoogaBooster.Economy.BalanceHarness` wraps it (`--profile smoke|nightly`, `--seeds`, `--months`, `--fixture`, `--calibrate`). xUnit tests (trait `Balance`) run the smoke profile in CI. Structural bands are written to hold whatever the calibrated numbers are: they test the shape of the feedback loops (conservation, price response, punishment of neglect, survival of a reasonable policy, stability, no runaway), which is what "detect runaway inflation or bankruptcy" needs. Cost: 24 months is 1.1 million ticks per run; at about 0.5 to 1 ms per tick (1 000 to 3 000 guests) that is 10 to 20 minutes per run per core, so nightly needs a many-core runner; the smoke profile (3 months) takes minutes. A CI workflow for the nightly run edits `.github/workflows/`, which the review prompt flags at major severity; it is therefore an optional task gated on the user's decision, not part of the default scope.

### D17 - Determinism of the economy

No wall clock inside aggregates (`TimeProvider` only at the outbox edge for the occurred-at stamp), no floating point, no `Dictionary` enumeration where order matters (ordered lists keyed by ids), ledger lines composed in category then subject order, an `EconomyStateHash` (FNV-1a 64 over fixed-order integers, same algorithm family as the world hash). Replay needs two logs: the simulation input log (including economy outputs) and the economy command log (operator commands with the tick applied). The golden master pins the smoke run.

### D18 - Charging construction without coupling the modules

Placement belongs to `park-world-grid`/`ride-catalog` (Simulation module). To charge for it without making the Simulation module depend on Economy, those modules call an `IBuildChargeGate` (declared in `Simulation.Abstractions` by the placement owner; default implementation allows everything) and `Economy` registers the real implementation, which runs on the simulation thread and calls `ParkFinance.Charge`. Refund on demolition goes through the same gate. If the earlier changes did not declare such a seam, group 1 adds it as a small, tested addition to those modules (reconciliation task) and the PR notes it.

### D19 - Workshop-friendly simplifications (explicit list)

Cut or reduced compared with the research: no land rights or purchase, no park-value or company-value formula, no award breadth beyond seven, no ride age or value depreciation, flat construction price per type (no per-piece pricing), marketing limited to two types, research is a linear list, staff are levels, four difficulty profiles instead of scenario-specific interest and funding, no recurring loan interest compounding, a 12-month 4-week calendar instead of eight months with uneven days. Each simplification is stated in `docs/economy/01-park-economy.md` with the effect it removes. Sandbox mode exists so a workshop can ignore money entirely.

## Dependencies and contracts

**Requires from `guests-and-needs` (reconcile names at apply):** singleton components and gateway commands: `ParkRating` (set), `ParkPricing` (set effective entrance fee), `SpawnModifiers` (set award permille and marketing demand), `StaffLevels` (set), `RevenueAccumulator` and per-ride `RideTraffic` (read-and-reset command returning a report DTO), `ParkMetrics` and `GuestCounters` (read: guests in park, happy share permille, lost guests cumulative, old litter, value for money, suggested capacity, average queue seconds). The rating also needs ride-derived metrics: uptime permille, average excitement and intensity of open rated rides and the open ride count. If `ParkMetrics` lacks them, task 1.3 adds them to the guest metrics system (reads `RideStatus`, `RideRatings` and the catalog's downtime figure) with tests. Also required: the `Park` mode, the input log, `ParkScenarioBuilder`, the thread-count harness, the golden-master approach.

**Requires from `ride-catalog` (intent):** per type: build cost, weekly upkeep, default price, category for research, unlock availability API (`UnlockRideType(typeId)` and a query of available types), per ride: `RideStatus`, downtime percentage, value, ratings, mechanic-count consumption. The placement and demolition flow with the charge gate (D18).

**Requires from `park-world-grid` / `ride-on-ecs` / `ecs-foundation`:** `ISimulationGateway` with a logged-command facility, the post-tick observer, `FixedTickDriver` in manual mode, `IBuildChargeGate` seam, the canonical world hash for the pairing manifest.

**Requires from `Shared/IntegrationMessages` and `Shared/Core`:** `IIntegrationEvent`, `[TopicName]`, `IIntegrationEventPublisher`, `DomainModel`, `DomainValidationException`, CQRS base classes, `BoogaBoosterTelemetry`.

**Provides to `coaster-editor`:** `IBuildChargeGate` implementation (it charges track costs), price defaults and the research unlock path; `Money` for costs.

**Provides to the client and others:** endpoints `GET /economy/summary`, `/economy/ledger`, `/economy/cashflow`, `/economy/rating`, `/economy/awards`, `/economy/research`, `/economy/marketing`, `/economy/staff`, `/economy/prices`, `/economy/scenario`, `/economy/scenarios`; commands `POST /economy/entrance-fee`, `/economy/ride-prices/{rideId}`, `/economy/loan/borrow`, `/economy/loan/repay`, `/economy/staff/hire`, `/economy/staff/fire`, `/economy/research/funding`, `/economy/research/focus`, `/economy/research/unlock`, `/economy/marketing/campaigns`, `/economy/scenario/start`, `/economy/save`, `/economy/load`; integration events `money-changed`, `rating-changed`, `scenario-status-changed`; DTOs under `Economy.Abstractions/DataTransferObjects/<Feature>/`.

**Configuration:** `Economy:Enabled`, `Economy:Calendar:TicksPerWeek`, `Economy:Storage:Directory`, `Economy:Currency:Label`, `Economy:Outbox:Capacity`.

## Research traceability

| Mechanism | Research section, label | Use in this design |
|---|---|---|
| Rating terms: happy share, uptime, ride quality, litter, lost guests | s2 VERIFIED formula | Structure only; own weights, own ideal mix, own caps, smoothing added |
| Casualty penalty | s2 VERIFIED | Dropped |
| Awards: 17 types, spawn effect plus or minus 25 percent | s2, s3 VERIFIED; thresholds UNKNOWN | Seven awards, own thresholds, clamped effect |
| Entrance fee against ride value cuts arrivals | s3 VERIFIED | Implemented in `guests-and-needs`; economy supplies the fee |
| Staff wages monthly, paid weekly at a quarter | s9 VERIFIED; hiring cost and caps UNKNOWN | Own weekly wages; free hiring; cap 50 (assumption) |
| Loan, interest per week formula, rate field | s14.1 VERIFIED; yearly percentage INFERRED | Own basis-point rate and 48-week year; no copy of the formula |
| Marketing: six campaign types, weekly price and added spawn chance | s14.2 VERIFIED | Two types, own prices and demand, discount folded into fee |
| Research funding levels and progress | s14.3 VERIFIED; absolute timing UNKNOWN | Own levels, points, costs; timing defined by our week |
| Park value and company value | s14.1 VERIFIED | Not used |
| Income per ride and profit estimate | s14.4 | Replaced by ledger lines per ride |
| Weekly rhythm (4096 ticks) | s0.1 VERIFIED | Weekly close kept as a mechanism; our week is 96 s |
| Workshop simplifications | s17 INFERRED recommendations | Adopted, listed in D19 |

Apply-time verification task: none of the research's "re-verify" numbers is a dependency; if calibration against the original is ever wanted, re-verify the cited lines at the pinned commit first.

## Risks / open questions

- **Contracts are intent.** `guests-and-needs`, `ride-catalog`, `park-world-grid` and `ride-on-ecs` are not yet implemented when this is written. The accumulator and metrics names, the post-tick hook shape, the charge gate and the unlock API are assumptions; group 1 reconciles them and may need small additions to those modules.
- **The hook can slow the tick.** In-thread economy work is budgeted (2 ms flush, 5 ms close). A bug or an unusually large ledger could exceed it; compaction and the lean projection are the mitigations; the benchmark records the numbers.
- **Positive feedback loop.** Rating raises arrivals, which raise income, which pays for more rides. Damping (smoothing, small crowd term, capacity factor in spawning) and the harness bands B6 and B7 guard it; if the harness finds oscillation, lower the smoothing step before touching weights.
- **Numbers are unproven.** Rates, prices, thresholds and scenario goals are hypotheses; the first calibration may change many of them and the golden masters with them (deliberate commits).
- **File persistence is host-local.** A second host or container restart without a volume loses the save. Acceptable for a workshop; a Dapr state store or database behind `IEconomyStore` is the production answer and a separate infrastructure change.
- **Chart dependency.** `p-chart` needs Chart.js (not installed, currency not verified here). A fallback SVG component is specified so the change is not blocked by that check.
- **Pairing economy and world saves** assumes `ecs-foundation`'s save/restore and hash; if the world cannot be restored mid-run (guests design: no save while field builds are pending) the economy load is refused with a clear message.
- **Nightly CI workflow** edits `.github/workflows/`, which the repo's review config flags at major severity; included as an optional task awaiting your decision. Without it, the nightly profile is a documented command.
- **Roles without effect.** Mechanics have an effect only if `ride-catalog` consumes `StaffLevels.Mechanics`; if it does not, mechanics are disabled in data until it does (task 1.4 checks).
- **Currency check list for apply time** (nothing asserted): Chart.js and the PrimeNG chart component's requirements, BenchmarkDotNet (already a candidate), no new backend package. `System.Text.Json`, `System.Security.Cryptography` and `System.Threading.Channels` are platform libraries of the pinned runtime.
- **Open questions for the user** (batched in task 1.5): enable the nightly CI workflow? Is a currency label of `coins` acceptable or should a real currency symbol be shown? Should `Economy:Enabled` default to true once `Park` mode is the default (later decision)?
- **AI tooling sync.** CLAUDE.md status table gains an `Economy` row and the messages README gains three topics; the Copilot mirror and the sync check are part of the final group.

## Migration Plan

1. Land with `Economy:Enabled=false` (default): no endpoints mapped, no hook registered, no behaviour change.
2. Enable with `Simulation:Guests:Mode=Park` and a scenario; the economy starts a scenario on first request or by configuration `Economy:StartScenario`.
3. Rollback: set `Economy:Enabled=false` and restart; the simulation falls back to the guests module's default inputs. Snapshot files are ignored. No database to migrate.
