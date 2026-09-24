# Architecture

Fill this in for your organisation. Several skills read it to route work to the
right service before they start reading code, so the routing table at the bottom
matters more than the prose.

Keep it short. This is a map, not documentation.

## Services

| Service | Stack | What lives there |
|---|---|---|
| `{{FRONTEND_REPO}}` | | The user-facing app |
| `{{CORE_SERVICE}}` | | The main backend |
| `{{IDENTITY_SERVICE}}` | | Users, roles, permissions, schedules |
| `{{CUSTOMER_SERVICE}}` | | Customer and account records |
| `{{ENGINE_SERVICE}}` | | Specialised engine, solver or recommender |
| `{{INTEGRATION_SERVICE}}` | | Data integration and sync |

## How they talk

Name the transports: the event bus, the message broker, queues, direct REST,
and the identity provider. Be specific about which pairs of services use which,
because these are the seams bugs cluster at.

## Feature to service

The most useful section. One line per feature area.

| Feature area | Service |
|---|---|
| Login, sessions, permissions | |
| Core domain workflow | |
| Customer data, merge, consent | |
| Scheduling or optimisation | |
| Reporting and sync | |

## Known seams

List the places where two systems have to agree: an event contract, a shared
enum, a sync job, a cache that can go stale. A symptom that crosses a boundary
should make you suspect the boundary first.
