# BoogaBooster — Copilot instructions

> **This file is a condensed mirror of [`CLAUDE.md`](../CLAUDE.md), which is canonical.**
> It exists because GitHub Copilot in the IDE (VS Code / Visual Studio) does not read
> `CLAUDE.md`, while Claude Code and Copilot CLI do. **Change one, change both.** For
> anything not covered here, read `CLAUDE.md`.

## What this project is

BoogaBooster is a **digital twin of a carnival ride**, and simultaneously a demo project
for teaching AI-assisted development. Both halves matter: the code is held to production
standards, and the agents/skills/MCP servers/specs in the repo are part of the
deliverable. `README.md` has the full tour.

The ride: one motor-driven **Great Mill** (four 6 m arms) carrying four motor-driven
**Hubs** (four 2 m arms each), each hub arm tip carrying a passively free-spinning
**Gondola** — 16 gondolas × 2 seats = 32 seats. Five driven axes, 16 free axes. Physics
runs on a fixed 1/120 s timestep and is **deterministic**; telemetry publishes at 30 Hz.

## Non-negotiable rules

1. **Before writing, changing or designing any C#, consult the
   `4dotnet-csharp-style-guide` MCP server first.** It is authoritative on target
   framework, API style, DDD/value objects, solution and folder structure, and testing
   tooling. If it conflicts with existing code, follow the guide and flag the
   discrepancy.
2. **Modules reference other modules' `.Abstractions` project only** — never another
   module project.
3. **No endpoint mappings in `Api/Program.cs`.** Endpoints belong to the owning module's
   `Endpoints/` class; `Program.cs` only composes.
4. **No business logic in endpoints.** Parse the request into a command/query and
   dispatch to an `ICommandHandler<T>` / `IQueryHandler<T,R>`.
5. **Handlers implement `ExecuteAsync`, not `HandleAsync`.** The CQRS base classes own
   the ADR-0009 instrumentation (activity, timing, outcome counter, duration
   histogram); a handler adds only its own span tags by overriding `EnrichActivity`
   (plus `EnrichActivityWithResponse` on a query) and its own domain metrics through
   `BoogaBoosterTelemetry.Meter`. OTEL is configured only in `ServiceDefaults`.
6. **No magic numbers in the physics.** Every constant lives in
   `DigitalTwin/Domain/RideParameters.cs`, derived in `docs/`.
7. **Determinism is a hard requirement** in the simulation. Randomness goes through an
   injected sampler; time goes through `TimeProvider`.
8. **Accessibility must pass AXE / WCAG AA** in the Angular app.
9. Module and `Shared/` libraries must hold **≥ 80 % line coverage**.

## Repository layout

```
docs/            physics & simulation derivations — the spec for the maths
openspec/        spec-driven workflow: specs/ (contract) + changes/ (in flight)
src/
  BoogaBooster.slnx              solution (new XML .slnx format)
  Aspire/                        AppHost (run entry point) + ServiceDefaults
  FourDotnet.BoogaBooster.Api    single root API — composition root only
  DigitalTwin/                   the ride: physics, state machine, telemetry
  Queue/                         guests, groups, arrival simulation
  Weather/                       autonomous weather world
  Controller/                    empty scaffold (ride control/programmes)
  Shared/                        Core (DDD + CQRS base) + IntegrationMessages
  Tests/                         one xUnit project per module
  FourDotnet.BoogaBooster.App    Angular 22 operator dashboard
```

## Status

`DigitalTwin`, `Queue`, `Weather`, both `Shared/` libraries, the API and the Angular app
are **built and tested** (~330 xUnit v3 tests, 23 Vitest spec files). `Controller/` is
an **empty scaffold** — two `.csproj` files and nothing else.

## Backend — .NET 10 modular monolith

Each bounded context is a folder with two projects: `FourDotnet.BoogaBooster.<Module>`
(domain + application + features + endpoints) and
`FourDotnet.BoogaBooster.<Module>.Abstractions` (public contract). Module anatomy:

```
<Module>ModuleExtensions.cs    Add<Module>Module() - DI registration
<Module>ModuleOptions.cs       bound options, SectionName const
Domain/                        aggregates, value objects, domain services
Application/                   application services, stores, background services
Features/<FeatureName>/        <Feature>Command|Query + its handler
Endpoints/                     Map<Module>Endpoints() - HTTP surface
```

CQRS abstractions come from `FourDotnet.BoogaBooster.Core.Cqrs`; DDD base classes
(`DomainModel`, `DomainModelState`, `DomainValidationException`) from
`FourDotnet.BoogaBooster.Core`. Broken invariants throw `DomainValidationException`,
which endpoints translate to `400`.

**Orchestration.** `Aspire/…AppHost` is the run entry point and declares the whole
graph: a RabbitMQ container, a **programmatically declared** Dapr `pubsub` component
(Aspire Community Toolkit — there are no `components/*.yaml` files), the API with its
Dapr sidecar, and the Angular app as a Vite resource.

**Messaging.** Publish by injecting `IIntegrationEventPublisher`. Declare events in
`Shared/IntegrationMessages/Events/<Module>/` with `[TopicName]`. Subscribe with a
minimal-API endpoint annotated `.WithTopic(...)`, discovered via
`MapSubscribeHandler()` — no subscription YAML. Read
`src/Shared/FourDotnet.BoogaBooster.IntegrationMessages/README.md` before adding an
event. Current flow: Weather publishes `weather-updated` → Queue scales its arrival
rate; Queue publishes `group-queued`.

## Frontend — Angular 22

Standalone, **zoneless** (no zone.js polyfill), signal-first. Vitest + `axe-core`;
Three.js for the 3D visualization. Part of the Aspire graph via `AddViteApp`, so the
AppHost starts it; `npm start` runs it standalone.

Detailed rules live in `src/FourDotnet.BoogaBooster.App/.claude/CLAUDE.md` and are
authoritative in the app. Key points: do **not** set `standalone: true` (v20+ default);
signals for state (`signal`/`computed`/`update`/`set`, never `mutate`);
`input()`/`output()` functions over decorators; `inject()` over constructor injection;
`OnPush`; native control flow (`@if`/`@for`/`@switch`); host bindings in the `host`
object, not `@HostBinding`/`@HostListener`.

**PrimeNG 22** + `@primeuix/themes` are installed and are the mandated component
library, but **no component uses them yet** — the existing panels are hand-rolled. New
UI work should adopt PrimeNG (consult the `primeng` MCP server first).

Telemetry arrives over **SSE** (`GET /ride/telemetry/stream`); queue and weather are
polled HTTP. In dev, `/api/*` is proxied to the API by `proxy.conf.js`.

## Physics and the ride domain

`docs/` holds the derivations for the whole simulation — coordinate frames, the
fixed-timestep tick, `τ = Iα` and the symplectic integrator, the motor torque curve and
loss model, the passive gondola pendulums, the G-force and imbalance maths. **It is the
specification the physics code and its tests are written against** — read the relevant
doc before changing anything in `DigitalTwin/Domain`.
`docs/appendix-parameters.md` carries the baseline values and a sanity check.

**Terminology:** `docs/` says *cart*, the code says **gondola**. Same thing; use
*gondola* in code.

## Game roadmap (demo scope)

The OpenSpec changes in `docs/roadmap.md` (goal: `docs/vision.md`) are for a workshop
**demo**, not a production release. `docs/demo-scope.md` says what may be relaxed
(benchmarks, the 80 % coverage floor on new game modules, extra tolerance tiers, deep
currency checks) and what may not (green build per commit, existing ride tests, no
destructive actions, original names and art, seeded determinism). Mark skipped tasks
`(deferred: demo scope)` and list them in the PR. On a larger conflict, stop and ask.

## Testing

- Backend: **xUnit v3** + **Moq** + **Bogus** +
  `Microsoft.Extensions.TimeProvider.Testing`. **No FluentAssertions.** One test project
  per module under `src/Tests/`.
- Frontend: **Vitest**, plus `axe-core` accessibility specs (`*.a11y.spec.ts`).

## Automated PR review (CI)

`.github/workflows/code-quality-check.yml` runs **GitHub Copilot CLI headless**
(`copilot -p`) on every pull request into `main`, reviewing the diff against `CLAUDE.md`,
`.claude/skills/*`, `openspec/specs/` and `docs/`. **Two models review every pull request
independently** (a matrix — currently `claude-sonnet-5` and `gpt-5.6-terra`), their reviews
are compared, and one review is posted in which every comment names the reviewers that
reported it. It **fails the check only on `blocking` findings**, taking the union of the
reviewers — one model catching a MUST violation alone still fails; `major` and below are
advisory.

- Review instruction: `.github/code-review/review-prompt.md` — a checked-in, reviewable
  file. It is **not** a slash command; it must not move into `.github/prompts/` or
  `.github/skills/`, which are discovery paths.
- Comparison: `.github/code-review/compare-reviews.mjs` pairs the two reviews
  deterministically; `.github/code-review/compare-prompt.md` drives a third Copilot session
  that narrates it. **The narrative is commentary — it cannot change a severity or the
  check result.**
- Publisher: `.github/code-review/publish-review.mjs`. Test both scripts with
  `node --test .github/code-review/publish-review.test.mjs .github/code-review/compare-reviews.test.mjs`.
- Every model run — both reviews and the comparison — has no shell, no network, no GitHub
  tools, no MCP servers and no write permission; each job then asserts the model left the
  working tree untouched.
- **The comparison cannot reach what the gate is computed from** (design D16):
  `findings.merged.json` lives outside the working tree until the narrative session has
  exited, and every file that session can reach is hashed before and verified after. Do not
  weaken either control.
- The reviewing models are listed in the workflow’s `strategy.matrix.model` and nowhere
  else; the comparison and the publisher derive the roster from the per-model artifacts.
- Requires the `COPILOT_GITHUB_TOKEN` secret (a PAT for an identity with a Copilot seat —
  the Actions `GITHUB_TOKEN` cannot drive the CLI). See the README for setup and cost.
- A green check is **not** a green build: this workflow does not compile or test anything.
- **Do not weaken the review configuration casually.** The prompt requires any PR touching
  `.github/workflows/`, `.github/code-review/`, `CLAUDE.md`, this file or `.claude/` to be
  flagged at `major` or higher, precisely so such changes get human eyes.

## Spec-driven workflow (OpenSpec)

Features go through `openspec/` rather than straight into code: explore → propose
(`proposal.md`, `design.md`, `specs/*/spec.md`, `tasks.md`) → apply → archive.

`openspec/specs/` holds the 20 published capabilities — **treat them as the current
behavioural contract** and read the relevant one before changing a feature.
`openspec/changes/` holds work in flight: `passenger-safety` (over-speed protection) and
`single-riders-queue` are proposed but **not yet implemented**.

Copilot CLI runs the workflow through the prompts in `.github/prompts/`
(`opsx-explore`, `opsx-propose`, `opsx-apply`, `opsx-archive`).

**Apply never happens on `main`.** The `git-change-workflow` skill
(`.claude/skills/git-change-workflow/SKILL.md`) first creates a worktree
(`.claude/worktrees/<change>`) and a `<type>/<change>` branch from `origin/main`; the work
lands as logical Conventional Commits; when every task is done the branch is pushed and a
PR into `main` is opened with `gh`. The agent never merges the PR.

## Known style-guide discrepancies

Do not copy these; fix them when you touch the surrounding code.

- **DTO placement.** The mandated layout is
  `<Module>.Abstractions/DataTransferObjects/<Feature>/` holding `<Feature>Request` and
  `<Feature>Response`. Today the Queue module's DTOs (`PersonDto`, `QueuedGroupDto`,
  `QueueStatusDto`) sit at the Abstractions project root, and the DigitalTwin/Weather
  request records are declared inline inside their `Endpoints` classes.

## Common commands

```bash
cd src
dotnet run --project Aspire/FourDotnet.BoogaBooster.Aspire.AppHost   # run everything
dotnet build BoogaBooster.slnx
dotnet test  BoogaBooster.slnx
```

The AppHost needs **Docker** (RabbitMQ container), an initialised **Dapr CLI**
(`dapr init`), and RabbitMQ credentials as secret Aspire parameters:

```bash
dotnet user-secrets --project Aspire/FourDotnet.BoogaBooster.Aspire.AppHost \
  set Parameters:rabbitmq-username guest
dotnet user-secrets --project Aspire/FourDotnet.BoogaBooster.Aspire.AppHost \
  set Parameters:rabbitmq-password guest
```

Angular (from `src/FourDotnet.BoogaBooster.App`, package manager **npm 11**):

```bash
npm start                              # ng serve
npm run build
npm test                               # vitest via ng test
npx vitest run path/to/file.spec.ts    # single file
```

## MCP servers

Configured in the root `.mcp.json` (workspace scope). **First run of Copilot CLI in this
repo: start it interactively once and trust the folder**, or `.mcp.json` is never loaded
and `copilot mcp list` shows user-scoped servers only.

| Server | Use it for |
| --- | --- |
| `4dotnet-csharp-style-guide` | **Authoritative** for all C# and solution-design decisions. Query first, every time. Needs the `4dotnet-csharp-style-guide` executable on `PATH`. |
| `primeng` | PrimeNG 22 component API, props, events, theming, accessibility. |
| `microsoft-learn` | Official Microsoft/Azure docs and code samples. |

## Agents and skills

Shared with Claude Code from `.claude/`; Copilot CLI and VS Code read them natively.

- **Agents** — `csharp-expert` (any `.cs`/`.csproj`/`.slnx` work; style-guide and
  ADR-driven) and `angular-architect` (anything under the Angular app; zoneless,
  signal-first, PrimeNG-aware).
- **Skills** — the C# style-guide set (`csharp-solution-structure`,
  `csharp-feature-slices`, `csharp-minimal-api-endpoints`, `csharp-domain-model`,
  `csharp-observability`, `csharp-aspire`, `csharp-unit-testing`), `dto-organization`,
  `test-coverage`, `git-change-workflow`, `ai-tooling-sync`, and the `openspec-*`
  change-workflow skills.

Visual Studio only reads MCP servers from the solution directory, so `src/.mcp.json`
mirrors the root `.mcp.json` (with a `servers` key). It is generated — edit the root file.

## Adding tooling

- **MCP server** → root `.mcp.json`. Never machine-local (`copilot mcp add`), or
  teammates won't get it.
- **Agent or skill** → `.claude/` (Copilot CLI reads it natively; it also scans
  `.github/skills/`). If it belongs in the distributable plugin, sync it into
  `plugins/4dotnet-boogabooster/` and bump the plugin version.
- **Slash command** → written twice: `.claude/commands/<ns>/<name>.md` **and**
  `.github/prompts/<ns>-<name>.prompt.md`. Keep them in sync.
- **Repo-wide instructions** → written twice: `CLAUDE.md` (canonical) **and** this file.
  Keep them in sync.
- **CI review assets** → `.github/code-review/`. Deliberately *not* a discovery path, so
  the prompt does not surface as a bogus slash command or skill.
- **Agent `tools:` lists** → list **both** tool vocabularies (`Read`/`Bash`/
  `mcp__server__tool` *and* `read`/`execute`/`server/*`), using the server names from
  `.mcp.json`. Unknown names are ignored by both tools; omitting one leaves the agent
  tool-less in that tool.
- **After any of the above** → follow the `ai-tooling-sync` skill and run
  `node .claude/skills/ai-tooling-sync/scripts/sync-ai-tooling.mjs` (`--fix` regenerates the mirrors). It must report no errors.
