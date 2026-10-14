# Proposal

## Why

With `booga` (`infra-template-tool`) and the published images (`container-images-ci`) in place, the C# Aspire AppHost is no longer needed to run BoogaBooster. Replacing it with a declarative template means the whole local system is described in one YAML file, and starts with one command on a machine that has only `booga` and Docker.

## What Changes

- Add **`infra.yaml`** at the repository root: a CloudFormation-style template that declares RabbitMQ, the Dapr pub/sub component, the API container with its Dapr sidecar, and the frontend container. It has parameters for the RabbitMQ credentials and the image tag, and outputs for the dashboard URL and the RabbitMQ management UI.
- `booga start infra.yaml` becomes **the** way to run the system.
- **BREAKING (developer workflow):** remove `src/Aspire/FourDotnet.BoogaBooster.Aspire.AppHost` and `src/Aspire/aspire.config.json`. `dotnet run --project …AppHost` stops working, and the AppHost's user-secrets no longer hold the RabbitMQ credentials. The credentials become template parameters.
- Move `FourDotnet.BoogaBooster.Aspire.ServiceDefaults` to **`src/Shared/FourDotnet.BoogaBooster.ServiceDefaults`**. It is application code (OpenTelemetry, health checks, resilience, service discovery) and its behaviour does not change. The `src/Aspire/` folder disappears.
- **Remove the Aspire-specific rules.** The `4dotnet-csharp-style-guide` and ADR-0008 no longer govern how the system is composed and run. The `csharp-aspire` skill is deleted, and `CLAUDE.md`, `.github/copilot-instructions.md`, the other skills, the `csharp-expert` agent, the distributable plugin and the code-review prompt are updated to describe the template instead of the AppHost.
- Update the README, `.vscode/launch.json`, code comments that mention the AppHost, and the in-flight OpenSpec changes that assume the AppHost (notably `workshop-prerequisites`).

## Capabilities

### New Capabilities
- `system-template`: the repository's `infra.yaml`, meaning what it declares, its parameters and outputs, and that it is the single entry point for running the whole system locally.

### Modified Capabilities
- `dapr-pubsub-infrastructure`: the pub/sub component, RabbitMQ broker and sidecars are declared in the system template instead of programmatically in `AppHost.cs`. The "not YAML" requirement is removed.
- `weather-api-client`: the `/api` proxy requirement no longer refers to the Aspire AppHost. Under the template, the frontend container's reverse proxy forwards `/api`; the dev-server proxy remains for standalone `npm start`.

## Impact

- **Removed:** `src/Aspire/FourDotnet.BoogaBooster.Aspire.AppHost/`, `src/Aspire/aspire.config.json`, `.claude/skills/csharp-aspire/` and its plugin copy.
- **Moved:** ServiceDefaults to `src/Shared/FourDotnet.BoogaBooster.ServiceDefaults/`. `BoogaBooster.slnx` and `FourDotnet.BoogaBooster.Api.csproj` are updated.
- **Added:** `infra.yaml`, an example parameter file, and a `.gitignore` entry for local parameter files.
- **Docs and tooling:** `CLAUDE.md`, `.github/copilot-instructions.md`, `README.md`, skills `csharp-observability`, `csharp-solution-structure`, `test-coverage`, agent `csharp-expert`, plugin (version bump), `.github/code-review/review-prompt.md`, `src/Shared/FourDotnet.BoogaBooster.IntegrationMessages/README.md`, comments in `IntegrationMessagingDefaults.cs`, `BoogaBoosterTelemetry.cs`, `proxy.conf.js` and `DomainInstrumentTests.cs`.
- **In-flight changes:** `workshop-prerequisites` (its AppHost-secrets and Dapr CLI checks), and the AppHost mentions in `backend-observation`, `ecs-foundation`, `job-system`, `park-economy` and `coaster-editor`.
- **Depends on:** `infra-template-tool` (a released `booga`) and `container-images-ci` (images in GHCR).
- **Review:** this change touches `CLAUDE.md`, `.claude/` and possibly `.github/`, so the CI code review flags it at `major` or higher for human review, as intended.
