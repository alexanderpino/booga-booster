## Purpose

Defines the `doctor` command-line tool that tells a human or an agent, in one read-only command, whether the current machine can apply the repository's OpenSpec changes, and what exact step fixes each gap.

## ADDED Requirements

### Requirement: Doctor is a dependency-free cross-platform Node script

The repository SHALL contain `tools/doctor/doctor.mjs`, runnable as `node tools/doctor/doctor.mjs` from any working directory on Windows, macOS and Linux with only a Node.js installation (Node 20 or newer, because it uses the built-in `fetch` and `node:test`). It SHALL import only `node:` built-in modules and files under `tools/doctor/`; there SHALL be no `package.json` dependency, no `npm install` and no network fetch of code. The repository root SHALL be derived from the script location (`tools/doctor/../..`) unless `--root <path>` is given. The entry file SHALL check `process.versions.node` before loading any other module and, on Node older than 20, print one line naming the required Node version and the install hint, and exit with status 2.

#### Scenario: Runs from another directory

- **WHEN** doctor is started with the current directory set to `src/`
- **THEN** it resolves the repository root to the directory that contains `tools/doctor/` and reads files from there

#### Scenario: Node too old for doctor itself

- **WHEN** it is started under Node 18
- **THEN** it prints a single line stating that Node 20 or newer is required, with the Node install hint, and exits with status 2 without loading other modules

#### Scenario: No install step

- **WHEN** a fresh clone is checked out and no package manager command has been run
- **THEN** `node tools/doctor/doctor.mjs --help` succeeds

### Requirement: Every check reports pass, fail or skip with an install hint

Each check SHALL yield exactly one result with a status of `pass`, `fail` or `skip`, a stable `id` (for example `dotnet-sdk`, `mcp:primeng`), a `group`, a human `title`, a `required` flag, a one-line `detail` stating what was found or why the check did not apply, and a `hint`. `fail` SHALL mean a required check did not pass; `skip` SHALL mean the check is optional and unmet, or not applicable, or depends on a failed check, and SHALL carry the reason in `detail`. A `fail` and an unmet-optional `skip` SHALL carry a non-empty `hint` that names the component, the minimum version where one is derivable, an official documentation URL and, for the current OS, the install command from the hint catalogue. A `pass` SHALL carry `hint: null`. A check that throws, or whose probe exceeds its timeout, SHALL become a `fail` (required) or `skip` (optional) with `detail` saying so; one check failing SHALL NOT stop the others from running.

#### Scenario: Required tool missing

- **WHEN** the .NET SDK is required and `dotnet` is not found
- **THEN** the result is `fail` with a `detail` containing "not found" and a `hint` containing the SDK major version derived from the repo and an `https://` URL

#### Scenario: Optional tool missing

- **WHEN** Blender is not installed and no unchecked task needs it
- **THEN** the result is `skip`, `required` is false and `detail` says it is optional and which tagged tasks would need it

#### Scenario: Probe crashes

- **WHEN** a check's probe throws an unexpected error
- **THEN** that check is reported as `fail` (or `skip` when optional) with the error message redacted, and every remaining check still runs

#### Scenario: Timeout

- **WHEN** a probe does not finish within the timeout (default 10 seconds, configurable with `--timeout <seconds>`)
- **THEN** the child process is killed and the check is reported as timed out, not left hanging

### Requirement: Output is quiet by default, with a verbose and a JSON mode

Without options, doctor SHALL print only the failing checks (id, detail and the hint on the following line) and one final summary line stating overall result and counts; when nothing fails it SHALL print only the summary line. `--verbose` SHALL print every check with a `PASS`, `FAIL` or `SKIP` prefix in ASCII. `--json` SHALL print exactly one JSON document to standard output and nothing else (diagnostics go to standard error), shaped as: `schemaVersion` (integer, 1), `tool` ("doctor"), `platform` (`win32`, `darwin` or `linux`), `change` (string or null), `deep` (boolean), `ok` (boolean), `summary` (`pass`, `fail`, `skip` integers) and `checks` (array of objects with `id`, `group`, `title`, `status`, `required`, `detail`, `hint` (string or null) and `durationMs` (integer)). The schema SHALL be committed as `tools/doctor/doctor-report.schema.json`. Output SHALL contain no colour codes. `--help` SHALL list every option with one line each and exit 0.

#### Scenario: All green, quiet

- **WHEN** every required check passes and doctor runs without options
- **THEN** standard output is one line beginning `doctor: OK` with the pass and skip counts, and the exit status is 0

#### Scenario: Failures listed

- **WHEN** two required checks fail
- **THEN** standard output lists those two checks each followed by a `fix:` line, then a final line beginning `doctor: FAIL` with `2 failed`

#### Scenario: JSON is clean

- **WHEN** `--json` is used and a check writes a diagnostic
- **THEN** standard output parses as a single JSON document that validates against `doctor-report.schema.json`, and the diagnostic appears only on standard error

### Requirement: Exit status distinguishes red, usage errors and green

Doctor SHALL exit 0 when no check has status `fail` (skips do not count), 1 when at least one check failed, and 2 for usage errors (unknown option, missing option value, `--change` naming a change that does not exist, unreadable repo root) and for the Node-too-old case. An internal error in doctor's own code outside any check SHALL exit 2 and print a one-line message, never exit 0.

#### Scenario: Skips do not fail

- **WHEN** only optional checks are unmet
- **THEN** the exit status is 0

#### Scenario: Unknown change

- **WHEN** `--change does-not-exist` is given
- **THEN** doctor prints that no such change folder exists under `openspec/changes/` and exits 2

### Requirement: Doctor only reads and never prints secrets

Every probe SHALL be read-only: it SHALL run only version, info and list style commands, read files, check the PATH, or issue an HTTP request that does not send credentials, and SHALL NOT install, update, start, stop, initialise or configure anything. The sole exception is `--deep`, whose `dotnet build` writes only gitignored build output and whose `npm ci` runs as `--dry-run` (see the deep-smoke-checks requirement). Doctor SHALL never print a secret value: for user-secrets and environment variables it SHALL report only whether a named key exists and is non-empty. All text placed in `detail`, `hint`, human output or JSON SHALL pass through a redactor that removes (a) the value of every secret read during the run, (b) the value of every environment variable whose name matches `PASSWORD`, `SECRET`, `TOKEN`, `KEY` or `CREDENTIAL` (case-insensitive), and (c) text of the form `<name> = <value>` or `<name>: <value>` where the name matches those words, replacing each with `[redacted]`. Child-process output SHALL be limited to its first line, truncated to 200 characters, before redaction.

#### Scenario: Secret value never leaks

- **WHEN** the fake user-secrets store holds `Parameters:rabbitmq-password` with value `hunter2` and a probe's output happens to echo it
- **THEN** neither standard output, standard error nor the JSON document contains `hunter2`

#### Scenario: Unsafe command refused by construction

- **WHEN** a check asks the command runner to execute a program that is not in the allow-list of read-only invocations kept in `tools/doctor/lib/probes.mjs`
- **THEN** the runner throws before spawning anything, and a test asserts that every command used by every check is in that list

### Requirement: Check list is derived from repository files

Doctor SHALL derive what it checks from repository files wherever the file carries the information, so the check list cannot drift from the code: the required .NET SDK major from the highest `<TargetFramework>` (`netN.0`) across `**/*.csproj` (and `global.json` `sdk.version` when that file exists); Node and npm requirements from `engines.node` and `packageManager` in the package manifests found outside `node_modules`; the MCP servers from root `.mcp.json`; secret parameter names from `AddParameter("<name>"` calls in the AppHost source and the `UserSecretsId` from the AppHost project file; whether Docker and Dapr are required from the resource methods the AppHost source calls; whether git-lfs is required from `filter=lfs` in `.gitattributes`; whether Playwright browsers are required from a Playwright dependency in a package manifest; and which optional tools are required from the unchecked `[needs-*]` tags in the selected change's `tasks.md`. A small `tools/doctor/doctor.config.json` SHALL hold only what cannot be derived (the tag-to-check mapping, which MCP servers are optional, the container-resource method names, the `ai-tooling-sync` script path, the gate wording), and doctor SHALL report a `fail` check `doctor-config` when the config names an MCP server that is not in `.mcp.json`. Anything doctor finds in those files and cannot handle (an MCP server `type` it does not know, an unparsable version range) SHALL be reported as a `fail` that names the file, never silently ignored.

#### Scenario: SDK version follows the code

- **WHEN** the fixture repository's projects all target `net10.0` and the installed SDK is 9.0.100
- **THEN** `dotnet-sdk` fails with a detail naming 10 as required and 9.0.100 as found; changing the fixture projects to `net9.0` makes it pass without any edit to doctor

#### Scenario: New MCP server needs no doctor change

- **WHEN** a stdio server `foo` with `command: "foo-mcp"` is added to the fixture `.mcp.json`
- **THEN** the report contains a check `mcp:foo` and it fails with a hint when `foo-mcp` is not on the PATH

#### Scenario: Unknown declaration is loud

- **WHEN** the fixture `.mcp.json` has a server with `type: "carrier-pigeon"`
- **THEN** its check fails with a detail naming `.mcp.json`, the server and the unsupported type

#### Scenario: Drift guard against the real repository

- **WHEN** the doctor test suite runs against the real repository files with a fake command runner
- **THEN** the derived .NET major equals the highest `netN.0` found by an independent glob in the test, every server in `.mcp.json` has a result, and every `AddParameter` name in the AppHost source appears in the `aspire-secrets` detail

### Requirement: Doctor scopes requirements to a change with --change

`--change <name>` SHALL read the unchecked tasks (`- [ ]`) of `openspec/changes/<name>/tasks.md`, map each `[needs-*]` tag to checks through the config (`needs-blender` to `blender` and `mcp:blender`; `needs-krita` to `krita` and `mcp:krita`; `needs-playwright` to `playwright-browsers`; `needs-mcp` to every optional MCP server), and make those checks required for the run. `--require <id,id,...>` SHALL make named checks required without a change. Without either option, optional checks stay optional. Checked tasks (`- [x]`) SHALL NOT make anything required.

#### Scenario: Asset change needs Blender

- **WHEN** `--change asset-pipeline` is used, an unchecked task is tagged `[needs-blender]`, and Blender is absent
- **THEN** `blender` fails with `required: true`, and the same absence without `--change` is a `skip`

#### Scenario: Finished work no longer demands the tool

- **WHEN** every task tagged `[needs-blender]` is checked
- **THEN** `--change` no longer makes Blender required

### Requirement: Deep smoke checks run only on request

`--deep` SHALL add two checks after all others: `deep:dotnet-build` (run `dotnet build` on the solution file found under `src/` with quiet verbosity) and `deep:npm-ci` (run `npm ci --dry-run --ignore-scripts` in each package directory found, which verifies lockfile and manifest agree without deleting or writing `node_modules`). Each SHALL be `skip` when its prerequisite check (`dotnet-sdk`, `node`/`npm`) failed, with a longer timeout (default 600 seconds). Without `--deep` neither runs, and the human output of a run without `--deep` SHALL NOT mention them as failures.

#### Scenario: Deep without SDK

- **WHEN** `--deep` is used and `dotnet-sdk` failed
- **THEN** `deep:dotnet-build` is `skip` with detail "depends on dotnet-sdk"

#### Scenario: Lockfile drift

- **WHEN** `npm ci --dry-run` exits non-zero because `package-lock.json` and `package.json` disagree
- **THEN** `deep:npm-ci` fails with the first output line redacted and a hint to regenerate the lockfile in a separate commit

### Requirement: Doctor is tested without any real tool installed

All logic SHALL sit behind an injectable context (platform, environment variables, file system reads, command runner, HTTP fetch, PATH lookup, clock) so `node --test tools/doctor/test` passes on a machine with none of .NET, Docker, Dapr, Blender or Krita installed and without network access. The tests SHALL cover: pass, fail and skip for each check; optional-to-required promotion by `--change` and `--require`; secret redaction in human, verbose and JSON output; exit codes 0, 1 and 2; the JSON document validating against `doctor-report.schema.json` using a dependency-free validator in the tests; timeout handling; per-OS hints and per-OS path probing for Windows, macOS and Linux fixtures; derivation from fixture repositories including the unknown-declaration failures; the read-only command allow-list; and quiet, verbose and `--help` output.

#### Scenario: Suite is self-contained

- **WHEN** `node --test tools/doctor/test` runs on a clean machine without network
- **THEN** it passes and spawns no process other than the Node test runner itself

#### Scenario: Mutation sanity

- **WHEN** the redactor is replaced by an identity function in a scratch copy
- **THEN** the redaction tests fail
