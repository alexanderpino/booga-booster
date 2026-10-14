# Tasks

## 1. API image

- [ ] 1.1 Configure container publishing for `FourDotnet.BoogaBooster.Api` (repository name, non-root, port 8080); verify `dotnet publish -t:PublishContainer` produces a local image whose user is not root
- [ ] 1.2 Run the image with `ASPNETCORE_ENVIRONMENT=Development` and verify `GET /weather`, `GET /health` and `GET /dapr/subscribe` on port 8080 return `200` without a redirect

## 2. Frontend image

- [ ] 2.1 Add the multi-stage Dockerfile and `.dockerignore` in `src/FourDotnet.BoogaBooster.App/` (Node build stage, unprivileged nginx run stage); verify `docker build` succeeds and the image's user is not root
- [ ] 2.2 Add the nginx config template (static files, `index.html` fallback, `/api/` proxy with prefix stripping, SSE settings) and the entrypoint that maps `services__fourdotnet-boogabooster-api__http__0` to `API_URL` and fails clearly when it is missing; verify `/`, a deep link and the missing-variable case by hand against the running image
- [ ] 2.3 Document both images (ports, environment variables, how to run them locally) in a `docs/container-images.md` page; verify the documented `docker run` commands work as written

## 3. Integration check

- [ ] 3.1 Add a script under `tools/` that starts both images on one Docker network, with the frontend pointed at the API; verify it exits 0 when `/api/weather` through the frontend returns `200`
- [ ] 3.2 Extend the script to open `/api/ride/telemetry/stream` through the frontend and assert that the first frame arrives within 2 s and that frames keep arriving for more than 60 s
- [ ] 3.3 Verify the script passes locally and record its command in `docs/container-images.md`

## 4. CI workflow

- [ ] 4.1 Add `.github/workflows/container-images.yml`: build both images (amd64 and arm64) on pull requests into `main` without pushing, with third-party actions pinned by SHA; verify the check runs on this change's PR
- [ ] 4.2 Run the integration script (task 3.1/3.2) in the workflow against the freshly built images; verify a deliberately broken nginx config fails the job (then revert)
- [ ] 4.3 On push to `main` and on `v*` tags, log in to GHCR with `GITHUB_TOKEN` (`packages: write`) and push with the tags `main`, `sha-<short>` and the version; verify after merge that both packages show the expected tags
- [ ] 4.4 Add a README section on the GHCR packages (visibility, `docker login ghcr.io` when private); verify the instructions against the published packages

## Workflow follow-up

- Apply in a worktree and branch per the `git-change-workflow` skill; open a PR into `main`.
- Production hardening (TLS, security headers, signing, SBOM) is `(deferred: demo scope)`; list it in the PR description.
