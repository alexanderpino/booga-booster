# Tasks

## 0. Preconditions

- [ ] 0.1 Verify `infra-template-tool` is released (`booga --version` works from the release) and `container-images-ci` has published both images to GHCR with the `main` tag; stop if either is missing

## 1. System template (`system-template`, `dapr-pubsub-infrastructure`)

- [ ] 1.1 Add `infra.yaml` at the repository root per design D1, adjusted to the released catalog's property names; verify `booga validate infra.yaml` exits 0
- [ ] 1.2 Add `infra.parameters.example.json` (no real secret) and the `*.parameters.local.json` entry to `.gitignore`; verify `git status` ignores a created `infra.parameters.local.json`
- [ ] 1.3 Start the system with `booga start infra.yaml` while the AppHost still exists; verify the dashboard shows live telemetry, queue and weather, `/api/weather` through the frontend returns `200`, a `weather-updated` event reaches the Queue module through RabbitMQ, and both outputs are printed
- [ ] 1.4 Add a CI step that runs `booga validate infra.yaml` on pull requests; verify it passes on this change's PR

## 2. Move ServiceDefaults

- [ ] 2.1 Move the project to `src/Shared/FourDotnet.BoogaBooster.ServiceDefaults/` (renamed project file, unchanged code and namespace), update `BoogaBooster.slnx` and the API's project reference; verify `dotnet build src/BoogaBooster.slnx` and `dotnet test src/BoogaBooster.slnx` pass
- [ ] 2.2 Update the code comments that name the old location or the AppHost (`BoogaBoosterTelemetry.cs`, `IntegrationMessagingDefaults.cs`, `DomainInstrumentTests.cs`, `proxy.conf.js`); verify the solution still builds and the Angular tests pass

## 3. Remove the AppHost

- [ ] 3.1 Delete `src/Aspire/FourDotnet.BoogaBooster.Aspire.AppHost/` and `src/Aspire/aspire.config.json`, remove the `/Aspire/` folder from `BoogaBooster.slnx`, and remove the Aspire launch configurations from `.vscode/launch.json`; verify `dotnet build` and `dotnet test` on the solution pass and `src/Aspire/` no longer exists
- [ ] 3.2 Re-run `booga start infra.yaml` after the removal; verify the same checks as task 1.3 pass

## 4. Rules, skills, agents and plugin

- [ ] 4.1 Add `docs/adr/0010-declarative-system-template.md` (Accepted, supersedes ADR-0008 here, links the three changes) and list it in `docs/adr/README.md`; verify the link from the README resolves
- [ ] 4.2 Delete `.claude/skills/csharp-aspire/` and `plugins/4dotnet-boogabooster/skills/csharp-aspire/`; bump the plugin version (minor) and remove "Aspire" from its description; verify no file references the `csharp-aspire` skill
- [ ] 4.3 Update `csharp-observability`, `csharp-solution-structure`, `test-coverage` and the `csharp-expert` agent (both `.claude/` and plugin copies) to the new ServiceDefaults path, without AppHost or ADR-0008 rules; verify with a grep for `AppHost` and `ADR-0008` in those files
- [ ] 4.4 Update `CLAUDE.md` and `.github/copilot-instructions.md` per design D5 (architecture, messaging, common commands, the explicit style-guide exception, the skill list); verify both describe `booga start infra.yaml` and neither mentions user-secrets or the AppHost
- [ ] 4.5 Update `.github/code-review/review-prompt.md` so it no longer expects an AppHost; verify `node --test .github/code-review/publish-review.test.mjs .github/code-review/compare-reviews.test.mjs` passes
- [ ] 4.6 Run `node .claude/skills/ai-tooling-sync/scripts/sync-ai-tooling.mjs --fix` and then without `--fix`; verify it reports no errors

## 5. Documentation and in-flight changes

- [ ] 5.1 Update `README.md` (how to run, prerequisites: `booga` and Docker; developer loop per design D4; secrets via parameter file) and `src/Shared/FourDotnet.BoogaBooster.IntegrationMessages/README.md` (pub/sub is declared in `infra.yaml`); verify the README's run instructions work from a fresh clone
- [ ] 5.2 Update `workshop-prerequisites` artifacts per design D7 (replace the AppHost-secrets and Dapr CLI checks; reference `booga doctor infra.yaml`); verify `openspec validate workshop-prerequisites` passes
- [ ] 5.3 Reword AppHost mentions in `backend-observation`, `ecs-foundation`, `job-system`, `park-economy` and `coaster-editor` to the system template; verify `openspec validate` passes for each

## 6. Final integration check

- [ ] 6.1 Grep the repository for `AppHost`, `ADR-0008`, `csharp-aspire`, `user-secrets` and `src/Aspire`; verify hits remain only in `openspec/changes/archive/`, the new ADR and this change's own artifacts
- [ ] 6.2 On a machine (or container) with only `booga` and Docker, clone the branch, run `booga doctor infra.yaml` and `booga start infra.yaml --parameters file://infra.parameters.local.json`; verify the doctor is green and the dashboard works end to end

## Workflow follow-up

- Apply in a worktree and branch per the `git-change-workflow` skill; open a PR into `main`. Expect `major` review findings for the `CLAUDE.md`, `.claude/` and `.github/` edits.
- Archive this change after merge so `dapr-pubsub-infrastructure`, `weather-api-client` and the new `system-template` specs are published.
