# Spec Delta

## MODIFIED Requirements

### Requirement: Reach the API through a dev proxy

The app SHALL call the weather API using relative `/api` paths. When the system runs from the system template, the frontend container's reverse proxy SHALL forward them to the API resolved from the service-discovery environment variable. When the app runs standalone with the dev server, the dev-server proxy SHALL forward them to the API from that variable or a localhost fallback, and SHALL accept the development certificate.

#### Scenario: Relative calls are proxied to the API

- **WHEN** the app calls `/api/weather` (or the disturbance paths)
- **THEN** the request is forwarded to the API resolved from the service-discovery environment variable
- **AND** the `/api` prefix is stripped so it maps to the API's `/weather` routes

#### Scenario: Runs under Aspire and standalone

- **WHEN** the system is started from the system template
- **THEN** the frontend container's reverse proxy targets the API's injected endpoint
- **AND** when the dev server is started standalone it falls back to a local default target
