## Purpose

Lists every machine prerequisite doctor verifies, what each check probes read-only, when it is required or optional, and what its failure tells the reader to do.

## ADDED Requirements

### Requirement: Runtime and toolchain checks

Doctor SHALL run these checks, each using only a version or info query:

- `dotnet-sdk`: runs `dotnet --version` with the repository root as working directory (so a `global.json` is honoured) and compares the major with the required major derived from the projects' `TargetFramework` (currently `net10.0`, so 10). Fails when `dotnet` is missing, when SDK resolution fails, or when the major is lower. Detail states found and required versions.
- `node`: compares the running Node version with `engines.node` of the Angular app's `package.json`; when no `engines` exists it falls back to `.nvmrc` or `.node-version`, and when none exists it is `skip` with detail "no Node requirement declared" and the drift test flags it. The range evaluator SHALL support `>=`, `>`, `<=`, `<`, `=`, `^`, `~`, `x` wildcards, space-separated AND and `||`; any other syntax fails the check with "unsupported range syntax in <file>".
- `npm`: runs `npm --version` and compares it with `packageManager` (`npm@<version>`): same major and installed version at least the declared one.
- `git`: runs `git --version`; required.
- `git-lfs`: required only when `.gitattributes` contains `filter=lfs`; otherwise `skip` with detail "not adopted (no filter=lfs in .gitattributes)". When required, runs `git lfs version`.
- `openspec-cli`: runs `openspec --version`; required, because every change is applied through it.

#### Scenario: SDK too old

- **WHEN** projects target `net10.0` and `dotnet --version` prints `9.0.304`
- **THEN** `dotnet-sdk` is `fail`, detail names `10` required and `9.0.304` found

#### Scenario: Node range satisfied

- **WHEN** `engines.node` is `>=22.0.0` and the running Node is 24.1.0
- **THEN** `node` passes

#### Scenario: Node range syntax unknown

- **WHEN** `engines.node` is `lts/*`
- **THEN** `node` fails with detail "unsupported range syntax in" followed by the file path

#### Scenario: git-lfs not adopted

- **WHEN** `.gitattributes` exists without `filter=lfs`
- **THEN** `git-lfs` is `skip` and does not affect the exit status, and adding `*.glb filter=lfs diff=lfs merge=lfs -text` to the fixture makes it required

### Requirement: Container and Dapr checks

`docker` SHALL be required when the AppHost source calls a container-backed resource method listed in `doctor.config.json` (currently including `AddRabbitMQ`); it runs `docker info --format "{{.ServerVersion}}"` (or `podman info` when `ASPIRE_CONTAINER_RUNTIME` is `podman`) and fails when the CLI is missing ("not found") or when the daemon is not reachable ("daemon not reachable"), with distinct hints for each. `dapr-cli` SHALL be required when the AppHost source or project files use Dapr; it runs `dapr --version`. `dapr-runtime` SHALL parse the same output and fail when the runtime version is reported as not installed (for example `n/a`), with the hint to initialise the runtime; it is `skip` with "depends on dapr-cli" when the CLI check failed.

#### Scenario: Daemon down

- **WHEN** `docker --version` succeeds but `docker info` exits non-zero
- **THEN** `docker` is `fail` with detail "daemon not reachable" and a hint that says to start the container engine, not to install it

#### Scenario: Dapr CLI present, runtime not initialised

- **WHEN** `dapr --version` reports a CLI version and a runtime version of `n/a`
- **THEN** `dapr-cli` passes and `dapr-runtime` fails with a hint naming the official initialisation step

### Requirement: Aspire AppHost secrets check

`aspire-secrets` SHALL read each parameter name declared with `AddParameter("<name>"` in the AppHost source (currently `rabbitmq-username` and `rabbitmq-password`) and verify that `Parameters:<name>` exists with a non-empty value in at least one of: the user-secrets file for the AppHost project's `UserSecretsId` (`%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json` on Windows, `~/.microsoft/usersecrets/<id>/secrets.json` elsewhere; the paths SHALL be confirmed against the official .NET documentation at apply time), the environment variable `Parameters__<name>`, or `appsettings.Development.json` of the AppHost. The check SHALL read the file directly, never run a state-changing command, and report only the names that are missing (`detail: "missing Parameters:rabbitmq-password"`). The hint SHALL show the `dotnet user-secrets --project <AppHost path> set Parameters:<name> <value>` form with the literal text `<value>` and no real value.

#### Scenario: One secret missing

- **WHEN** the secrets file holds `Parameters:rabbitmq-username` but not the password
- **THEN** the check fails, the detail names only `Parameters:rabbitmq-password`, and neither value appears anywhere in the output

#### Scenario: Fresh machine without secrets file

- **WHEN** the secrets file does not exist
- **THEN** the check fails listing both parameter names and the hint contains two `set` commands with `<value>` placeholders

#### Scenario: Environment variable accepted

- **WHEN** the file is absent but `Parameters__rabbitmq-username` and `Parameters__rabbitmq-password` are set and non-empty
- **THEN** the check passes

### Requirement: Authoring application checks

`blender` and `krita` SHALL probe, in order: an override variable (`DOCTOR_BLENDER_PATH`, `DOCTOR_KRITA_PATH`), the executable name on the PATH, then known install locations per OS (Windows: under the `Program Files` directories; macOS: `/Applications`; Linux: PATH, and a Flatpak listing for Krita and Blender). They run a version query (`--version`) on the found executable and report the version. Both are optional; each becomes required when an unchecked task of the selected change carries the matching tag or when named by `--require`. `playwright-browsers` SHALL be optional unless a package manifest depends on Playwright (derived) or a `[needs-playwright]` task is unchecked, and checks the Playwright browser cache directory (`PLAYWRIGHT_BROWSERS_PATH` or the per-OS default) for a Chromium folder, without launching a browser.

#### Scenario: Blender found in a standard Windows location

- **WHEN** the fake file system has `C:\Program Files\Blender Foundation\Blender 4.5\blender.exe` and the fake runner answers its `--version` query
- **THEN** `blender` passes with the reported version and no PATH entry is needed

#### Scenario: Required by task tags

- **WHEN** `--change asset-pipeline` is used and Blender is absent
- **THEN** `blender` fails and the hint contains the vendor download URL and the text "needed by tasks tagged [needs-blender]"

### Requirement: MCP servers are read from .mcp.json

For every server in root `.mcp.json` doctor SHALL emit `mcp:<name>`. For `type: "stdio"` it SHALL resolve `command` without executing it: absolute or relative paths by file existence, bare names by scanning the PATH (honouring `PATHEXT` on Windows). A server whose command is `npx` additionally requires the `npx` command to be present, and doctor SHALL NOT download the package. For `type: "http"` (or `sse`) it SHALL issue one request to `url` with a 5-second timeout and no credentials; any HTTP response, including 401, 403 and 405, counts as reachable, and a network error is a failure. With `--offline`, HTTP servers are `skip` with detail "offline". A `${VAR}` reference in `command`, `args`, `env` or `url` to an unset environment variable SHALL fail the check naming the variable but never printing any value. Servers named in `optionalMcpServers` of `doctor.config.json` (initially `blender` and `krita`, which need a desktop application on the developer machine) are optional and `skip` when unmet unless promoted by tags or `--require`. A resolved command proves installation, not that the server can reach its host application; doctor SHALL say so in the detail of passing optional servers ("command found; connection not tested"), and the smoke tests in `authoring-mcp-servers` cover connection.

#### Scenario: Style-guide executable absent

- **WHEN** `4dotnet-csharp-style-guide` is not on the PATH
- **THEN** `mcp:4dotnet-csharp-style-guide` fails with a hint that names the executable, states that `CLAUDE.md` makes consulting it mandatory for C# work, and points to `docs/prerequisites.md`

#### Scenario: npx-based server

- **WHEN** `primeng` runs through `npx` and `npx` resolves on the PATH
- **THEN** `mcp:primeng` passes without any network access

#### Scenario: HTTP server reachable but unauthorised

- **WHEN** the `microsoft-learn` URL answers with status 405
- **THEN** `mcp:microsoft-learn` passes with detail "reachable (HTTP 405)"

#### Scenario: Offline run

- **WHEN** `--offline` is given
- **THEN** `mcp:microsoft-learn` is `skip` with detail "offline" and the exit status is unaffected

#### Scenario: Required server not declared

- **WHEN** `--change asset-pipeline` makes `mcp:blender` required and `.mcp.json` has no `blender` server
- **THEN** `mcp:blender` is emitted as `fail` with detail "required by tasks but not declared in .mcp.json" and a hint pointing to the `authoring-mcp-servers` setup section of `docs/prerequisites.md`

#### Scenario: Windows executable extension

- **WHEN** the platform is `win32`, `PATHEXT` contains `.CMD` and the fake PATH has `npx.cmd`
- **THEN** command `npx` resolves

### Requirement: AI tooling sync status is included

`ai-tooling-sync` SHALL run `node <script>` where the script path comes from `doctor.config.json` (currently `.claude/skills/ai-tooling-sync/scripts/sync-ai-tooling.mjs`) without `--fix`, with the repository root as working directory, and map exit status 0 to `pass` and non-zero to `fail`, putting the first redacted output line in `detail` and "run the script with --fix, then review the remaining findings" in the hint. A missing script is `fail` with detail "script not found". It is required.

#### Scenario: Drift reported

- **WHEN** the script exits 1
- **THEN** `ai-tooling-sync` is `fail` and no file in the repository was modified by doctor

### Requirement: Check identifiers and groups are stable

Check ids SHALL be exactly: `dotnet-sdk`, `node`, `npm`, `git`, `git-lfs`, `openspec-cli`, `docker`, `dapr-cli`, `dapr-runtime`, `aspire-secrets`, `playwright-browsers`, `blender`, `krita`, `mcp:<server name>`, `ai-tooling-sync`, `deep:dotnet-build`, `deep:npm-ci`, `doctor-config`. Groups SHALL be `runtime`, `containers`, `secrets`, `authoring`, `mcp`, `repo` and `deep`. Later changes and the gate wording refer to these ids; renaming one SHALL require a spec change.

#### Scenario: Unknown id in tag mapping

- **WHEN** `doctor.config.json` maps a tag to an id that no check emits
- **THEN** `doctor-config` fails naming the tag and the id
