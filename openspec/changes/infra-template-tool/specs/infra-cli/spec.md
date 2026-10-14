# Spec Delta

## Purpose

Defines the `booga` command line: starting a system from a template, validating a template, checking machine prerequisites, and the guarantee that the tool only starts things and never generates, builds or persists anything.

## ADDED Requirements

### Requirement: start runs the system described by a template

`booga start <template>` SHALL validate the template, resolve its parameters, start every resource in dependency order with the Aspire orchestrator and dashboard, print the dashboard URL and the resolved `Outputs`, and keep running until interrupted. On interrupt it SHALL stop every resource it started.

#### Scenario: Successful start

- **WHEN** the user runs `booga start infra.yaml` on a valid template
- **THEN** all resources start, the dashboard URL and each output are printed, and the process keeps running

#### Scenario: Ctrl+C stops everything

- **WHEN** the user interrupts a running `booga start`
- **THEN** every container and process it started is stopped before it exits

#### Scenario: Invalid template starts nothing

- **WHEN** the template has validation errors
- **THEN** `booga start` prints them, starts nothing and exits non-zero

### Requirement: The tool only starts things

The tool SHALL NOT generate source code, SHALL NOT write files into the template's directory or the repository, SHALL NOT invoke a build of any project, and SHALL NOT keep state between runs. Data a resource itself persists (for example a declared container volume) is not tool state.

#### Scenario: Working tree untouched

- **WHEN** `booga start infra.yaml` runs and is stopped
- **THEN** `git status` in the repository shows no new or changed files

#### Scenario: No build is triggered

- **WHEN** a template is started
- **THEN** no compiler or build tool is invoked by `booga`

### Requirement: Parameters are supplied CloudFormation-style

`start`, `validate` and `doctor` SHALL accept `--parameters` either as `file://<path>` to a JSON array of `{ "ParameterKey", "ParameterValue" }` objects, or as one or more inline `ParameterKey=<k>,ParameterValue=<v>` pairs. Inline values SHALL override file values. A key that is not a declared parameter SHALL be an error.

#### Scenario: Parameter file

- **WHEN** the user runs `booga start infra.yaml --parameters file://params.dev.json`
- **THEN** each parameter takes its value from the file, and parameters not in the file take their `Default`

#### Scenario: Undeclared parameter key

- **WHEN** the parameter file contains `ParameterKey: Imagetag` and the template declares `ImageTag`
- **THEN** the tool fails before starting anything and suggests `ImageTag`

### Requirement: Missing secrets are prompted for

When a `NoEcho` parameter has no supplied value and no `Default`, `start` SHALL prompt for it without echoing input when attached to a terminal, and SHALL fail with an error naming the parameter otherwise.

#### Scenario: Interactive prompt

- **WHEN** `RabbitMqPassword` is `NoEcho` without a value and `booga start` runs in a terminal
- **THEN** the user is asked for it and the typed characters are not shown

#### Scenario: Non-interactive run

- **WHEN** the same template runs with no terminal attached
- **THEN** the tool exits non-zero naming `RabbitMqPassword` as missing

### Requirement: validate checks a template without side effects

`booga validate <template>` SHALL run every check `start` runs before starting, SHALL start nothing and contact no container runtime or network, and SHALL exit `0` when the template is valid and non-zero otherwise.

#### Scenario: Valid template

- **WHEN** the user runs `booga validate infra.yaml` on a valid template
- **THEN** it reports the template as valid and exits 0 without starting anything

### Requirement: doctor derives prerequisites from the template

`booga doctor <template>` SHALL determine the machine prerequisites from the resource types the template uses, check each one, and report each as pass or fail with an install hint. A container runtime (Docker or Podman) SHALL be required only when the template uses a container-backed type. The .NET SDK SHALL never be a prerequisite.

#### Scenario: Container template without Docker

- **WHEN** the template uses `Docker::Container` and no container runtime is reachable
- **THEN** doctor reports the container runtime as failing, with an install hint, and exits non-zero

#### Scenario: Template without containers

- **WHEN** the template uses no container-backed type
- **THEN** doctor does not check for a container runtime

### Requirement: Exit codes distinguish outcomes

The tool SHALL exit `0` on success, `1` when a template, parameter or prerequisite check fails, and `2` on a usage error (unknown command or option, missing template path).

#### Scenario: Usage error

- **WHEN** the user runs `booga strat infra.yaml`
- **THEN** the tool prints usage and exits 2

### Requirement: Secrets never appear in output

No command SHALL print, log or include in an error the value of a `NoEcho` parameter.

#### Scenario: Error near a secret

- **WHEN** a resource that receives a `NoEcho` value fails to start
- **THEN** the error message does not contain the secret's value
