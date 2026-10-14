## Purpose

Defines `docs/prerequisites.md`, the single human- and agent-readable guide to preparing a machine, and the hint catalogue that keeps that guide and doctor's output identical.

## ADDED Requirements

### Requirement: A prerequisites guide covers every doctor check

`docs/prerequisites.md` SHALL contain: a short "Start here" section with the one command `node tools/doctor/doctor.mjs` (and `--change <name>` for a specific change); a table of every check id from `prerequisite-checks` with component, required version (stated as "derived from <file>" rather than a copied number wherever doctor derives it), required or optional, and the section that fixes it; one section per component with install steps for Windows, macOS and Linux each, a verification command, and the common failure with its cure; a "Reading doctor output" section (statuses, exit codes 0/1/2, `--json` shape, `--verbose`, `--deep`, `--offline`, `--change`, `--require`); a "Secrets" section showing how to set the AppHost parameters with placeholders only; an "MCP servers" section (what each declared server is for, how it resolves, the Blender and Krita setup and smoke-test recipes, and the risk record owned by `authoring-mcp-servers`); and a "Fresh machine, in order" checklist that an agent can follow top to bottom ending with a green doctor run.

#### Scenario: Every check has a section

- **WHEN** the documentation test lists the check ids doctor can emit (static ids plus the ids derived from `.mcp.json`)
- **THEN** each id appears in the guide's table and links to an existing heading in the same file

#### Scenario: Fresh machine walk-through

- **WHEN** a reader starts from a machine with only Node installed and follows the checklist
- **THEN** the checklist ends with `node tools/doctor/doctor.mjs --deep` expected to exit 0 and says what a skipped optional line means

### Requirement: Install guidance is sourced from official documentation and dated

Every install command and download URL in `docs/prerequisites.md` and the hint catalogue SHALL come from the tool's official documentation read at apply time, not from memory, and each component section SHALL record the source URL and the date it was checked. Version numbers in the guide SHALL only appear as a minimum derived from a repo file or as a quote of what the official page says on the check date. A component whose official page cannot be reached at apply time SHALL be marked "unverified" in the guide and in the catalogue rather than filled from memory.

#### Scenario: Source recorded

- **WHEN** the Docker section is read
- **THEN** it names the vendor documentation URL and a check date in `YYYY-MM-DD` form

#### Scenario: Unreachable source

- **WHEN** the Dapr documentation cannot be fetched during apply
- **THEN** the Dapr section and catalogue entry carry the word "unverified" and doctor's hint includes it

### Requirement: One hint catalogue feeds both doctor and the guide

Hints SHALL live in `tools/doctor/hints.json`, keyed by check id (or `mcp:default` for servers without an entry), each entry holding `component`, `docs` (official URL), `checked` (date) and per-OS `win32`, `darwin`, `linux` command or instruction strings, and optionally `note`. Doctor SHALL compose a failure hint from the catalogue entry for the current OS plus the derived minimum version; if the OS has no entry it SHALL print the `docs` URL. A documentation test SHALL fail when a catalogue entry's `docs` URL or any of its commands is missing from `docs/prerequisites.md`, or when a check id has no catalogue entry.

#### Scenario: Hint matches guide

- **WHEN** `hints.json` changes a Linux install command without the guide changing
- **THEN** the documentation test fails naming the check id

#### Scenario: Hint for the current OS

- **WHEN** a check fails on `darwin`
- **THEN** the hint text contains the `darwin` string of its catalogue entry and the official URL, and not the `win32` string

### Requirement: Prerequisites live in one place

After this change, `README.md` "Prerequisites" and `CLAUDE.md` SHALL point to `docs/prerequisites.md` and to doctor instead of restating tool versions, and SHALL keep no version number that doctor derives from a repo file. `.github/copilot-instructions.md` SHALL carry the same condensed pointer. The `ai-tooling-sync` check SHALL pass.

#### Scenario: No copied versions

- **WHEN** the README prerequisites section is read
- **THEN** it contains a link to `docs/prerequisites.md` and the doctor command, and no `.NET 10` or `npm 11` literal
