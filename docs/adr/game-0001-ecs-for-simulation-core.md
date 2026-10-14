# game-0001: ECS for the simulation core, DDD for the existing modules

- Status: Proposed (the user must accept)
- Date: 2026-10-14
- Deciders: project owner
- Related: game-0002, game-0003, game-0004, game-0005, game-0006
- Note: the `game-` prefix avoids collision with ADR-0003/0004/0007/0009, which live in the external style-guide MCP server and not in this repo.

## Context

BoogaBooster today is a digital twin of one carnival ride. The `DigitalTwin` module is a DDD module (aggregates, deterministic fixed 1/120 s symplectic physics, telemetry over SSE at 30 Hz). The roadmap (docs/roadmap.md) grows it into an Octopus ride and then a park simulation with thousands of guests, many rides and an economy. Two forces pull in different directions:

- The existing business modules need invariants, transactions, domain events and readable aggregates. DDD fits this and stays.
- A park simulation touches tens of thousands of small, homogeneous things (guests, gondolas, scene nodes) every tick. Object graphs with per-entity heap objects, virtual calls and pointer chasing do not scale and are hard to parallelise deterministically.

We need one place where the "many small things every tick" workload lives, with an explicit boundary to the DDD modules.

## Decision drivers

1. Throughput for 5,000+ entities at a fixed 120 Hz step.
2. Determinism (game-0006): fixed iteration order, replayable.
3. Parallelisability on the job system (game-0003) without locks.
4. Teachability in an AI workshop: a small core is readable end to end.
5. Fit with the modular monolith: no leakage of ECS types into other modules.
6. Dependency risk: currency and licence of third-party libraries (must be re-checked at apply time).

## Boundary: where aggregates end and the World begins

- A new module (working name `Simulation`, plus `Simulation.Abstractions`) owns the `World`: entity ids, component storage (game-0002), systems, the scene graph (game-0004).
- Aggregates (for example `Ride`, `Park`, `RideModel` definitions, economy ledger) remain in their DDD modules. They hold configuration and business state that needs invariants: ride model parameters, prices, opening state, ledger entries.
- The World holds per-tick runtime state: positions, angles, velocities, guest needs, queue positions.
- Direction of flow:
  - Aggregate -> World (commands): a handler creates or changes aggregates, then calls an `ISimulationGateway` abstraction to spawn entities / set component singletons. The World never reads aggregate repositories.
  - World -> Aggregate (events): systems emit plain integration events (for example `GuestEnteredRide`, `TicketSold`, `RideBrokeDown`) collected in an event buffer; a drain step after each tick (or batch of ticks) publishes them through the existing RabbitMQ/Dapr path. Aggregates react in their own transactions.
  - World -> DomainModel snapshot: the existing `DigitalTwin` telemetry DTOs are produced by a read-only projection system that copies component data into the DTOs. DTOs are never the storage format.
- Rule: no ECS type (Entity, Archetype, Query) appears in any `.Abstractions` project of another module. Only DTOs and events cross the boundary.

## Considered options

### Option A: Keep everything DDD (objects per guest/gondola)
- Pros: one paradigm, no new concepts, existing code and tests stay.
- Cons: object-per-entity allocation and pointer chasing; hard to run in parallel deterministically; physics already needed ad-hoc tuning at 16 gondolas; poor showcase for the data-oriented topics.

### Option B: Third-party ECS library in the new module
- Pros: tested, fast, features (queries, command buffers, sometimes multithreaded queries).
- Cons: API and lifecycle owned by others; library's own threading model may fight our job system and determinism rules; each candidate needs a currency check; less teachable (black box); NuGet/native dependency to keep current.

### Option C: Hand-rolled minimal ECS core (recommended)
- Pros: ~1-2k lines, fully readable in a workshop; storage layout (game-0002) and ordering are under our control, which determinism needs; no dependency risk; plugs directly into our job system.
- Cons: we own bugs and performance; no ecosystem tooling; must be benchmarked against a library to prove it is not embarrassing.

### Option D: ECS only on the TypeScript client, backend stays DDD
- Pros: smaller backend change.
- Cons: contradicts game-0005 (backend-authoritative simulation); physics and economy would live in the browser.

## Candidate table

Facts were checked on the official GitHub pages on 2026-10-14 unless marked. "Latest release/version" could not be read from the fetched pages (Releases sections were empty in the fetch), so it is unverified for all; apply-time currency check is mandatory.

| Candidate | Side | Storage model | Licence | Archived? | Notes | Verification |
|---|---|---|---|---|---|---|
| Arch (genaray/Arch) | .NET | Archetype and chunks (16 KB chunks), optional multithreaded queries | Apache-2.0 | No (no notice on page) | README shows `2.1.0-beta` in install command; targets netstandard2.1, net6, net8 listed - net10 support NOT verified | Page read; version/currency unverified |
| Friflo.Engine.ECS | .NET | Contiguous component memory, query chunks; storage terminology not stated on README | MIT | No (no notice; 2026 copyright) | Lists .NET Standard 2.1 and .NET 5-10, multithreaded queries, parallel query job | Page read; latest release unverified |
| Flecs.NET (BeanCheeseBurrito) | .NET wrapper | Native flecs C library (archetype-based per upstream, unverified here) | MIT | No | Needs native package `Flecs.NET.Native`; examples target .NET 9; multithreading not covered on page | Page read; native-interop risk for determinism and workshop portability |
| Hand-rolled core | .NET | Our choice (game-0002) | n/a | n/a | Full control | n/a |
| bitECS | TS | SoA over typed arrays (Float32Array etc.), plain arrays also possible | MPL-2.0 | No | "Thread-friendly" per README; SharedArrayBuffer use not confirmed on the page | Page read; MPL-2.0 file-level copyleft needs a licence check |
| miniplex | TS | Entities are plain JS objects, archetypal query indexes | MIT | No | Object-based, not data-oriented memory layout; fine for small UI-side state, not for 5k+ guests | Page read |

Unverified everywhere: NuGet/npm latest versions and dates, benchmark claims, .NET 10 compatibility of Arch and Flecs.NET.

## Decision

Adopt ECS for the simulation core in a new module, keep all existing modules DDD, and enforce the boundary described above. Implement a **hand-rolled minimal core** (entity ids with generations, SoA component arrays, queries, deterministic system scheduling) for the workshop, behind our own interfaces (`IWorld`, `ISystem`). Run a spike at apply time of change 2 (ecs-foundation) that benchmarks the hand-rolled core against Arch and Friflo on the guest-movement scenario; if a library beats it by more than the agreed threshold (Validation) AND passes the currency check AND can honour game-0006, replace the storage behind the interfaces. On the client, the simulation is not duplicated (game-0005); a TS ECS (bitECS, miniplex) is only a candidate for client-side render-state if profiling demands it.

Reasoning for a teaching workshop: a small readable core exposes every concept (SoA, generations, ordering, dependency counters) that the ADRs teach; a library hides them. The dependency and determinism costs of libraries also fall away.

## Consequences

- Good: determinism controllable end to end; teachable; no dependency churn; boundary keeps DDD modules clean.
- Bad: we maintain the core; risk of reinventing slower; two paradigms in one solution (needs onboarding docs).
- Neutral: interface seam makes a later library swap possible; a projection step costs copy time each telemetry tick.

## Validation

- Benchmark (BenchmarkDotNet, in a dedicated bench project): iterate 10,000 entities with Position+Velocity at <= 0.5 ms per pass on one thread on the CI reference machine; hand-rolled core within 2x of the best library candidate, otherwise revisit this ADR.
- Architecture test: no project other than the Simulation module references ECS types (NetArchTest or an equivalent reflection test) - zero violations.
- Replay test passes (game-0006).
- Boundary test: a spawned ride appears as entities after the gateway call; a World event reaches an aggregate handler exactly once.

## References

- docs/roadmap.md; game-0002..0006 in this folder.
- https://github.com/genaray/Arch (read 2026-10-14)
- https://github.com/friflo/Friflo.Engine.ECS (read 2026-10-14)
- https://github.com/BeanCheeseBurrito/Flecs.NET (read 2026-10-14)
- https://github.com/NateTheGreatt/bitECS (read 2026-10-14)
- https://github.com/hmans/miniplex (read 2026-10-14)
