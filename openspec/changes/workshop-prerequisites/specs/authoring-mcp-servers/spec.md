## Purpose

Provides the Blender and Krita MCP servers that asset authoring depends on as machine prerequisites: vetted at apply time, declared once for the whole team, documented, risk-assessed and smoke-tested. Moved here from the `asset-pipeline` change.

## ADDED Requirements

### Requirement: Blender and Krita MCP servers are configured for the whole team

The root `.mcp.json` SHALL contain servers named `blender` and `krita`, and `src/.mcp.json` SHALL contain the same two servers in its `servers` key, produced by the `ai-tooling-sync` mirror step rather than by hand. The exact server packages or executables, commands, arguments and any required host application add-on SHALL be identified at apply time from each server's official repository (not a mirror or registry listing) and recorded in `docs/prerequisites.md` with the source link and the date checked; they SHALL pass the currency check (not deprecated in the pinned version, official documentation names no different default, upstream not archived and released within the last 12 months). No package name SHALL be taken from memory. If no candidate passes for a server, that server SHALL NOT be added and the failure SHALL be recorded in this change's design decisions. The configuration SHALL contain no secrets, no absolute user paths and no auto-approval.

#### Scenario: Both files agree

- **WHEN** the sync check runs after the servers are added
- **THEN** `node .claude/skills/ai-tooling-sync/scripts/sync-ai-tooling.mjs` reports no errors
- **AND** `src/.mcp.json` lists `blender` and `krita` under `servers`

#### Scenario: Package identified from the official source

- **WHEN** the `blender` entry is committed
- **THEN** `docs/prerequisites.md` names the upstream repository URL for it and the date the currency check was done

#### Scenario: No candidate passes

- **WHEN** no Krita MCP server candidate passes the currency check
- **THEN** only `blender` is added, the failure is recorded in the design, the Krita-driven tasks of dependent changes stay blocked, and the manual Krita path is documented in `docs/prerequisites.md`

#### Scenario: Doctor sees the new servers

- **WHEN** the entries exist
- **THEN** `node tools/doctor/doctor.mjs --verbose` lists `mcp:blender` and `mcp:krita` without any change to doctor's code, as optional unless a `[needs-blender]`, `[needs-krita]` or `[needs-mcp]` task is unchecked in the selected change

### Requirement: Adding MCP servers carries documentation obligations

In the same change, `CLAUDE.md` (the "MCP servers available" table and a one-line pointer to `docs/prerequisites.md`), `.github/copilot-instructions.md` (the condensed mirror of the same) and `README.md` (the MCP server section) SHALL describe the new servers, state that they are local-only and optional (they need Blender or Krita running with an add-on on the developer machine), and state that CI review and headless runs do not use them. The instruction files SHALL NOT name any skill. The `ai-tooling-sync` check SHALL pass. The pull request description SHALL state that it touches `CLAUDE.md` and MCP configuration.

#### Scenario: Instruction files agree

- **WHEN** the change is complete
- **THEN** `CLAUDE.md` and `.github/copilot-instructions.md` both mention the `blender` and `krita` servers
- **AND** the sync check reports no error

#### Scenario: Teammate without Blender

- **WHEN** a developer without Blender opens the repository
- **THEN** `docs/prerequisites.md` states the Blender and Krita servers can be ignored, that a configured server whose application is absent may show a warning in the client, and that build, tests and `doctor` without `--change` still exit 0

### Requirement: MCP risk is assessed and recorded

Before enabling a server, the apply step SHALL record in `docs/prerequisites.md` whether the server can execute arbitrary code in the host application, access the network or download third-party assets, and which tools it exposes. Third-party downloads through any server SHALL be subject to the provenance and licence rules of the asset pipeline. The repository SHALL NOT pre-approve these tools in a shared permissions file.

#### Scenario: Code-execution capability

- **WHEN** a candidate server exposes a tool that runs arbitrary Python in Blender
- **THEN** the documentation lists that tool and states that agent instructions must prefer the server's structured tools and state each script they run

### Requirement: Servers are smoke-tested through their host applications

For each added server the apply step SHALL, when the host application is available, start the application with the add-on, connect, call one read-only tool and one create or draw call in a throwaway scene or document outside the repository, discard it, and record the working session recipe in `docs/prerequisites.md`. If the Krita server cannot paint, the guide SHALL document the manual Krita step instead. Smoke tests are tagged `[needs-blender]` and `[needs-krita]` and are not part of doctor, which only proves the command resolves.

#### Scenario: Blender recipe recorded

- **WHEN** the Blender smoke test passes
- **THEN** `docs/prerequisites.md` contains the start order (application, add-on, client) and the two tool calls used, and no file under `designs/` was changed

#### Scenario: Application unavailable

- **WHEN** Blender is not installed on the apply machine
- **THEN** the smoke-test task stays unchecked with the reason in the PR description, and doctor with `--change asset-pipeline` remains red for `blender`
