# Spec Delta

## Purpose

Defines the tools and documentation that let a person or an agent write a correct catalog definition: a JSON Schema, a dependency-free validation script that gives the same verdicts as the server, and an authoring guide.

## ADDED Requirements

### Requirement: Published JSON Schema

The repository SHALL contain a JSON Schema (draft 2020-12) for the definition format at `src/Catalog/FourDotnet.BoogaBooster.Catalog/Schema/catalog-definition.schema.json` with `additionalProperties: false` on every object. The schema SHALL express what a schema can express (types, enums, ranges, patterns, required fields per kind); rules that need several fields (file name equals id, attach cell inside footprint, kind-section matching) SHALL be checked by the loader and the script, and listed in the authoring guide.

#### Scenario: Typo is caught by schema

- **WHEN** a definition has the property `buildCostCent`
- **THEN** validating it against the schema fails

#### Scenario: Roster validates

- **WHEN** every shipped definition is validated against the schema
- **THEN** all pass

### Requirement: Validation script

`npm run catalog:validate` (run in the Angular app folder) SHALL run `tools/catalog/validate-catalog.mjs`, which uses no network and no installed packages beyond Node, reads all definitions of a directory (default: the shipped definitions; `--dir <path>` for another; repeatable), applies every rule of the definition format capability including the semantic ones, prints one line per error as `<file>: <code>: <json path>: <detail>` and a final summary, and exits with code 1 on any error, 0 otherwise. `--json` SHALL print the errors as a JSON array. When the asset manifest exists, a `--check-art` flag SHALL warn about art asset ids missing from it; the absence of the manifest SHALL be reported as information, not an error.

#### Scenario: Clean run

- **WHEN** the script runs on the shipped roster
- **THEN** it prints a summary of 20 definitions, 0 errors, and exits 0

#### Scenario: Error line format

- **WHEN** a definition has a negative cost
- **THEN** a line `frost-cart.json: ValueOutOfRange: economy.buildCostCents: -1 is below 0` (or equivalent wording) is printed and the exit code is 1

#### Scenario: Missing manifest

- **WHEN** `--check-art` is used and no manifest exists
- **THEN** the script prints that art ids were not checked and does not fail for it

### Requirement: Script and loader agree

For a shared set of fixture files, the script and the C# loader SHALL report the same set of error codes and JSON paths. The fixtures SHALL include at least one valid file and one invalid file for every error code of the definition format capability. A test on each side SHALL read the same fixture directory and expected-results file.

#### Scenario: Parity

- **WHEN** both validators run over the fixtures
- **THEN** their `(file, code, path)` sets equal the expected-results file

#### Scenario: New error code

- **WHEN** a new error code is added to the loader without a fixture
- **THEN** the coverage test over the code list fails

### Requirement: Authoring guide

`docs/park/catalog-authoring-guide.md` SHALL explain, for a person or an agent: the file layout and naming, every field with units and ranges, the rules per kind, how to choose footprint, clearance, attach points and rotations (with an ASCII diagram), how ratings and levels combine with a worked example, how to pick art ids and tints, how to run the script, how to add the file to the extra directory and restart, the error code table with fixes, the originality rule (own names and numbers, no third-party names or assets, denylist), and a copy-paste prompt recipe for an agent. Every example JSON in the guide SHALL be extracted and validated by a test.

#### Scenario: Examples are valid

- **WHEN** the guide's example blocks are extracted and validated
- **THEN** each marked valid passes and each marked invalid fails with the stated code

#### Scenario: Error table complete

- **WHEN** the guide's error table is compared with the code list
- **THEN** every code appears exactly once

### Requirement: Originality check

A name denylist file SHALL list third-party ride, product and brand names that must not appear in definition names, ids, descriptions or art asset ids. The script and the loader tests SHALL fail a definition containing a listed term (case-insensitive, whole words).

#### Scenario: Listed term

- **WHEN** a definition name contains a denylisted word
- **THEN** the script reports `NameDenied` and exits 1

#### Scenario: Roster passes

- **WHEN** the script runs on the roster
- **THEN** no `NameDenied` is reported
