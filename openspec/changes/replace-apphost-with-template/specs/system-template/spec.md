# Spec Delta

## Purpose

Defines the repository's system template, `infra.yaml`: the single declarative description from which the whole BoogaBooster system is started locally.

## ADDED Requirements

### Requirement: One template starts the whole system

The repository SHALL contain `infra.yaml` at its root, and `booga start infra.yaml` SHALL start RabbitMQ, the Dapr pub/sub component, the API with its Dapr sidecar, and the frontend. No other command SHALL be needed to run the system locally.

#### Scenario: Start from a clean clone

- **WHEN** a user with `booga` and Docker, and no .NET SDK, Node or Dapr CLI, runs `booga start infra.yaml` in a fresh clone and supplies the RabbitMQ password
- **THEN** the operator dashboard loads in the browser with live telemetry, queue and weather

### Requirement: The template is valid and declarative

`infra.yaml` SHALL pass `booga validate` and SHALL use only the constructs the template format allows.

#### Scenario: Validation in CI

- **WHEN** CI runs `booga validate infra.yaml`
- **THEN** it exits 0

### Requirement: Applications run from published images

The API and frontend resources SHALL run the published `booga-booster-api` and `booga-booster-frontend` images, with the tag taken from the `ImageTag` parameter (default `main`).

#### Scenario: Pinning a version

- **WHEN** the user starts the template with `ImageTag` set to `1.4.0`
- **THEN** both the API and frontend containers run the `1.4.0` images

#### Scenario: Locally built images

- **WHEN** a developer has built images locally with tag `dev` and starts the template with `ImageTag` set to `dev`
- **THEN** the locally built images are used

### Requirement: RabbitMQ credentials are template parameters

The RabbitMQ user name and password SHALL be template parameters. The password SHALL be `NoEcho` without a default. Local parameter files matching `*.parameters.local.json` SHALL be ignored by git.

#### Scenario: Local parameter file stays out of git

- **WHEN** a developer saves the password in `infra.parameters.local.json`
- **THEN** `git status` does not list that file

### Requirement: Resource names keep service discovery working

The API resource SHALL keep the runtime name `fourdotnet-boogabooster-api`, and the Dapr pub/sub component SHALL keep the name `pubsub`, so the frontend's API discovery and the API's `IntegrationMessagingDefaults.PubSubName` keep matching.

#### Scenario: Frontend reaches the API

- **WHEN** the system is started from the template
- **THEN** `/api/weather` through the frontend returns `200`

### Requirement: The API runs in the Development environment

The API resource SHALL run with `ASPNETCORE_ENVIRONMENT=Development`, so that its health endpoints are available and its readiness can gate dependants.

#### Scenario: Health endpoint

- **WHEN** the API container is running under the template
- **THEN** its `/health` endpoint returns `200`

### Requirement: Outputs point at what a user opens

The template SHALL declare outputs for the frontend URL and the RabbitMQ management URL.

#### Scenario: Outputs after start

- **WHEN** `booga start infra.yaml` has started everything
- **THEN** it prints the frontend URL and the RabbitMQ management URL
