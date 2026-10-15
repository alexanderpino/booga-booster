# Spec Delta

## Purpose

Defines the container images that let the API and the frontend run without the .NET SDK or Node on the machine, their run-time contract, and how CI builds and publishes them.

## ADDED Requirements

### Requirement: API image runs the API without the SDK

The `booga-booster-api` image SHALL contain the published API and the ASP.NET Core runtime only, SHALL run as a non-root user, and SHALL listen on plain HTTP on container port `8080`.

#### Scenario: API answers in a container

- **WHEN** the API image is started with `ASPNETCORE_ENVIRONMENT=Development`
- **THEN** `GET /weather` on container port 8080 returns `200`

#### Scenario: Non-root

- **WHEN** the image's configured user is inspected
- **THEN** it is not `root`

### Requirement: API image exposes health and Dapr endpoints

When run with `ASPNETCORE_ENVIRONMENT=Development`, the API container SHALL serve the health endpoints and the Dapr subscription endpoint over plain HTTP on port 8080 without redirecting to HTTPS.

#### Scenario: Subscription discovery over HTTP

- **WHEN** a client sends `GET /dapr/subscribe` to the container on port 8080
- **THEN** it receives the topic subscriptions with `200`, not a redirect

### Requirement: Frontend image serves the built app

The `booga-booster-frontend` image SHALL serve the production build of the Angular app as static files on container port `8080` without Node, SHALL run as a non-root user, and SHALL answer unknown non-`/api` paths with the app's `index.html`.

#### Scenario: App shell

- **WHEN** a browser requests `/` from the frontend container
- **THEN** it receives the app's `index.html`

#### Scenario: Deep link

- **WHEN** a browser requests a client-side route that is not a file
- **THEN** it receives `index.html` with `200`

### Requirement: Frontend reverse-proxies /api to the API

The frontend container SHALL forward requests under `/api/` to the API with the `/api` prefix removed. The API address SHALL be read at container start from `services__fourdotnet-boogabooster-api__http__0`; when that variable is missing the container SHALL fail to start with a message naming it.

#### Scenario: Proxied call

- **WHEN** the browser calls `/api/weather` on the frontend container
- **THEN** the API receives `GET /weather` and its response is returned

#### Scenario: Missing API address

- **WHEN** the frontend container starts without `services__fourdotnet-boogabooster-api__http__0`
- **THEN** it exits non-zero and its log names the missing variable

### Requirement: Server-Sent Events pass through unbuffered

The frontend's reverse proxy SHALL pass `text/event-stream` responses through without buffering, and SHALL keep a stream open for as long as the client and the API keep it open.

#### Scenario: Telemetry frames arrive live

- **WHEN** the browser opens `/api/ride/telemetry/stream` through the frontend container
- **THEN** telemetry frames arrive at the API's emission rate rather than in delayed batches, and the connection stays open for more than 60 seconds

### Requirement: CI builds images on pull requests

CI SHALL build both images on every pull request into `main` and SHALL NOT push them.

#### Scenario: Broken image build fails the PR

- **WHEN** a pull request breaks the frontend image build
- **THEN** the image workflow's check fails on that pull request

### Requirement: CI publishes images to GHCR

On every push to `main` and on every version tag `v*`, CI SHALL push both images to `ghcr.io/alexanderpino/booga-booster-api` and `ghcr.io/alexanderpino/booga-booster-frontend`, tagged `main` (on `main`), `sha-<short commit>`, and the version without the `v` (on a tag).

#### Scenario: Merge to main

- **WHEN** a commit lands on `main`
- **THEN** both images exist in GHCR under the tags `main` and `sha-<that commit>`

#### Scenario: Version tag

- **WHEN** tag `v1.4.0` is pushed
- **THEN** both images exist under the tag `1.4.0`
