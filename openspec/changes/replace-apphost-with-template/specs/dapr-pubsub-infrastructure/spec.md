# Spec Delta

## ADDED Requirements

### Requirement: Dapr pub/sub declared in the system template

The Dapr pub/sub component SHALL be declared as a resource in the system template (`infra.yaml`), backed by the template's RabbitMQ broker. The template SHALL NOT spell out the broker's connection string; it SHALL be derived from the broker reference. No static Dapr component YAML files SHALL exist in the repository.

#### Scenario: Component declared in the template

- **WHEN** `infra.yaml` is inspected
- **THEN** it contains a pub/sub component resource named `pubsub` whose broker is a reference to the RabbitMQ resource
- **AND** there is no `components/*.yaml` file defining the pub/sub component

#### Scenario: RabbitMQ backs the component

- **WHEN** the system is started from the template
- **THEN** the `pubsub` component is of type `pubsub.rabbitmq` and is connected to the template's RabbitMQ broker

## MODIFIED Requirements

### Requirement: RabbitMQ broker provisioned by Aspire

The system template SHALL declare RabbitMQ as a broker resource, which Aspire runs, that acts as the Dapr pub/sub broker. Its credentials SHALL be supplied as template parameters, with the password as a secret (`NoEcho`), and its management plugin SHALL be exposed for local diagnostics.

#### Scenario: RabbitMQ resource is available before dependents start

- **WHEN** the distributed application starts
- **THEN** RabbitMQ starts and dependent resources and components wait for it before initializing

#### Scenario: Management plugin exposed

- **WHEN** RabbitMQ is provisioned locally
- **THEN** its management plugin endpoint is exposed for inspection

### Requirement: Dapr sidecars attached to publishing and subscribing projects

Every application that publishes or subscribes to integration events SHALL have a Dapr sidecar declared in the system template, and that sidecar SHALL reference the pub/sub component so the application can publish to and receive from the broker.

#### Scenario: API project gets a sidecar referencing pub/sub

- **WHEN** the API resource is declared in the system template
- **THEN** a sidecar resource is declared for it that references the pub/sub component

#### Scenario: Sidecar waits for the broker

- **WHEN** an application's Dapr sidecar starts
- **THEN** the pub/sub component and its RabbitMQ dependency are ready before the application begins publishing or subscribing

## REMOVED Requirements

### Requirement: Dapr pub/sub configured via Aspire Community Toolkit, not YAML

**Reason**: The C# AppHost is replaced by the declarative system template (`infra.yaml`), which is YAML by design. The requirement's intent, no hand-written Dapr component files and no hand-written connection strings, is kept in "Dapr pub/sub declared in the system template".

**Migration**: Declare or change the pub/sub component in `infra.yaml` as a `Dapr::Component::PubSub` resource instead of in `AppHost.cs`.
