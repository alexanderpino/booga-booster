# Proposal

## Why

The system is meant to start from a template on a machine that has no .NET SDK, Node or Dapr CLI (see `infra-template-tool`). Today the API is only runnable from source with the SDK, and the frontend only through the Vite dev server with Node. Both applications therefore have to arrive as ready-made container images built by CI.

## What Changes

- Add a container image for the API, **`booga-booster-api`**, built from `FourDotnet.BoogaBooster.Api`. It listens on plain HTTP inside the container and needs no SDK at run time.
- Add a container image for the frontend, **`booga-booster-frontend`**. It serves the production Angular build as static files and reverse-proxies `/api/*` to the API, including the Server-Sent Events telemetry stream without buffering. The API address comes from the Aspire service-discovery environment variable, so the image needs no rebuild per environment.
- Add a CI workflow that builds both images on every pull request (no push) and publishes them to GitHub Container Registry (`ghcr.io/alexanderpino/…`) on every push to `main` and on version tags. Tags are `main`, `sha-<short>` and, on a version tag, the version.
- No behaviour changes in the API or the Angular app.

## Capabilities

### New Capabilities
- `container-images`: the API and frontend images, their run-time contract (ports, environment, health, `/api` reverse proxy and SSE passthrough), and how CI builds, tags and publishes them.

### Modified Capabilities

None. The frontend's existing dev-server proxy (`weather-api-client`) keeps working for `npm start`; `replace-apphost-with-template` updates that requirement when the AppHost goes.

## Impact

- **New:** a container build definition for the API (in or next to `src/FourDotnet.BoogaBooster.Api/`), and for the frontend a container build definition plus reverse-proxy configuration in `src/FourDotnet.BoogaBooster.App/`.
- **New workflow:** `.github/workflows/container-images.yml`, which needs `packages: write` to push to GHCR. Per the code-review prompt, a PR that touches `.github/workflows/` is flagged at `major` for human review.
- **Registry:** two GHCR packages under the repository owner. Their visibility (public or private) is set once in GitHub.
- **Not affected:** application code, tests and the AppHost.
