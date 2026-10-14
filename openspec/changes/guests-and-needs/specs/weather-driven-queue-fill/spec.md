# Spec Delta

## ADDED Requirements

### Requirement: The background filler can be switched off by configuration

The Queue module SHALL provide a configuration option `Queue:Filler:Enabled` (default true) that, when false, prevents the background filler from enqueuing any guest and from starting its timer. All other Queue behaviour (the queue service, the status endpoint, the weather subscription and the group-queued event for any caller that enqueues) SHALL be unchanged.

#### Scenario: Default keeps filling

- **WHEN** the host starts without the option
- **THEN** the filler behaves exactly as the published requirements describe

#### Scenario: Disabled filler enqueues nothing

- **WHEN** the option is false and the host runs for several fill intervals
- **THEN** no guest is enqueued by the filler and the queue length is unchanged

#### Scenario: Other Queue features stay available

- **WHEN** the option is false
- **THEN** the weather-update subscription still acknowledges events and the queue status endpoint still answers
