## Context

Prerequisites today are prose: the README "Prerequisites" table (.NET 10 SDK, Node 22+ / npm 11, Docker, Dapr CLI initialised, `4dotnet-csharp-style-guide` on PATH) and a few lines in `CLAUDE.md` (AppHost user-secrets `Parameters:rabbitmq-username` / `-password`, Docker, `dapr init`). Facts in the repo that an executable check can use: all projects target `net10.0` (`src/**/*.csproj`), there is no `global.json`; the Angular app's `package.json` sets `packageManager: npm@11.5.2` and no `engines` (so "Node 22+" is README-only); root `.mcp.json` declares `4dotnet-csharp-style-guide` (stdio executable), `primeng` (stdio via `npx -y @primeng/mcp`) and `microsoft-learn` (http); `src/Aspire/FourDotnet.BoogaBooster.Aspire.AppHost/AppHost.cs` calls `AddParameter("rabbitmq-username"|"rabbitmq-password", secret: true)`, `AddRabbitMQ`, `AddDaprComponent` and `.WithDaprSidecar`, and its project file has a `UserSecretsId`; `.claude/skills/ai-tooling-sync/scripts/sync-ai-tooling.mjs` is a read-only check unless `--fix` is passed; `.github/code-review/*.mjs` already set the precedent of dependency-free Node scripts tested with `node --test`. The tasks vocabulary `[no-tools]`, `[needs-blender]`, `[needs-krita]`, `[needs-playwright]`, `[needs-mcp]` is introduced by `asset-pipeline` and reused here. OpenSpec changes will be applied on another machine that has all MCP servers and more installed; this change must make that machine's readiness provable before change 1 starts. Shared brief rules (module anatomy, xUnit, Angular) do not apply: no product code is touched, no C# is written, so the style-guide MCP step does not apply.

## Goals / Non-Goals

**Goals:**
- One command that answers "can this machine apply the roadmap?" with exact, OS-specific fixes, usable by a human or an agent, with machine-readable output.
- A check list that follows the repo's own files, so it does not rot.
- A gate in the roadmap and in every change so that apply never starts on a red machine.
- Doctor fully testable on a machine that has none of the tools.
- Take the Blender/Krita MCP server provisioning out of `asset-pipeline` and into the place that owns prerequisites.

**Non-Goals:**
- Installing, updating or configuring anything (doctor reports; humans or an explicitly instructed agent act).
- Proving MCP servers can talk to Blender or Krita (smoke tests, not doctor).
- Verifying cloud accounts, tokens or CI secrets (`COPILOT_GITHUB_TOKEN` is a CI concern).
- Containers for the dev environment, version managers, or a bootstrap installer.
- Editing product code, CI workflows or `.claude/` skills.

## Decisions

### D1 - Plain Node ESM script at `tools/doctor/doctor.mjs`

Why Node: the machine already needs Node for the Angular app, `openspec` and the existing `.mjs` tooling; a single script runs the same on Windows, macOS and Linux and needs no install step (only `node:` built-ins: `node:child_process`, `node:fs`, `node:path`, `node:os`, `node:url`, `node:test` for tests, the global `fetch`). Why `tools/doctor/` at the repo root and not under the app: it spans both stacks and must run before `npm ci`, and the app's `tools/assets/` (from `asset-pipeline`) is frontend-scoped. The entry file `doctor.mjs` only checks `process.versions.node` (floor 20, because of global `fetch` and `node:test`; the app's `engines` is stricter and is reported by the `node` check) and then `import()`s `lib/main.mjs`, so a too-old Node produces a clear message instead of a syntax error.
Rejected: PowerShell plus a bash twin (two codebases to keep equal, drift is the problem being solved); a `dotnet tool` (needs the SDK it is supposed to find missing); an npm package or `npx` (needs install and network); Python (not a repo prerequisite).

### D2 - Layering for testability: context in, results out

`lib/context.mjs` builds the real context `{ platform, env, cwd, root, now, fs: { readText, readDir, exists }, which(name), run(program, args, { cwd, timeoutMs }), fetch(url, { timeoutMs }) }`; tests build a fake from in-memory maps. Every check is a pure async function `(ctx, derived, options) -> { status, detail, hint? }`; `lib/main.mjs` runs them, wraps each in timeout and try/catch, applies promotion (`--change`, `--require`), redacts, sorts into the report and chooses the exit status. Module layout: `lib/args.mjs`, `lib/semver.mjs`, `lib/redact.mjs`, `lib/probes.mjs` (allow-list and runner), `lib/derive/{dotnet,node,mcp,apphost,tags,lfs}.mjs`, `lib/checks/{runtime,containers,secrets,authoring,mcp,repo,deep}.mjs`, `lib/hints.mjs`, `lib/report.mjs`, `lib/validate-report.mjs`, `lib/gate.mjs`. Files stay small so each can be tested in isolation.

### D3 - Read-only by construction: an allow-list of invocations

`lib/probes.mjs` exports the only table of commands doctor may run, as `{ program, argv-pattern }` entries (for example `dotnet --version`, `dotnet --list-sdks`, `node --version`, `npm --version`, `git --version`, `git lfs version`, `docker info --format ...`, `podman info`, `dapr --version`, `openspec --version`, `blender --version`, `krita --version`, `flatpak list`, `node <ai-tooling-sync script>`, and in `--deep` only `dotnet build ...` and `npm ci --dry-run --ignore-scripts`). `ctx.run` refuses anything else before spawning, with `shell: false` and an argument array (no shell interpolation, so a path with spaces or a hostile `.mcp.json` value cannot inject). A test asserts every command string used by any check is in the table. The `--deep` exception is explicit: `dotnet build` writes `bin/` and `obj/`, which are gitignored build output; `npm ci` would delete `node_modules`, so only `--dry-run` is used, which validates lockfile against manifest. Alternative considered: full `npm ci` (rejected: destroys an existing install, violates read-only); documented limitation: a dry run does not prove the registry is reachable or that install scripts work.

### D4 - Secrets: read keys, never values; redact everything on the way out

The secrets check reads the user-secrets JSON directly rather than running `dotnet user-secrets list`, which prints values. Only key presence and non-emptiness leave the check. The redactor (`lib/redact.mjs`) is the last step before any string reaches stdout, stderr or JSON, and it is seeded with every secret value the run read plus values of environment variables with secret-looking names. Child-process output is cut to its first line and 200 characters before redaction because tools sometimes echo configuration. The `${VAR}` expansion check for MCP entries records only the variable name.

### D5 - Status model: pass / fail / skip, with optional-to-required promotion

A tool can be required (red when missing) or optional (grey). Requirement is computed, not configured per machine: derived defaults (for example Docker is required because AppHost uses a container resource), then `--change <name>` promotion from unchecked `[needs-*]` tags, then `--require`. This is what makes a laptop without Blender green for `ecs-foundation` and red for `asset-pipeline`. A dependent check whose prerequisite failed is `skip` ("depends on X") so one root cause does not produce a cascade of failures, while the root cause still fails the run. `skip` never affects exit status.

### D6 - Derivation rules and what stays in config

Derive: .NET major from the highest `netN.0` in `**/*.csproj` (honouring `global.json` if added later); Node from `engines.node` (fallback `.nvmrc`/`.node-version`); npm from `packageManager`; MCP servers from `.mcp.json`; secret names and `UserSecretsId` from the AppHost; Docker/Dapr need from the AppHost source; LFS from `.gitattributes`; Playwright from package manifests; tool requirements from task tags. Config (`tools/doctor/doctor.config.json`) holds only non-derivable facts: tag to check ids, optional MCP servers (`blender`, `krita`), container-resource method names, the sync script path, `gateExempt`. Unknown declarations fail loudly (unknown MCP `type`, unparseable range). A drift test reads the real repo with a fake runner and compares doctor's derived values with independent globs. The Node requirement has no machine-readable source today, so task 6.1 adds `engines.node` to the app `package.json`, with the range taken from the pinned `@angular/cli` package's own `engines` at apply time (never from memory, never from the README).

### D7 - Semver subset, hand-written

Only what the repo's manifests use is supported: comparators `>= > <= < =`, `^`, `~`, `x`/`*` wildcards, AND by space, OR by `||`. Anything else fails with "unsupported range syntax" rather than guessing. No third-party semver package (zero-install rule). Alternative: shell out to `npm`'s semver (rejected: depends on npm internals).

### D8 - Tool discovery without executing unknown things

PATH lookup (`which`) scans `PATH` entries and `PATHEXT` on Windows and never runs the candidate. Version queries only run for tools doctor knows. Blender and Krita add known install locations per OS (Windows `Program Files`, macOS `/Applications`, Linux PATH plus Flatpak) and an override variable; exact folder patterns are confirmed against the vendors' install documentation at apply time. MCP stdio commands are resolved but not started: starting a server could spawn a desktop bridge or contact a network; connection is the smoke test's job (`authoring-mcp-servers`).

### D9 - Hints are data, shared with the guide

`tools/doctor/hints.json` is the single source of per-OS install text and official URLs; `docs/prerequisites.md` repeats them for people, and a documentation test fails when the two diverge. Doctor never invents a command: if the catalogue has none for the OS it prints the official URL. All catalogue content is gathered at apply time from official documentation (task group 5), dated, and marked "unverified" when a page cannot be reached. This design deliberately names no package-manager command, package id or version.

### D10 - Gate by convention plus lint

The gate cannot be enforced by OpenSpec itself, so it is a task-list convention with a linter: group `## 0. Preconditions` with a fixed first task (wording in `prerequisite-gate`). The linter (`doctor --lint-gate`) reads the roadmap to know which changes are in scope, so a new change added to the roadmap without the group goes red immediately. Alternative: a git hook or CI job (rejected for now: editing `.github/workflows/` draws the major-severity review and CI machines are not where agents apply changes; stated as a follow-up in Open Questions). Wording for later changes, verbatim, is in the `prerequisite-gate` spec; the apply step of this change inserts it into the other changes (they were written before this one existed and this planning pass may not edit them).

Wording per change, for reference:

```
## 0. Preconditions

- [ ] 0.1 Run `node tools/doctor/doctor.mjs --change <change-name>` from the repository root. If it exits non-zero, STOP: report the failing checks and their hints, install nothing without being asked, and do not start group 1. Verify: exit status 0.
```

### D11 - MCP servers: identify at apply time, never assume (moved from asset-pipeline D10)

No package, command or version for the `blender` and `krita` servers is named here because none is verified. The research task finds each server's official repository (not a mirror or a registry listing), reads its install instructions, tool list and release history, applies the currency check, and records URL, date, command and arguments in `docs/prerequisites.md`. Only then are the entries added to root `.mcp.json`, and `src/.mcp.json` is regenerated with the sync script's `--fix` (the mirror uses `servers` instead of `mcpServers`; the sync script owns that translation). The servers drive desktop applications over a local socket or similar, so they are local and optional: teammates without them lose nothing, but a configured server that cannot start may show a warning in the client, which the guide states. If a server exposes arbitrary-code execution in Blender (to be verified from its documentation), that is recorded as a risk. No permission pre-approval is committed. Obligations from `CLAUDE.md` "Rules when adding tooling": root `.mcp.json` only (never machine-local); `CLAUDE.md` MCP table and `.github/copilot-instructions.md` condensed mirror updated together; `README.md` MCP sections updated; any agent `tools:` list using the servers must list both vocabularies and the project server names (no agent is added here, so this applies only if apply decides otherwise); then the sync script until clean. Instruction files name no skill; they link to `docs/prerequisites.md`.

### D12 - Currency check for what this change introduces

No third-party library is introduced (D1, D7). Node built-ins used: `node:test` runner, global `fetch`, `node:child_process` `spawn`/`execFile` with `shell: false`; at apply time the Node documentation for the version in use is read to confirm none is deprecated or experimental in that version (`node --test` flags, `fetch` timeout via `AbortSignal.timeout`), and the Node 20 floor in D1 is re-checked against the Node release schedule on nodejs.org (a floor whose line is end-of-life is raised to the oldest supported line, and spec scenarios are updated in the same commit). Findings are recorded here under D12 at apply time.

## Dependencies and contracts

**Requires from earlier changes:** none. Uses existing infrastructure only: root `.mcp.json` and the `src/.mcp.json` mirror rules, `ai-tooling-sync` (invoked, not modified), the OpenSpec CLI, the `[needs-*]` tag vocabulary.

**Provides to later changes:**
- The command `node tools/doctor/doctor.mjs [--change <name>] [--require ids] [--json] [--verbose] [--deep] [--offline] [--timeout s] [--root path] [--lint-gate] [--help]` and exit statuses 0/1/2.
- The JSON report (`doctor-report.schema.json`, `schemaVersion: 1`) and the stable check ids in `prerequisite-checks`.
- The `## 0. Preconditions` task group wording (D10) that every later change carries.
- `docs/prerequisites.md` as the install reference, and `tools/doctor/hints.json`.
- To `asset-pipeline` (11): `blender` and `krita` MCP servers declared in `.mcp.json`, their setup/smoke-test/risk records in `docs/prerequisites.md`, and `doctor --change asset-pipeline` as its gate. `asset-pipeline` no longer touches `.mcp.json`, `CLAUDE.md`, `.github/copilot-instructions.md` or `README.md` for MCP purposes.
- To every change: the convention that new tools they need are declared where doctor derives from (a `package.json`, a csproj, `.mcp.json`, a task tag), so doctor needs no edit.

## Risks / Trade-offs

- [Install hints go stale as vendors change their docs] -> hints are data with dates; `checked` dates in the catalogue; documentation test keeps guide and catalogue equal; re-check is a one-hour task, noted in the guide.
- [The install source of the `4dotnet-csharp-style-guide` executable is not documented in this repo] -> the hint states the executable name and `CLAUDE.md`'s rule, and the apply step asks the repository owners for the internal install location; entry stays "unverified" until answered (Open Questions).
- [Heuristic derivation misreads a file (regex on `AppHost.cs`, `csproj`)] -> parsing is deliberately narrow (`AddParameter("..."`, `<TargetFramework>`), unknown shapes fail loudly, and the drift test compares against independent globs on the real repo.
- [PATH-only detection misses Blender or Krita installed in odd places] -> override environment variable, known locations per OS, and a hint saying how to set the override.
- [A passing MCP check only proves the command resolves] -> stated in the detail text and in the guide; smoke tests cover the connection.
- [`--deep` is not read-only] -> opt-in, writes only gitignored build output, `npm ci` is a dry run; stated in `--help`.
- [Another machine has a different shell or locale, so parsing tool output is fragile] -> only first lines of version queries are parsed with tolerant regexes; locale-dependent output (`dapr --version` layout, `docker info` text) is parsed from structured flags where the tool offers them, otherwise a failed parse is `fail` with the raw first line shown (redacted).
- [Gate lint scope depends on parsing `docs/roadmap.md` headings] -> the heading format `### N name` is pinned by a test; a roadmap that does not parse makes the lint exit 2, not 0.
- [Editing other changes' `tasks.md` at apply could conflict with their authors] -> single commit that only adds the group; listed in the PR; the gate lint tells which changes remain.
- [Conflict with shared brief decisions] -> none; this change is tooling only and touches no simulation, physics or UI decision.

## Migration Plan

Additive. Order: doctor core and tests (no repo-state dependence); checks; CLI; hints and guide; `engines` and pointer edits; MCP servers; gate insertion; verification. Rollback of any group is a revert of its commit; removing `tools/doctor/` and the Preconditions groups restores today's behaviour. The `asset-pipeline` change was already edited in the planning pass so that it does not duplicate group 7's work.

## Open Questions

- Where does the `4dotnet-csharp-style-guide` executable come from (internal feed, repository, installer)? Needed for an exact hint; ask the repo owners at apply.
- Should the gate also run in CI (`.github/workflows/`) for pull requests? Default: no in this change (major-severity review, maintainers' decision); `--lint-gate` is ready to be wired in.
- Which Podman/Docker variants should the container check accept beyond `ASPIRE_CONTAINER_RUNTIME=podman`? Default: only those two.
- Does the Dapr CLI print the runtime version in a stable, parseable way across versions? Verify against the installed CLI and its documentation at apply; if not, fall back to checking for the Dapr runtime containers with a read-only `docker ps` filter and record the decision under D12.
