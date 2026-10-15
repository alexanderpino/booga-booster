# Spec Delta

## Purpose

Defines the closed catalog of resource types a template may use: how types are named by provider, the properties each accepts, the attributes each publishes, and what each starts.

## ADDED Requirements

### Requirement: Types are named by provider namespace

A resource `Type` SHALL have the form `<Provider>::<Name>` or `<Provider>::<Group>::<Name>`, where `<Provider>` is a catalog namespace. A type that is not in the catalog SHALL be a validation error that suggests the closest known type.

#### Scenario: Unknown type with suggestion

- **WHEN** a resource has `Type: Rabbitmq::Broker`
- **THEN** validation fails and the error asks whether `RabbitMQ::Broker` was meant

#### Scenario: Unknown provider

- **WHEN** a resource has `Type: Oracle::Database` and the catalog has no `Oracle` provider
- **THEN** validation fails and the error lists the available providers

### Requirement: Types validate their own properties

Each type SHALL define its accepted properties with their value types and which are required. An unknown property, a missing required property or a value of the wrong type SHALL be a validation error naming the property and the type.

#### Scenario: Unknown property

- **WHEN** a `Docker::Container` resource has a property `Imag`
- **THEN** validation fails and the error asks whether `Image` was meant

### Requirement: The catalog can be listed

The tool SHALL list the catalog's types, and for a given type its properties (with value type, whether required, and default) and its published attributes.

#### Scenario: Describing a type

- **WHEN** the user asks for the description of `RabbitMQ::Broker`
- **THEN** the tool prints its properties and its attributes `Endpoint` and `ManagementUrl`

### Requirement: Docker::Container runs any image

`Docker::Container` SHALL start a container from `Image` (required) and `Tag` (default `latest`), and SHALL accept `Name`, `Ports`, `Environment`, `Volumes`, `Connections` and `Public`. It SHALL publish the attributes `Url` and `Endpoint`.

#### Scenario: Public container

- **WHEN** a `Docker::Container` has `Public: true` and a port
- **THEN** it is reachable from the host on that port and `Fn::GetAtt: [<Id>, Url]` returns its URL

### Requirement: Resource names are stable for service discovery

A resource's runtime name SHALL be its `Name` property when given, and its logical ID in lower case otherwise. Connection information injected into dependants SHALL use the runtime name, following Aspire's conventions (`ConnectionStrings__<name>` and `services__<name>__<endpoint>__0`).

#### Scenario: Named API container

- **WHEN** resource `Frontend` has `Connections: [{ Ref: Api }]` and `Api` has `Name: fourdotnet-boogabooster-api`
- **THEN** the frontend receives an environment variable `services__fourdotnet-boogabooster-api__http__0` holding the API's address

### Requirement: RabbitMQ::Broker runs a RabbitMQ broker

`RabbitMQ::Broker` SHALL start a RabbitMQ broker with `Username` and `Password` (both required), `Port` and an optional `ManagementPort` that enables the management plugin. It SHALL publish `Endpoint` and, when the management plugin is enabled, `ManagementUrl`.

#### Scenario: Management plugin enabled

- **WHEN** a broker declares `ManagementPort: 15672`
- **THEN** the management UI is reachable on host port 15672

### Requirement: Dapr::Component::PubSub declares a Dapr pub/sub component

`Dapr::Component::PubSub` SHALL declare a Dapr pub/sub component named `ComponentName`, backed by the resource given in `Broker`. The connection details for the broker SHALL be derived by the catalog; the template SHALL NOT have to spell out a connection string.

#### Scenario: RabbitMQ-backed pub/sub

- **WHEN** a component has `ComponentName: pubsub` and `Broker: { Ref: Messaging }` where `Messaging` is a `RabbitMQ::Broker`
- **THEN** a `pubsub.rabbitmq` component named `pubsub` is available to sidecars that reference it, connected to that broker with its credentials

#### Scenario: Unsupported broker

- **WHEN** `Broker` refers to a `Docker::Container`
- **THEN** validation fails and the error lists the broker types the component supports

### Requirement: Dapr::Sidecar attaches a sidecar to an app

`Dapr::Sidecar` SHALL attach a Dapr sidecar to the resource in `App`, using `AppProtocol` (`http` or `https`, default `http`) and `AppPort`, and SHALL make the components listed in `Components` available to it. The sidecar SHALL NOT require the Dapr CLI on the machine.

#### Scenario: Sidecar sees its components

- **WHEN** `ApiSidecar` has `App: { Ref: Api }` and `Components: [{ Ref: PubSub }]`
- **THEN** the app can publish to and receive from the `pubsub` component through its sidecar

#### Scenario: Subscriptions are discovered

- **WHEN** the app serves its Dapr subscription endpoint over the declared `AppProtocol`
- **THEN** the sidecar registers the app's topic subscriptions at start
