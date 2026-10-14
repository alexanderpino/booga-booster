# Proposal

## Why

The local system is described by a C# Aspire AppHost, so changing the infrastructure means editing, compiling and understanding code. We want the infrastructure described as a plain, declarative YAML template that anyone can edit in Notepad, in the style of a CloudFormation template but without its infrastructure-as-code engine. Aspire stays the runtime underneath, and starting the system must not require the .NET SDK.

## What Changes

- Add a command-line tool, **`booga`**, that lives in this repository under `tools/booga/`.
- `booga start <template>` parses a YAML template, validates it, builds the Aspire application model **in memory** and runs it until it is stopped. It **does not generate code, write files, build projects or keep state**.
- Define a **template format** modelled on CloudFormation: `TemplateFormatVersion`, `Description`, `Metadata`, `Parameters`, `Resources` (`Type`, `Properties`, `DependsOn`) and `Outputs`. The only intrinsic functions are `Ref` and `Fn::GetAtt`. There are no `Conditions`, `Mappings`, string functions, `Transform`, macros or `Custom::` resources.
- Define a closed **type catalog**: `Type: <Provider>::<...>`, where the provider segment is a catalog namespace. The first release has `Docker::Container`, `RabbitMQ::Broker`, `Dapr::Component::PubSub` and `Dapr::Sidecar`. Each type publishes the attributes that `Fn::GetAtt` may read.
- Supply parameters CloudFormation-style: a JSON parameter file (`file://params.json`) or inline `ParameterKey=…,ParameterValue=…`. A `NoEcho` parameter without a value is prompted for interactively and is never printed.
- Add `booga validate <template>`, which reports file/line-addressed errors and has no side effects.
- Add `booga doctor <template>`, which derives the machine prerequisites from the types the template uses. A container runtime is required only when the template contains container-backed types.
- Distribute `booga` as a **self-contained** binary (no .NET SDK or runtime needed) on GitHub Releases and **winget**. npm and pip are `(deferred: demo scope)`.
- The `4dotnet-csharp-style-guide` and ADR-0008 do not apply to anything Aspire-related in this tool. This is a deliberate deviation. The repository's testing rules (xUnit v3, Moq, Bogus, no FluentAssertions, ≥ 80 % line coverage) still apply.

## Capabilities

### New Capabilities
- `infra-template-format`: the template's sections, value and reference semantics (`Ref`, `Fn::GetAtt`), parameters, the forbidden constructs, and validation error reporting.
- `infra-type-catalog`: the closed catalog of resource types in provider namespaces, each type's properties and published attributes, and what each type starts.
- `infra-cli`: the `booga start`, `validate` and `doctor` commands, their parameter inputs, outputs and exit codes, and the guarantee that nothing is generated, built or persisted.
- `infra-cli-distribution`: the self-contained binary, the supported platforms and the release channels.

### Modified Capabilities

None. The system's existing behaviour is not changed by adding the tool; `replace-apphost-with-template` moves the system onto it.

## Impact

- **New:** `tools/booga/` (CLI, template parser, catalog, model factory, tests, packaging).
- **New dependencies (tool only):** Aspire hosting packages for the in-memory model, the Aspire orchestrator (DCP) and dashboard binaries bundled per platform, and a YAML parser.
- **CI:** a release workflow that builds the per-platform binaries and publishes a GitHub Release and a winget manifest.
- **Repo rules:** a documented exception to the style guide and ADR-0008 for Aspire-related code. `CLAUDE.md` and its Copilot mirror are updated by `replace-apphost-with-template`, not here.
- **Not affected:** the existing modules, the AppHost (still the way to run the system until `replace-apphost-with-template` lands).
