# Tasks

## 1. Spikes (decide before building)

- [ ] 1.1 Check the licence terms of the Aspire orchestration (DCP) and dashboard packages for redistribution in a self-contained binary; record the finding in design.md D6 and verify the decision is either "bundle" or "first-run download"
- [ ] 1.2 Prove a `daprio/daprd` container next to an app container on Aspire's container network: publish and subscribe through RabbitMQ, with subscriptions discovered over the app channel; record the working flags and `AppProtocol` outcome in design.md D5
- [ ] 1.3 Prove a self-contained single-file app that builds an Aspire model in memory and runs it with DCP and dashboard paths pointing at bundled binaries, on a machine (container) without .NET; verify the dashboard opens

## 2. Scaffolding

- [ ] 2.1 Create `tools/booga/` with `Booga.slnx`, the `Booga.Cli`, `Booga.Templates` and `Booga.Catalog` projects and their test projects (xUnit v3, Moq, Bogus); verify `dotnet build tools/booga/Booga.slnx` and an empty `dotnet test` pass
- [ ] 2.2 Add `tools/booga/README.md` with usage and the recorded exception to the style guide and ADR-0008; verify the README's example command matches the CLI's help text

## 3. Template parsing and validation (`infra-template-format`)

- [ ] 3.1 Parse YAML into a position-carrying document model, rejecting YAML tags with the mapping-form hint; verify with tests for valid input, tag rejection and line/column reporting
- [ ] 3.2 Validate sections, `TemplateFormatVersion`, logical IDs and resource keys; verify each scenario in the spec has a test
- [ ] 3.3 Validate `Ref` and `Fn::GetAtt` targets with closest-match suggestions, and reject every computing construct (`Conditions`, `Mappings`, `Fn::Sub`, `Fn::If`, `Fn::Join`, `Fn::Select`, `Fn::Split`, `Fn::FindInMap`, `Transform`, `Custom::`) with the specific message; verify with tests per construct
- [ ] 3.4 Build the dependency graph from references and `DependsOn` and report cycles; verify with tests for implicit order, explicit order and a cycle
- [ ] 3.5 Validate parameters (`Type`, `AllowedValues`, `NoEcho`) and outputs (no secret outputs); verify with tests
- [ ] 3.6 Collect all diagnostics in one pass; verify a template with several errors reports them all with locations

## 4. Catalog (`infra-type-catalog`)

- [ ] 4.1 Define the catalog type contract (name, property schema, attributes, prerequisites, factory) and the provider-namespace registry with unknown-type and unknown-provider suggestions; verify with tests
- [ ] 4.2 Implement `Docker::Container` including runtime names and Aspire-convention connection environment variables; verify by inspecting the built (not run) Aspire model in tests
- [ ] 4.3 Implement `RabbitMQ::Broker` with credentials from parameters and the optional management port; verify via model-inspection tests
- [ ] 4.4 Implement `Dapr::Component::PubSub` (connection derived from the broker, component manifest delivered as container files) and `Dapr::Sidecar` (daprd container per design D5); verify via model-inspection tests and the unsupported-broker error
- [ ] 4.5 Verify line coverage of `Booga.Templates` and `Booga.Catalog` is at least 80 % with the `test-coverage` approach

## 5. Command line (`infra-cli`)

- [ ] 5.1 Implement `--parameters` (`file://` JSON and inline pairs, inline overrides file, undeclared-key suggestions) and the hidden prompt for missing `NoEcho` values, with the non-interactive failure; verify with tests
- [ ] 5.2 Implement `booga validate` with exit codes 0/1/2 and no side effects; verify with tests that assert no container runtime is contacted
- [ ] 5.3 Implement `booga start`: validate, resolve, build the model, run, print the dashboard URL and outputs, stop everything on interrupt; verify with the end-to-end suite (task 5.6)
- [ ] 5.4 Implement `booga doctor` with prerequisites derived from the template's types (container runtime only for container-backed types; never the .NET SDK); verify with tests using a faked runtime probe
- [ ] 5.5 Implement `booga types [<Type>]` listing types, properties and attributes; verify against the catalog in a test
- [ ] 5.6 Add the end-to-end suite (needs a container runtime): start a sample template with a broker, pub/sub, a sidecar and a container, check the outputs, interrupt, then assert all containers are gone and `git status` is clean
- [ ] 5.7 Add a test that no `NoEcho` value appears in any command output or error, including a failing resource start

## 6. Distribution (`infra-cli-distribution`)

- [ ] 6.1 Publish self-contained single-file binaries for `win-x64`, `win-arm64`, `linux-x64`, `linux-arm64` and `osx-arm64` with the bundled DCP and dashboard; verify `booga --version` prints both versions on each built binary in CI
- [ ] 6.2 Add a release workflow that builds on a tag, runs the tests, and publishes the archives with SHA-256 checksums to a GitHub Release; verify with a dry-run release on a test tag
- [ ] 6.3 Add the winget manifest under `tools/booga/packaging/winget/` and its generation step in the release workflow; verify with `winget validate` in CI
- [ ] 6.4 Record npm and pip packaging as `(deferred: demo scope)` in `tools/booga/README.md` and in the PR description

## Workflow follow-up

- Apply in a worktree and branch per the `git-change-workflow` skill; open a PR into `main`.
- `replace-apphost-with-template` depends on this change and on `container-images-ci`.
