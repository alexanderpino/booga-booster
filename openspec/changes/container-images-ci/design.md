# Design

## Context

- The API (`src/FourDotnet.BoogaBooster.Api`) is an ASP.NET Core app on .NET 10. It calls `UseHttpsRedirection()`, and through ServiceDefaults it maps `/health` and `/alive` only in the `Development` environment. Its Dapr subscription endpoint comes from `MapSubscribeHandler()`.
- The frontend (`src/FourDotnet.BoogaBooster.App`) uses relative `/api/*` URLs everywhere, including the SSE telemetry stream (`/api/ride/telemetry/stream`). In development those URLs reach the API through `proxy.conf.js`, which reads `services__fourdotnet-boogabooster-api__https__0`/`__http__0`.
- The only existing workflow is `code-quality-check.yml`.

## Goals / Non-Goals

**Goals:**
- Images that run with nothing but a container runtime, and that need no rebuild per environment.
- Keep the app's relative `/api` URLs unchanged, so the frontend code does not change.

**Non-Goals:**
- Production hardening (TLS termination, CSP headers, image signing, SBOMs) `(deferred: demo scope)`.
- Multi-architecture images beyond `linux/amd64` and `linux/arm64`.
- Changing the API's code. Health endpoints stay `Development`-only; the template sets `ASPNETCORE_ENVIRONMENT=Development` (in `replace-apphost-with-template`).

## Decisions

### D1. API image via the .NET SDK's built-in container publishing

CI runs `dotnet publish src/FourDotnet.BoogaBooster.Api -t:PublishContainer` with the repository and tag set from the workflow. This produces a non-root, runtime-only image on Microsoft's ASP.NET base image, listening on 8080 by default (`ASPNETCORE_HTTP_PORTS=8080`).

*Alternative:* a multi-stage Dockerfile. Rejected: more to maintain for the same result. A Dockerfile remains possible if base-image customisation is needed later.

No HTTPS port is configured in the container, so `UseHttpsRedirection()` cannot determine one and does not redirect. That keeps `/dapr/subscribe` on plain HTTP, as the sidecar's app channel needs.

### D2. Frontend image: Angular build plus unprivileged nginx

A multi-stage Dockerfile in `src/FourDotnet.BoogaBooster.App/`. Stage 1 (`node`) runs `npm ci` and `npm run build`. Stage 2 is `nginxinc/nginx-unprivileged`, which copies the browser build output and an nginx config template. The nginx image renders `/etc/nginx/templates/*.template` with environment variables at start, so the API address is injected at run time, not baked in.

```
browser ──▶ frontend :8080 ──┬── /            -> static files, fallback index.html
                             └── /api/(.*)    -> ${API_URL}/$1
                                                 proxy_buffering off
                                                 proxy_read_timeout 1h
                                                 proxy_http_version 1.1
```

The variable name `services__fourdotnet-boogabooster-api__http__0` contains characters that are awkward in nginx config. An entrypoint script copies it into `API_URL` and exits with a clear message when it is missing.

*Alternative:* letting the API serve the static files. Rejected: it couples the two builds and changes the API's code and HTTP surface.

### D3. SSE passthrough

The `/api` location turns off proxy buffering and caching, uses HTTP/1.1 with an empty `Connection` header, and sets a long read timeout. No API change is required. This is covered by an integration check (task 3.2).

### D4. One workflow, PRs build and `main`/tags push

`.github/workflows/container-images.yml` runs on `pull_request` into `main` (build only), `push` to `main`, and tags `v*`. It uses `docker/setup-buildx-action`, `docker/metadata-action` for the tag set and `docker/login-action` with `GITHUB_TOKEN` (`packages: write` only on non-PR events). Third-party actions are pinned by commit SHA.

## Risks / Trade-offs

- [nginx buffers or times out the SSE stream] → explicit config (D3) and a check that frames arrive live and the stream survives more than 60 s.
- [GHCR packages default to private, so the template's pull fails on other machines] → document in the README that the packages must be made public once, or that `docker login ghcr.io` is needed.
- [`linux/arm64` builds under QEMU are slow] → accepted. Native ARM runners are an option if build times hurt.
- [The workflow change gets a `major` code-review finding] → expected by design; it asks for human review.

## Migration Plan

Additive. The images are not used until `replace-apphost-with-template` points the template at them. Rollback is removing the workflow and the build files. Published images can stay.
