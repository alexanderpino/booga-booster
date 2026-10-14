## Why

The roadmap in `docs/roadmap.md` (changes 1 to 11) will be applied by agents on a different machine than the one that planned it. That machine has the MCP servers and more installed, but nothing in the repo proves which tools each change needs or whether they are present. Today the only statement is a README table (`README.md`, "Prerequisites") and scattered lines in `CLAUDE.md`; it cannot be executed, it already drifts (the README says Node 22+, `package.json` declares only `packageManager`), and a missing Docker daemon, uninitialised Dapr runtime, absent AppHost user-secrets, unreachable MCP server or missing Blender surfaces as a confusing failure ten tasks into an apply run. This change makes "is this machine ready?" a single command with an exact install hint for every red line, documents the install steps once, gates every later change on it, and takes over the Blender/Krita MCP server provisioning that `asset-pipeline` used to carry, because MCP servers are machine prerequisites rather than part of the asset pipeline.

## What Changes

- **`doctor` script** `tools/doctor/doctor.mjs`: dependency-free Node (ESM, `node:` built-ins only, no `npm install`), cross-platform (Windows, macOS, Linux). Read-only probes only; never installs, never modifies state, never prints secret values. Reports `pass` / `fail` / `skip` per check with the exact install hint; `--json` machine output, `--help`, quiet by default, non-zero exit on failure, `--change <name>` to scope required tools to one change, `--deep` for `dotnet build` and `npm ci` smoke checks.
- **Derived check list**: versions and requirements are read from repo files (csproj `TargetFramework`, `global.json` if present, `package.json` `engines`/`packageManager`, `.mcp.json`, `AppHost.cs` parameters and resources, `.gitattributes`, the `[needs-*]` tags in `openspec/changes/<name>/tasks.md`) instead of a hardcoded list; a test fails when a source file declares something doctor does not handle.
- **Checks**: .NET SDK, Node, npm, git (git-lfs only when `.gitattributes` adopts it), Docker daemon (or Podman when Aspire is told to use it), Dapr CLI and initialised runtime, `openspec` CLI, Aspire AppHost secrets (names only), Playwright browsers (optional), Blender and Krita (optional, required for `[needs-blender]` / `[needs-krita]` work), every MCP server declared in `.mcp.json`, the `ai-tooling-sync` exit status, and (behind `--deep`) `dotnet build` and `npm ci`.
- **`docs/prerequisites.md`**: per-component install steps for Windows, macOS and Linux, versions, how to run doctor, how to read its output, written for a human or an agent on a fresh machine. Install commands are taken from each tool's official documentation at apply time.
- **Doctor tests** with `node:test` and an injected command runner / fake environment (no real tool needed): pass, fail, skip, secret redaction, exit codes, JSON schema, derivation from fixture repos.
- **Gate**: `docs/roadmap.md` lists this change as change 0 first in the apply order; every other active change's task list starts with a group "0. Preconditions" whose first task is "run doctor; stop if red". The wording is fixed in this change's design and inserted into the other changes at apply time; `doctor --lint-gate` checks it.
- **MCP servers for asset authoring (moved from `asset-pipeline`)**: research and vetting of the official Blender and Krita MCP servers at apply time, their `.mcp.json` entries and `src/.mcp.json` mirror, the `CLAUDE.md` / `.github/copilot-instructions.md` / `README.md` obligations, the risk record, and the smoke tests. `asset-pipeline` keeps only the skill, prompts and art direction and declares a dependency on this change.
- **`package.json` `engines`** for the Angular app, so the Node requirement is derivable instead of living in README prose.

## Capabilities

### New Capabilities
- `environment-doctor`: The `doctor` CLI contract: statuses, output modes, exit codes, safety rules, secret redaction, derivation from repo files, testability.
- `prerequisite-checks`: The individual checks, what each probes, when it is required versus optional, and the install hint each failure carries.
- `prerequisites-documentation`: `docs/prerequisites.md` content and the hint catalogue shared by the document and doctor.
- `authoring-mcp-servers`: The `blender` and `krita` MCP servers: vetting, configuration, instruction-file obligations, risk record, smoke tests.
- `prerequisite-gate`: Apply order starting at change 0 and the "Preconditions" task group every later change carries.

### Modified Capabilities
<!-- None. No published capability changes behaviour; this change adds tooling around the repo, not product behaviour. -->

## Impact

- **New files**: `tools/doctor/` (CLI, `lib/`, `checks/`, `hints`, `doctor.config.json`, `test/`), `docs/prerequisites.md`.
- **Edited files**: `docs/roadmap.md` (planning edit now), `src/FourDotnet.BoogaBooster.App/package.json` (`engines`), root `.mcp.json` and `src/.mcp.json` (Blender/Krita servers), `CLAUDE.md`, `.github/copilot-instructions.md`, `README.md` (MCP rows, pointer to doctor and `docs/prerequisites.md`, prerequisites table replaced by a pointer), and the `tasks.md` of every other active change listed in `docs/roadmap.md` (Preconditions group).
- **Backend / frontend runtime**: none. No product code, no endpoint, no telemetry change.
- **Dependencies**: none on other changes. Provides the doctor gate and the `blender`/`krita` MCP servers to all later changes, notably `asset-pipeline` (11).
- **Governance**: touches `CLAUDE.md`, `.claude/`-adjacent tooling docs and MCP configuration, so the PR will be flagged major by the CI review by design; the `ai-tooling-sync` check must pass. `tools/doctor/` is not a skill, agent or command, so it needs no mirror.

> **Quality bar:** this change is for a workshop demo, not a production release. See `docs/demo-scope.md` for what may be deferred and what may not.
