# Design

## Context

Today the system starts from `src/Aspire/FourDotnet.BoogaBooster.Aspire.AppHost/AppHost.cs`, a compiled C# program that calls Aspire's builder API (`AddRabbitMQ`, `AddDaprComponent`, `AddProject`, `AddViteApp`). The Aspire SDK restores the orchestrator (DCP) and dashboard binaries from NuGet when the AppHost is built. Both of those therefore need the .NET SDK on the machine. See proposal.md for why that has to change.

This change only delivers the tool. Moving BoogaBooster onto it is `replace-apphost-with-template`, and the images the template will run come from `container-images-ci`.

## Goals / Non-Goals

**Goals:**
- A template is data only. All knowledge of how to wire resources lives in the catalog, not in the template.
- `booga` interprets the template at run time: no code generation, no build, no state.
- `booga` runs on a machine without .NET.
- The first catalog covers exactly what BoogaBooster needs: containers, RabbitMQ and Dapr pub/sub with sidecars.

**Non-Goals:**
- Deploying to a cloud, or any create/update/delete lifecycle (stacks, change sets, drift, rollback).
- `Azure::`, `Aws::`, `Oracle::` and other providers. The catalog is designed for them (see D4) but none ships in this change.
- Catalog types defined as YAML files by users. Possible later, out of scope now.
- Running projects from source (`dotnet run`) or Node dev servers. Applications arrive as images.
- npm and pip packages `(deferred: demo scope)`.

## Decisions

### D1. Interpret the template into an in-memory Aspire model

`booga` calls `DistributedApplication.CreateBuilder`, adds a resource for each template resource through a per-type factory, then `Build()` and `RunAsync()`.

```
template.yaml
   │ parse       YAML -> TemplateDocument (every node keeps file/line/column)
   ▼
validate         sections · logical IDs · types · properties · Ref/GetAtt · cycles · forbidden constructs
   │
   ▼ resolve     parameters: defaults <- file:// <- inline <- prompt (NoEcho)
   ▼ factory     topological order; ICatalogType.AddTo(builder, resource, resolved refs)
Aspire app model (in memory)
   │ run
   ▼
DCP + dashboard (bundled) -> containers
```

*Alternatives considered:* generating a file-based `apphost.cs` and running it (rejected: it generates code and needs the SDK); translating to Docker Compose (rejected: loses Aspire's dashboard, health-based start order and service-discovery conventions).

### D2. Parser keeps source positions; validation collects every error

The parser is YamlDotNet's representation model (not object deserialization), so every node keeps its `Mark` (line and column). Validation walks the whole document and returns a list of diagnostics `{file, line, column, code, message, suggestion?}`. Suggestions use edit distance against the known names (logical IDs, parameters, types, properties, attributes). YAML tags are detected on the node and rejected with the mapping-form hint.

### D3. Ref and Fn::GetAtt are the only intrinsics, resolved to typed references

A `Ref` to a parameter becomes its value (or an Aspire secret parameter for `NoEcho`). A `Ref` or `Fn::GetAtt` to a resource becomes a typed reference that the target's catalog type turns into an Aspire `WithReference`/`WaitFor` pair, an endpoint reference or an attribute value. Every recognised `Fn::*` other than `GetAtt` gets its own "templates do not compute values" error, so the message is specific rather than "unknown key".

### D4. Catalog is code inside the tool, keyed by provider namespace

Each type implements one interface: its `Type` name, a property schema (name, value kind, required, default), its published attributes, its prerequisites (for `doctor`), and `AddTo(builder, …)`. Types register under their provider namespace (`Docker`, `RabbitMQ`, `Dapr`). Adding a provider later, such as `Azure::Storage::Emulator` using Aspire's Azure hosting package, means adding classes to the catalog without changing the format.

The property schema also drives `booga types` output and future JSON Schema generation.

### D5. Dapr without the Dapr CLI: daprd as a container

The Community Toolkit's sidecar starts `daprd` as a host process installed by `dapr init`. `Dapr::Sidecar` instead adds a `daprio/daprd` container, pinned to a version, on the same container network as its app. Component manifests are put into the container with Aspire's container-files support rather than written to disk (see the "only starts things" requirement). The app channel points at the app container's network alias. `AppProtocol` maps to `--app-protocol`. The app reaches its sidecar through `DAPR_HTTP_ENDPOINT`/`DAPR_GRPC_ENDPOINT` environment variables pointing at the sidecar container.

*Alternative:* keeping the Toolkit sidecar (rejected: needs the Dapr CLI on the machine).

### D6. Bundle DCP and the dashboard; point Aspire at them

The release build references the platform-specific Aspire orchestration and dashboard packages for each RID and copies their binaries next to the executable. At start, `booga` sets Aspire's DCP CLI path and dashboard path configuration to those bundled locations before building the app. The bundled Aspire version is pinned to the repository's (13.6.x) and reported by `--version`.

*Alternative:* downloading DCP on first run (rejected: needs network access and a writable cache, and breaks the "no restore" requirement).

### D7. Self-contained single-file, not Native AOT

Aspire hosting uses reflection and is not trimming-safe, so `booga` is published `--self-contained` as a single file without trimming. The binary is large; that is accepted.

### D8. Location and build

```
tools/booga/
  Booga.slnx
  src/Booga.Cli/          command line (start · validate · doctor · types), exit codes
  src/Booga.Templates/    YAML parsing, document model, validation, parameter resolution
  src/Booga.Catalog/      catalog types and the Aspire model factory
  tests/Booga.Templates.Tests/
  tests/Booga.Catalog.Tests/
  tests/Booga.Cli.Tests/
  packaging/winget/
```

`tools/booga` has its own solution so the BoogaBooster solution build is unaffected. Its tests follow the repository rules (xUnit v3, Moq, Bogus, no FluentAssertions, ≥ 80 % line coverage on `Booga.Templates` and `Booga.Catalog`). The `4dotnet-csharp-style-guide` and ADR-0008 are deliberately not applied to it. A short README in `tools/booga/` records that exception.

### D9. Tests that need Docker are separated

Parser, validation, parameter resolution and the factory are tested without a runtime: the factory test inspects the built Aspire model (resources, annotations, references) without running it. A small end-to-end suite starts a template with a real container runtime and is marked so it runs in CI and is skipped locally when no runtime is reachable.

## Risks / Trade-offs

- [DCP's redistribution terms may not allow bundling] → check the licence of the orchestration and dashboard packages first (task 1.1). If bundling is not allowed, fall back to a first-run download into the user's cache and record that the "no network at start" requirement holds only after the first run.
- [Configuration keys for the DCP and dashboard paths are not a stable public API] → pin the Aspire version, cover the path setup with an end-to-end test, and treat Aspire upgrades as deliberate changes.
- [daprd in a container may not discover subscriptions over HTTPS with the development certificate] → spike in task 1.2. The fallback is `AppProtocol: http` on the container port. HTTPS inside the container network adds nothing locally.
- [Binary size, an estimated hundreds of MB per platform] → accepted; publish compressed archives.
- [Podman and Docker differ in networking] → the end-to-end suite runs on Docker. Podman is best effort and is listed as such in `doctor`'s hint.
- [Spelling of property names (PascalCase) versus Aspire's API names] → the catalog defines the template's names. They are not mirrored from Aspire's method parameters, so Aspire renames do not leak into templates.

## Migration Plan

Additive. Nothing in the repository uses `booga` until `replace-apphost-with-template` lands. Rollback is deleting `tools/booga/` and the release workflow.
