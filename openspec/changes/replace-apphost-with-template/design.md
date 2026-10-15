# Design

## Context

`src/Aspire/FourDotnet.BoogaBooster.Aspire.AppHost/AppHost.cs` currently declares four things: RabbitMQ (parameter-backed credentials from the AppHost's user-secrets, port 5672, management on 15672), a programmatic Dapr `pubsub.rabbitmq` component, the API project with a Toolkit Dapr sidecar (`AppProtocol = "https"`), and the frontend as a Vite app. `ServiceDefaults` is referenced by the API only. Many documents, skills and in-flight changes describe the AppHost as the entry point (see proposal.md, Impact).

This change depends on `infra-template-tool` (types `Docker::Container`, `RabbitMQ::Broker`, `Dapr::Component::PubSub`, `Dapr::Sidecar`) and on `container-images-ci` (images listening on HTTP 8080, frontend `/api` proxy driven by `services__fourdotnet-boogabooster-api__http__0`).

## Goals / Non-Goals

**Goals:**
- Run the same system as today from `infra.yaml`, with the same resource names and broker settings.
- Leave no rule, skill or document that still requires an AppHost or applies ADR-0008.

**Non-Goals:**
- Changing application behaviour or code beyond moving ServiceDefaults.
- A source-level debug experience (F5 into the API through Aspire). Developers debug the API with `dotnet run` against a template-started broker, or build a local `dev` image (D4).
- Changing the style guide outside this repository.

## Decisions

### D1. The template

```yaml
TemplateFormatVersion: "2026-10-14"
Description: BoogaBooster - digital twin of the Octopus carnival ride, run locally

Parameters:
  RabbitMqUsername:
    Type: String
    Default: guest
  RabbitMqPassword:
    Type: String
    NoEcho: true
    Description: Password for the local RabbitMQ broker
  ImageTag:
    Type: String
    Default: main
    Description: Tag of the booga-booster-api and booga-booster-frontend images

Resources:
  Messaging:
    Type: RabbitMQ::Broker
    Properties:
      Name: messaging
      Username: { Ref: RabbitMqUsername }
      Password: { Ref: RabbitMqPassword }
      Port: 5672
      ManagementPort: 15672

  PubSub:
    Type: Dapr::Component::PubSub
    Properties:
      ComponentName: pubsub              # = IntegrationMessagingDefaults.PubSubName
      Broker: { Ref: Messaging }

  Api:
    Type: Docker::Container
    Properties:
      Name: fourdotnet-boogabooster-api  # frontend discovers the API by this name
      Image: ghcr.io/alexanderpino/booga-booster-api
      Tag: { Ref: ImageTag }
      Ports: [ 8080 ]
      Environment:
        ASPNETCORE_ENVIRONMENT: Development
      Connections:
        - { Ref: Messaging }

  ApiSidecar:
    Type: Dapr::Sidecar
    Properties:
      App: { Ref: Api }
      AppProtocol: http                  # TLS adds nothing inside the local container network
      AppPort: 8080
      Components:
        - { Ref: PubSub }

  Frontend:
    Type: Docker::Container
    Properties:
      Name: frontend
      Image: ghcr.io/alexanderpino/booga-booster-frontend
      Tag: { Ref: ImageTag }
      Ports: [ 8080 ]
      Public: true
      Connections:
        - { Ref: Api }

Outputs:
  FrontendUrl:
    Description: Operator dashboard
    Value: { Fn::GetAtt: [Frontend, Url] }
  RabbitMqManagementUrl:
    Description: RabbitMQ management UI
    Value: { Fn::GetAtt: [Messaging, ManagementUrl] }
```

`AppProtocol` changes from `https` to `http`. The AppHost needed HTTPS only because the API ran with its development certificate and HTTPS redirection on the host. In the container the API listens on HTTP and does not redirect (see `container-images-ci` design D1). The exact property names follow `infra-template-tool`'s catalog as delivered; this listing is the intended shape.

### D2. Credentials move from user-secrets to parameters

`RabbitMqPassword` has no default, so `booga` prompts for it. To avoid typing it every time, a developer copies `infra.parameters.example.json` to `infra.parameters.local.json` (git-ignored via `*.parameters.local.json`) and runs `booga start infra.yaml --parameters file://infra.parameters.local.json`. The example file holds no real secret.

*Alternative:* `Default: guest` for the password. Rejected: it would make `NoEcho` meaningless and teach a bad habit, even for a local broker.

### D3. ServiceDefaults moves to `src/Shared/` unchanged

The project becomes `src/Shared/FourDotnet.BoogaBooster.ServiceDefaults/FourDotnet.BoogaBooster.ServiceDefaults.csproj`. Its code and namespace stay as they are, so `Program.cs` keeps calling `AddServiceDefaults()`/`MapDefaultEndpoints()` unchanged. Only the solution file and the API's project reference change. Aspire still injects the OTLP endpoint into containers, so the dashboard keeps showing the API's traces and metrics, and the observability rule ("OTEL configured once, in ServiceDefaults") stays true.

### D4. Developer loop without an AppHost

- **Run the system:** `booga start infra.yaml`.
- **Try local API or frontend changes:** build images locally under the GHCR name with tag `dev` (`dotnet publish … -t:PublishContainer` and `docker build`, documented in `docs/container-images.md`), then start with `ImageTag=dev`.
- **Debug the API in an IDE:** start the template, stop the API container in the dashboard, and run the API with `dotnet run`. Pub/sub then needs a host-side sidecar, which is out of scope. Documented as a known limitation.

### D5. Remove the Aspire rules rather than rewrite them

- Delete `.claude/skills/csharp-aspire/` and `plugins/4dotnet-boogabooster/skills/csharp-aspire/`. Bump the plugin version (minor: a skill is removed) and drop "Aspire" from its description.
- In `csharp-observability`, `csharp-solution-structure`, `test-coverage` and the `csharp-expert` agent (both copies), replace AppHost/ServiceDefaults references with the new ServiceDefaults path and drop ADR-0008.
- In `CLAUDE.md` and `.github/copilot-instructions.md`: rewrite the Aspire bullets, the "Common commands" and the user-secrets block around `booga start infra.yaml`. Add an explicit statement that the `4dotnet-csharp-style-guide` and ADR-0008 do not apply to how the system is composed and run. Remove `csharp-aspire` from the skill list.
- In `.github/code-review/review-prompt.md`, replace AppHost wording so the reviewer does not demand an AppHost. Changing the prompt is itself a `major` finding by design.
- Run `node .claude/skills/ai-tooling-sync/scripts/sync-ai-tooling.mjs` (with `--fix` for mechanical mirrors) and require it to report no errors.

### D6. Record the decision as an ADR

Add `docs/adr/0010-declarative-system-template.md` (status Accepted), which supersedes ADR-0008 for this repository and links the three changes. It makes the deviation from the style guide explicit for future readers and reviewers.

### D7. In-flight changes are updated, not left stale

`workshop-prerequisites` replaces its "Aspire AppHost secrets check" and Dapr CLI/`dapr init` checks with "`booga` installed" and "container runtime reachable" checks. Its environment doctor may call `booga doctor infra.yaml` for the template-derived part. The other in-flight changes get their AppHost mentions reworded to the template. These are text edits in their artifacts; none of them is applied by this change.

## Risks / Trade-offs

- [Developers lose F5 debugging and hot reload] → documented loop (D4). This is accepted because the user prioritised "no SDK needed".
- [Template property names differ from the delivered catalog] → task 1.1 validates the template against the released `booga` before anything is removed.
- [Removing the AppHost before the template works leaves no way to run the system] → order the tasks so the template is proven end to end (group 1) before anything is deleted (group 3).
- [Stale AppHost references remain somewhere] → task 6.1 greps the repository for `AppHost`, `ADR-0008`, `csharp-aspire` and `user-secrets` and allows hits only in archived changes and the new ADR.
- [The PR touches `CLAUDE.md`, `.claude/` and `.github/`] → expected `major` review findings; a human must approve.

## Migration Plan

1. Prove `infra.yaml` end to end against the released `booga` and the published images, while the AppHost still exists.
2. Move ServiceDefaults; build and test the solution.
3. Delete the AppHost and `aspire.config.json`; build and test the solution.
4. Update docs, skills, plugin, review prompt and in-flight changes; run the sync check.

Rollback: revert the PR. The AppHost and its user-secrets ID return unchanged.
