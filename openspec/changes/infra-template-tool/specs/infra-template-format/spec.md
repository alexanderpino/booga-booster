# Spec Delta

## Purpose

Defines the declarative, CloudFormation-style YAML template that describes a locally run system: its sections, how values and references work, and which constructs are deliberately not allowed.

## ADDED Requirements

### Requirement: Template is a YAML document with CloudFormation-style sections

A template SHALL be a single YAML document whose top-level keys are limited to `TemplateFormatVersion`, `Description`, `Metadata`, `Parameters`, `Resources` and `Outputs`. `TemplateFormatVersion` and `Resources` SHALL be required. Any other top-level key SHALL be a validation error.

#### Scenario: Minimal valid template

- **WHEN** a template contains `TemplateFormatVersion: "2026-10-14"` and a `Resources` section with one resource
- **THEN** the template validates successfully

#### Scenario: Unknown section is rejected

- **WHEN** a template contains a top-level `Conditions` key
- **THEN** validation fails with an error naming `Conditions` as an unsupported section and the line it is on

#### Scenario: Unsupported format version

- **WHEN** `TemplateFormatVersion` holds a version the tool does not support
- **THEN** validation fails and the error lists the supported versions

### Requirement: Template is plain YAML without custom tags

A template SHALL be valid plain YAML. YAML tags (for example `!Ref`, `!GetAtt`, `!Sub`) SHALL NOT be accepted; intrinsic functions SHALL be written in mapping form (`{ Ref: X }`, `{ Fn::GetAtt: [X, Attr] }`).

#### Scenario: Short-form tag is rejected with a hint

- **WHEN** a property value is written as `!Ref Messaging`
- **THEN** validation fails and the error suggests `{ Ref: Messaging }`

### Requirement: Resources have a logical ID, a Type and Properties

Each entry under `Resources` SHALL be keyed by a logical ID of letters and digits starting with a letter, and SHALL hold a `Type` and, optionally, `Properties`, `DependsOn` and `Metadata`. Any other key on a resource SHALL be a validation error.

#### Scenario: Resource without a Type

- **WHEN** a resource has `Properties` but no `Type`
- **THEN** validation fails with an error naming the logical ID

#### Scenario: Invalid logical ID

- **WHEN** a resource's logical ID is `my-api`
- **THEN** validation fails because logical IDs are alphanumeric

### Requirement: Ref resolves parameters and resources

`{ Ref: <name> }` SHALL resolve to the value of the parameter with that name, or, for a resource, to that resource as a connection target. A `Ref` to a name that is neither a parameter nor a resource SHALL be a validation error.

#### Scenario: Ref to a parameter yields its value

- **WHEN** a property is `{ Ref: ImageTag }` and the parameter `ImageTag` resolves to `1.4.0`
- **THEN** the property's value is `1.4.0`

#### Scenario: Ref to an unknown name suggests the closest match

- **WHEN** a property is `{ Ref: Mesaging }` and a resource `Messaging` exists
- **THEN** validation fails and the error asks whether `Messaging` was meant

### Requirement: Fn::GetAtt reads published attributes

`{ Fn::GetAtt: [<LogicalId>, <Attribute>] }` SHALL resolve to the named attribute of a resource. Only attributes the resource's type publishes SHALL be accepted; any other attribute SHALL be a validation error listing the published ones.

#### Scenario: Reading a published attribute

- **WHEN** an output's value is `{ Fn::GetAtt: [Messaging, ManagementUrl] }` and `RabbitMQ::Broker` publishes `ManagementUrl`
- **THEN** the output resolves to the broker's management URL once it is running

#### Scenario: Unknown attribute

- **WHEN** a template reads `{ Fn::GetAtt: [Messaging, Colour] }`
- **THEN** validation fails and the error lists the attributes `RabbitMQ::Broker` publishes

### Requirement: No computation in templates

The template format SHALL NOT support any construct that computes a value: `Conditions`, `Mappings`, `Fn::FindInMap`, `Fn::If`, `Fn::Sub`, `Fn::Join`, `Fn::Select`, `Fn::Split`, `Transform`, macros and `Custom::` resource types SHALL all be validation errors.

#### Scenario: String substitution is rejected

- **WHEN** a property value is `{ Fn::Sub: "amqp://${Messaging}" }`
- **THEN** validation fails and the error states that templates do not compute values

### Requirement: References imply start order

A resource that refers to another resource through `Ref` or `Fn::GetAtt` SHALL wait for that resource to be ready before it starts. `DependsOn` SHALL add an order dependency without a connection. A dependency cycle SHALL be a validation error naming the resources in the cycle.

#### Scenario: Implicit dependency

- **WHEN** resource `Api` has `Connections: [{ Ref: Messaging }]`
- **THEN** `Api` starts only after `Messaging` reports ready

#### Scenario: Cycle

- **WHEN** `A` refers to `B` and `B` has `DependsOn: A`
- **THEN** validation fails and the error names `A` and `B`

### Requirement: Parameters declare typed inputs

Each parameter SHALL declare `Type` (`String` or `Number`) and MAY declare `Default`, `Description`, `AllowedValues` and `NoEcho`. A supplied value that does not match its type or `AllowedValues` SHALL be rejected before anything starts.

#### Scenario: Value outside AllowedValues

- **WHEN** parameter `Mode` allows `emulator` and `local` and the value `cloud` is supplied
- **THEN** the tool fails before starting anything and names the allowed values

### Requirement: NoEcho parameters are secrets

A parameter with `NoEcho: true` SHALL be treated as a secret: its value SHALL never appear in console output, logs, validation errors or `Outputs`, and SHALL be passed to the runtime as a secret.

#### Scenario: Secret in an output

- **WHEN** an output's value is `{ Ref: RabbitMqPassword }` and that parameter is `NoEcho`
- **THEN** validation fails because outputs cannot expose secrets

### Requirement: Outputs describe what to show after start

Each entry under `Outputs` SHALL hold a `Value` (a literal, `Ref` or `Fn::GetAtt`) and MAY hold a `Description`. A `Ref` to a resource in an output SHALL resolve to that resource's primary URL.

#### Scenario: Output referencing a resource

- **WHEN** an output is `{ Value: { Ref: Frontend } }`
- **THEN** after start the output shows the frontend's URL

### Requirement: Validation errors are located and complete

Every validation error SHALL name the template file and the line and column it refers to. Validation SHALL report all errors it finds in one pass rather than stopping at the first.

#### Scenario: Several errors at once

- **WHEN** a template has an unknown type on line 12 and an unknown `Ref` on line 30
- **THEN** both errors are reported, each with its line number
