# Spec Delta

## Purpose

Defines how the `booga` tool is packaged and released so that it runs on a machine without the .NET SDK or runtime installed.

## ADDED Requirements

### Requirement: booga is a self-contained binary

`booga` SHALL be released as a self-contained executable per platform that includes the .NET runtime, the Aspire orchestrator and the Aspire dashboard. Running it SHALL NOT require the .NET SDK, a .NET runtime or network access to a package feed.

#### Scenario: Machine without .NET

- **WHEN** `booga validate infra.yaml` runs on a machine with no .NET SDK or runtime installed
- **THEN** it runs and reports the template's validity

#### Scenario: No package restore at start

- **WHEN** `booga start` runs on a machine without network access to NuGet
- **THEN** the orchestrator and dashboard start from the bundled binaries

### Requirement: Supported platforms

Releases SHALL include binaries for `win-x64`, `win-arm64`, `linux-x64`, `linux-arm64` and `osx-arm64`.

#### Scenario: Release assets

- **WHEN** a release is published
- **THEN** it has one archive per supported platform, each with a SHA-256 checksum

### Requirement: Version is reported

`booga --version` SHALL print the tool's version and the Aspire version it bundles.

#### Scenario: Version output

- **WHEN** the user runs `booga --version`
- **THEN** both versions are printed and the tool exits 0

### Requirement: Released on GitHub Releases and winget

Each tagged release SHALL publish the platform archives to GitHub Releases and SHALL produce a winget manifest that installs the Windows binary and puts `booga` on `PATH`.

#### Scenario: Install with winget

- **WHEN** a Windows user installs the package with winget
- **THEN** `booga --version` works in a new terminal
