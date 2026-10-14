## Purpose

Makes a green doctor run the mandatory first step of every roadmap change, by order in the roadmap and by a standard first task group in each change's task list.

## ADDED Requirements

### Requirement: The roadmap applies workshop-prerequisites first

`docs/roadmap.md` SHALL list `workshop-prerequisites` as change 0 at the top of the apply order, in the dependency graph as the root that every other change depends on, in the change table with its size, and in a "Doctor gate" note stating that no change is applied while `node tools/doctor/doctor.mjs --change <name>` exits non-zero. Change 0 SHALL be applied before any other change and SHALL be the only change that does not carry the Preconditions group (its own tasks create doctor).

#### Scenario: Order is visible

- **WHEN** the roadmap is read
- **THEN** the first entry of the apply order is `0 workshop-prerequisites` and every other change shows it as a dependency or sits below it in the graph

### Requirement: Every roadmap change starts with a Preconditions task group

The `tasks.md` of every other change listed in `docs/roadmap.md` SHALL begin, before group 1, with a group titled exactly `## 0. Preconditions` whose first task is, verbatim except for the change name:

```
- [ ] 0.1 Run `node tools/doctor/doctor.mjs --change <change-name>` from the repository root. If it exits non-zero, STOP: report the failing checks and their hints, install nothing without being asked, and do not start group 1. Verify: exit status 0.
```

Group 0 SHALL NOT renumber existing groups. A change that needs tools beyond the default checks SHALL tag its tasks with the `[needs-*]` vocabulary so `--change` makes those checks required.

#### Scenario: Conforming change

- **WHEN** `tasks.md` of `ecs-foundation` starts with the group and the task naming `--change ecs-foundation`
- **THEN** the gate lint passes for it

#### Scenario: Wrong change name

- **WHEN** the task names `--change job-system` inside `ecs-foundation`
- **THEN** the gate lint reports the mismatch

### Requirement: The gate is machine-checkable

`node tools/doctor/doctor.mjs --lint-gate` SHALL read the change names from the `### N <name>` headings of `docs/roadmap.md`, exempt those listed in `gateExempt` of `doctor.config.json` (initially `workshop-prerequisites`), and for every other change that still exists under `openspec/changes/` verify that the first `## ` heading of its `tasks.md` is `## 0. Preconditions` and that task 0.1 contains `node tools/doctor/doctor.mjs --change <that change's folder name>`. It SHALL print one line per violation, exit 1 when any exist and 0 otherwise, support `--json`, and treat a roadmap change whose folder was already archived as satisfied.

#### Scenario: Missing group

- **WHEN** `tasks.md` of `scene-graph` starts with `## 1.`
- **THEN** the lint prints `scene-graph: first task group is not "0. Preconditions"` and exits 1

#### Scenario: Archived change ignored

- **WHEN** a roadmap change has moved to `openspec/changes/archive/`
- **THEN** the lint does not report it

### Requirement: The gate wording is added to the other changes at apply time

When change 0 is applied, the apply step SHALL insert the Preconditions group into each other roadmap change's `tasks.md` (edits to planning artifacts only, one commit, listed in the pull request description) and run the gate lint to green. Later changes written after this change was applied SHALL include the group from the start.

#### Scenario: Insert and verify

- **WHEN** the insertion commit is made
- **THEN** `--lint-gate` exits 0 and `openspec validate --all` shows no new errors
