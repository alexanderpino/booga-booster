# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **Canonical instructions.** This file is the single source of truth for repo-wide AI
> instructions. Claude Code and GitHub Copilot CLI both read it natively.
> `.github/copilot-instructions.md` is a condensed mirror for GitHub Copilot in the IDE
> (VS Code / Visual Studio), which does **not** read `CLAUDE.md`. **Change one, change
> both.**

## What this project is

BoogaBooster is a **digital twin of a carnival ride**, and simultaneously a demo
project for teaching AI-assisted development. Both halves matter: the code is real and
held to production standards, and the agents/skills/MCP servers/specs in this repo are
themselves part of the deliverable. See `README.md` for the full tour.

The ride is a three-level nested rotating rig: one motor-driven **Great Mill** (four
6 m arms) carrying four motor-driven **Hubs** (four 2 m arms each), each hub arm tip
carrying a passively free-spinning **Gondola** — 16 gondolas, 2 seats each, 32 seats.
Five driven axes, 16 free axes. The physics runs on a fixed 1/120 s timestep and is
**deterministic**; telemetry publishes at 30 Hz.

## Status

The backend is substantially built and tested (~330 xUnit v3 tests; 23 Vitest spec
files in the frontend). Current state per module:

| Area | State |
| --- | --- |
| `DigitalTwin` | **Built.** Domain aggregates, physics, guarded lifecycle state machine, boarding coordinator, SSE telemetry stream, feature handlers, endpoints. |
| `Queue` | **Built.** Queue domain, arrival/interval planners, background filler service, weather subscription, endpoint. |
| `Weather` | **Built.** Weather domain and value objects, simulation service, disturbance commands, integration-event publishing, endpoints. |
| `Controller` | **Empty scaffold** — two `.csproj` files and nothing else. Intended home for ride control/operations logic (e.g. scripted ride programmes). |
| `Shared/Core` | **Built.** `DomainModel`/`DomainModelState`/`DomainValidationException` (ADR-0003) plus the `Cqrs/` handler abstractions. |
| `Shared/IntegrationMessages` | **Built.** `IIntegrationEvent`, `[TopicName]`, `TopicNameResolver`, Dapr-backed publisher. Has its own `README.md`. |
| `FourDotnet.BoogaBooster.Api` | **Built.** Pure composition root — no endpoints of its own. |
| Angular app | **Built.** Operator dashboard with live telemetry, queue, weather and 3D visualization. Wired into the Aspire graph as a Vite resource. |

## Architecture

Two stacks live under `src/`, wired together by .NET Aspire. The solution file is
`src/BoogaBooster.slnx` (the new XML `.slnx` format).

### .NET backend (.NET 10)

The backend follows the **Modular Monolith** layout mandated by the
`4dotnet-csharp-style-guide` MCP server (ADR-0004): each bounded context is a folder
containing a module project plus an `.Abstractions` project, shared code lives under
`Shared/`, and a single root API hosts every module.

- **`Aspire/FourDotnet.BoogaBooster.Aspire.AppHost`** — the Aspire orchestrator and the
  run/debug entry point for the whole system. `AppHost.cs` declares the distributed
  application graph: a RabbitMQ container, a **programmatically declared** Dapr
  `pubsub` component (via the Aspire Community Toolkit — there are no
  `components/*.yaml` files), the API with its Dapr sidecar, and the Angular app as a
  Vite resource. Uses `Aspire.AppHost.Sdk`.
- **`Aspire/FourDotnet.BoogaBooster.Aspire.ServiceDefaults`** — shared cross-cutting
  config (OpenTelemetry, health checks, service discovery, resilience). Every service
  project references this and calls `builder.AddServiceDefaults()` +
  `app.MapDefaultEndpoints()`.
- **`FourDotnet.BoogaBooster.Api`** — the ASP.NET Core minimal-API web service and
  single root API for the modular monolith; the only project registered in the AppHost
  graph. It is a **composition root only**: it calls each module's `AddXxxModule()` and
  `MapXxxEndpoints()` and contains no endpoints or business logic itself.
- **`Controller/`, `DigitalTwin/`, `Queue/`, `Weather/`** — one bounded-context module
  per folder. Each holds a module project (`FourDotnet.BoogaBooster.<Module>`) and its
  public-contract project (`FourDotnet.BoogaBooster.<Module>.Abstractions`). Other
  modules may reference the `.Abstractions` project **only**, never the module project.
- **`Shared/`** — cross-cutting shared libraries: `FourDotnet.BoogaBooster.Core` (DDD
  base classes per ADR-0003 and the CQRS abstractions) and
  `FourDotnet.BoogaBooster.IntegrationMessages` (the event-bus contract). Must not hold
  module-specific logic.
- **`Tests/`** — one xUnit project per module/shared library.

### Module anatomy (ADR-0007)

Every module owns its own DI wiring and HTTP surface:

```
<Module>/FourDotnet.BoogaBooster.<Module>/
  <Module>ModuleExtensions.cs    Add<Module>Module() - DI registration
  <Module>ModuleOptions.cs       bound options, SectionName const
  Domain/                        aggregates, value objects, domain services
  Application/                   application services, stores, background services
  Features/<FeatureName>/        <Feature>Command|Query + its handler
  Endpoints/                     Map<Module>Endpoints() - HTTP surface
```

**Endpoints contain no business logic.** They parse/validate the request into a command
or query and dispatch it to an injected `ICommandHandler<T>` / `IQueryHandler<T,R>`
(from `FourDotnet.BoogaBooster.Core.Cqrs`). Broken invariants throw
`DomainValidationException`, which endpoints translate to a `400`.

**Never add endpoint mappings to `Api/Program.cs`.** Add them to the owning module's
`Endpoints/` class and let `Program.cs` call the module's map method.

**Handlers implement `ExecuteAsync`, not `HandleAsync`.** The `CommandHandler<T>` /
`QueryHandler<T,R>` base classes own the observability plumbing ADR-0009 requires —
they start the activity on the shared `ActivitySource`, time the invocation and record
the outcome counter and duration histogram — and call the handler's
`protected override ExecuteAsync`. A handler adds only its own span tags by overriding
`EnrichActivity` (and `EnrichActivityWithResponse` on a query), and its own domain
metrics through `BoogaBoosterTelemetry.Meter`. The shared `ActivitySource`/`Meter` live
in `Shared/Core/Observability/BoogaBoosterTelemetry.cs` and are registered once, in
`ServiceDefaults`; never configure OTEL in a module or in the API host.

### Messaging

Cross-module coupling that should stay async goes over Dapr pub/sub on RabbitMQ:

- Publish by injecting `IIntegrationEventPublisher`.
- Declare an event in `Shared/IntegrationMessages/Events/<Module>/`, annotated with
  `[TopicName]`.
- Subscribe with a plain minimal-API endpoint annotated `.WithTopic(...)`, discovered
  via the host's `MapSubscribeHandler()`. No static subscription YAML.
- Read `src/Shared/FourDotnet.BoogaBooster.IntegrationMessages/README.md` before
  adding an event.

Current flow: Weather publishes `weather-updated` → Queue consumes it and scales its
arrival rate; Queue publishes `group-queued`.

### Angular frontend (Angular 22)

- **`FourDotnet.BoogaBooster.App`** — standalone, **zoneless** (no zone.js polyfill),
  signal-first Angular 22 app. Vitest for tests (with `axe-core` a11y specs), Prettier
  configured, Three.js for the 3D ride visualization. It **is** part of the Aspire
  graph (`AddViteApp`), so `dotnet run` on the AppHost starts it too; it can also be
  run standalone with `npm start`.
- Telemetry arrives over **SSE** (`GET /ride/telemetry/stream`); queue and weather are
  polled HTTP. In dev, `/api/*` is proxied to the API by `proxy.conf.js`, which reads
  the Aspire-injected service-discovery env var and falls back to
  `https://localhost:7001`.
- **PrimeNG 22** + `@primeuix/themes` are installed and are the mandated component
  library, but **no component uses them yet** — the existing panels are hand-rolled.
  New UI work should adopt PrimeNG (consult the `primeng` MCP server first).

## C# conventions

**IMPORTANT: When writing, modifying, or designing any C# (csharp) code, you MUST ALWAYS consult the `4dotnet-csharp-style-guide` MCP server FIRST.** This is not optional. Before producing C# code, changing project/folder structure, or making any solution-design decision, query that server for the applicable guidance and follow it.

- Use `list_documents` to see all available ADRs and guidelines, `search_documents` to find rules by keyword (e.g. `testing`, `minimal-api`, `ddd`), and `get_document` to read a document's full content.
- The server is authoritative on: target framework, API style (minimal APIs vs. controllers), domain modeling (DDD, value objects, entity state), solution/module/folder structure, unit testing tooling, and other C# standards.
- If a rule from the style guide conflicts with existing code in this repo, follow the style guide and flag the discrepancy.

### Known style-guide discrepancies

Do not copy these patterns; fix them when you touch the surrounding code.

- **DTO placement.** The mandated layout is
  `<Module>.Abstractions/DataTransferObjects/<Feature>/` holding `<Feature>Request` and
  `<Feature>Response`. Today the Queue module's DTOs (`PersonDto`, `QueuedGroupDto`,
  `QueueStatusDto`) sit at the Abstractions project root, and the DigitalTwin/Weather
  request records are declared inline inside their `Endpoints` classes. The
  `dto-organization` skill knows the correct layout — use it.

## Physics and the ride domain

- `docs/` holds the derivations for the whole simulation — coordinate frames, the
  fixed-timestep tick, `τ = Iα` and the symplectic integrator, the motor torque curve
  and loss model, the passive gondola pendulums, and the G-force/imbalance maths.
  **It is the specification the physics code and its tests are written against.** Read
  the relevant doc before changing anything in `DigitalTwin/Domain`.
- **No magic numbers in the physics.** Every constant lives in
  `DigitalTwin/Domain/RideParameters.cs`, with its derivation or rationale in `docs/`.
  `docs/appendix-parameters.md` carries the baseline values and a sanity check.
- **Determinism is a hard requirement.** The tick is a pure function of state at a
  fixed 1/120 s step. Randomness goes through an injected sampler
  (`IRideEventSampler`, `IWeatherSampler`, `IPersonGenerator`) that tests can seed.
  Time goes through `TimeProvider`.
- **Terminology.** `docs/` calls the passenger vehicles *carts*; the code calls them
  **gondolas**. Same thing. Use *gondola* in code.

## Game roadmap (demo scope)

The OpenSpec changes that grow the Octopus into a theme-park tycoon (listed in
`docs/roadmap.md`, goal in `docs/vision.md`) are for an AI-workshop **demo**, not a
production release. For those changes, `docs/demo-scope.md` decides what may be
relaxed (benchmarks, the 80 % coverage floor on new game modules, extra tolerance tiers,
deep currency checks, optional a11y specs, rare edge cases, security hardening) and what may not (green build per commit,
the existing ride tests, no destructive actions, original names and art, seeded
determinism). Deferring is allowed, silent dropping is not: mark the task
`(deferred: demo scope)` and list it in the PR description. Everything else in this file
still applies; when it conflicts with `docs/demo-scope.md` for more than a few lines,
stop and ask.

## Testing

- Backend: **xUnit v3** + **Moq** + **Bogus** + `Microsoft.Extensions.TimeProvider.Testing`.
  **No FluentAssertions.** One test project per module under `src/Tests/`.
- Module libraries (`FourDotnet.BoogaBooster.<Module>`) and `Shared/` libraries MUST
  hold **≥ 80 % line coverage**. The `test-coverage` skill maintains this and aims well
  past the floor.
- Frontend: **Vitest**, plus `axe-core` accessibility specs (`*.a11y.spec.ts`).
  Accessibility must pass AXE / WCAG AA.

## Spec-driven workflow (OpenSpec)

Features are not written straight into code. They go through `openspec/`:

1. `/opsx:explore` — think the idea through and clarify requirements.
2. `/opsx:propose` — generate a change folder with `proposal.md`, `design.md`,
   per-capability `specs/*/spec.md`, and `tasks.md`.
3. `/opsx:apply` — work the task list, tests included. **Never on `main`:** the
   `git-change-workflow` skill first creates a worktree (`.claude/worktrees/<change>`)
   and a `<type>/<change>` branch from `origin/main`, the work lands as logical
   Conventional Commits, and when every task is done the branch is pushed and a PR
   into `main` is opened with `gh`. The PR is never merged by the agent.
4. `/opsx:archive` — publish capability specs into `openspec/specs/` and move the
   change into `openspec/changes/archive/`.

`openspec/specs/` holds the 20 published capabilities — **treat them as the current
behavioural contract** and read the relevant one before changing a feature.
`openspec/changes/` holds work in flight; `passenger-safety` (over-speed protection)
and `single-riders-queue` are proposed but **not yet implemented**.

## Common commands

Run all .NET commands from `src/` (where `BoogaBooster.slnx` lives).

```bash
# Run the whole system via Aspire (dashboard + RabbitMQ + Dapr sidecar + API + frontend)
dotnet run --project Aspire/FourDotnet.BoogaBooster.Aspire.AppHost

# Build / test the whole solution
dotnet build BoogaBooster.slnx
dotnet test BoogaBooster.slnx
```

The AppHost needs RabbitMQ credentials, which are secret Aspire parameters — set them
once with `dotnet user-secrets`:

```bash
dotnet user-secrets --project Aspire/FourDotnet.BoogaBooster.Aspire.AppHost \
  set Parameters:rabbitmq-username guest
dotnet user-secrets --project Aspire/FourDotnet.BoogaBooster.Aspire.AppHost \
  set Parameters:rabbitmq-password guest
```

Running the AppHost also requires **Docker** (for the RabbitMQ container) and an
initialised **Dapr CLI** (`dapr init`).

Angular commands run from `src/FourDotnet.BoogaBooster.App` (package manager: **npm 11**):

```bash
npm start          # ng serve (dev server)
npm run build      # ng build
npm test           # vitest via ng test
npx vitest run path/to/file.spec.ts   # run a single test file
```

## Angular conventions

Detailed frontend rules live in `src/FourDotnet.BoogaBooster.App/.claude/CLAUDE.md` and are authoritative when working in the Angular app. Key points: standalone components (do **not** set `standalone: true` — it is the v20+ default), signals for state (`signal`/`computed`/`update`/`set`, never `mutate`), `input()`/`output()` functions over decorators, `inject()` over constructor injection, `OnPush` change detection, native control flow (`@if`/`@for`/`@switch`), and host bindings in the `host` object rather than `@HostBinding`/`@HostListener`. Accessibility must pass AXE / WCAG AA.

## AI tooling (Claude Code + GitHub Copilot)

This repo is set up so **Claude Code and GitHub Copilot (CLI, VS Code and Visual Studio) share the same MCP servers, agents, skills and commands**. Every asset has one canonical source under `.claude/` (or `.mcp.json` / `CLAUDE.md`); where a product cannot read it, a mirror is derived from it:

| Asset | Canonical source | Claude Code | Copilot CLI | VS Code | Visual Studio | Mirror |
| --- | --- | :-: | :-: | :-: | :-: | --- |
| MCP servers | `.mcp.json` (repo root) | ✅ | ✅ | ✅ | — | `src/.mcp.json` (`servers` key, solution dir) |
| Agents | `.claude/agents/*.md` | ✅ | ✅ | ✅ | — | — |
| Skills | `.claude/skills/<name>/SKILL.md` | ✅ | ✅ | ✅ | — | `.github/skills/` (OpenSpec-generated twins only) |
| Instructions | `CLAUDE.md` | ✅ | ✅ | — | — | `.github/copilot-instructions.md` (condensed, by hand) |
| Slash commands | `.claude/commands/<ns>/<name>.md` | ✅ | — | — | — | `.github/prompts/<ns>-<name>.prompt.md` |
| CI code review | `.github/code-review/` + `.github/workflows/code-quality-check.yml` | — | ✅ (headless, in Actions) | — | — | — |

**Keep them in sync with the `ai-tooling-sync` skill.** After touching any agent, skill,
command, MCP server, the plugin or either instruction file, run
`node .claude/skills/ai-tooling-sync/scripts/sync-ai-tooling.mjs` (add `--fix` to regenerate the mechanical mirrors). It must report no errors
before you commit.

### MCP servers available

| Server | Use it for |
| --- | --- |
| `4dotnet-csharp-style-guide` | **Authoritative** for all C# and solution-design decisions. Query it first, every time. Requires the `4dotnet-csharp-style-guide` executable on `PATH`. |
| `primeng` | PrimeNG 22 component API, props, events, theming, accessibility. Consult before using any PrimeNG component. |
| `microsoft-learn` | Official Microsoft/Azure documentation and code samples. |

### Automated PR review (CI)

`.github/workflows/code-quality-check.yml` runs **GitHub Copilot CLI headless** (`copilot -p`)
on every pull request into `main`, reviewing the diff against the standards in this file,
`.claude/skills/*`, `openspec/specs/` and `docs/`. **Two models review every pull request
independently** (a matrix — currently `claude-sonnet-5` and `gpt-5.6-terra`), their reviews
are compared, and one review is posted in which every comment names the reviewers that
reported it. It **fails the check only on `blocking` findings**, taking the union of the
reviewers: one model catching a MUST violation alone still fails the check.

- The review instruction is `.github/code-review/review-prompt.md` — a checked-in,
  reviewable file. It is *not* a slash command and must not move into `.github/prompts/`
  or `.claude/skills/`, which are discovery paths.
- The two reviews are paired by `.github/code-review/compare-reviews.mjs` (deterministic:
  same file, close line, same category or overlapping title) and then narrated by a third
  Copilot session driven by `.github/code-review/compare-prompt.md`. **That narrative is
  commentary — it cannot change a severity, add a finding, or move the gate.**
- Findings are published by `.github/code-review/publish-review.mjs`. Test both scripts with
  `node --test .github/code-review/publish-review.test.mjs .github/code-review/compare-reviews.test.mjs`.
- Every model run — both reviews and the comparison — has no shell, no network, no GitHub
  tools, no MCP servers and no write permission, and each job asserts the model left the
  working tree untouched.
- **The comparison cannot reach what the gate is computed from.** `findings.merged.json` is
  written outside the working tree and placed only after the narrative session has exited,
  and every file that session *can* reach is hashed before and verified after. Do not move
  that file into the scratch directory before the model runs, and do not remove the
  fingerprint check: without them, an injected narrative could rewrite the findings the
  check gates on (design D16).
- Adding or removing a reviewing model is an edit to the workflow’s `strategy.matrix.model`
  list and nowhere else: the comparison and the publisher derive the roster from the
  per-model artifacts.
- Requires the `COPILOT_GITHUB_TOKEN` secret; see the README for setup and cost.
- **Do not weaken the review configuration casually.** The prompt requires any PR touching
  `.github/workflows/`, `.github/code-review/`, `CLAUDE.md` or `.claude/` to be flagged at
  `major` or higher, precisely so such changes get human eyes.

### Agents and skills

- **Agents** — `csharp-expert` (any `.cs`/`.csproj`/`.slnx` work; style-guide and
  ADR-driven) and `angular-architect` (anything under the Angular app; zoneless,
  signal-first, PrimeNG-aware).
- **Skills** — the C# style-guide set (`csharp-solution-structure`,
  `csharp-feature-slices`, `csharp-minimal-api-endpoints`, `csharp-domain-model`,
  `csharp-observability`, `csharp-aspire`, `csharp-unit-testing`), `dto-organization`
  (DTO placement), `test-coverage` (≥ 80 % backend line coverage),
  `git-change-workflow` (worktree + branch → logical commits → push → PR for every
  OpenSpec change), `ai-tooling-sync` (Claude ⇄ Copilot parity for all of the
  above), and the `openspec-*` change-workflow skills.

**Rules when adding tooling:**

- **MCP server** → add it to the root `.mcp.json`. Do not add it machine-locally (`claude mcp add`, `copilot mcp add`), or teammates won't get it.
- **Agent or skill** → put it under `.claude/`. Both tools pick it up. If it is part of the distributable plugin, also sync it into `plugins/4dotnet-boogabooster/` and bump the plugin version.
- **Slash command** → must be written twice: `.claude/commands/<ns>/<name>.md` for Claude and `.github/prompts/<ns>-<name>.prompt.md` for Copilot. Keep the two in sync.
- **Repo-wide instructions** → also written twice: this file (canonical, read by Claude
  Code and Copilot CLI) and `.github/copilot-instructions.md` (condensed mirror, read by
  Copilot in the IDE). Keep the two in sync.
- **Agent `tools:` lists** → the two tools use different tool vocabularies (`Read`/`Bash`/`mcp__server__tool` vs. `read`/`execute`/`server/*`). Unknown names are ignored by both, so list **both** vocabularies on the `tools:` line, use the project server name from `.mcp.json` (never a personal plugin's `mcp__plugin_…` prefix), and check every `mcp__<server>__<tool>` against the live server; otherwise the agent silently loses tools in one of them.
- **Then run the sync check** → `node .claude/skills/ai-tooling-sync/scripts/sync-ai-tooling.mjs`. It must report no errors.

**First run of Copilot CLI in this repo:** start it interactively once and trust the folder. Workspace configuration (`.mcp.json`) is only loaded for trusted folders — until then `copilot mcp list` shows user-scoped servers only.

The `plugins/4dotnet-boogabooster/` plugin is dual-format: `.claude-plugin/plugin.json` is read by both `claude plugin` and `copilot plugin`.
