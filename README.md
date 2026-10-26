# Ticketium

> Event Ticketing & Seat Reservation System — a platform for organizing events and buying tickets.

**Team:** x53AIBoosted

**Members:**

- Vladyslav Hrytsyk
- Oleksandr Kuzmeniuk
- Danylo Kozlovskyi

Ticketium lets organizers create events and set seat capacities, while buyers browse events, pick seats, and pay. Its core challenge is concurrency: safely handling many users trying to buy the **same seat at the same time**.

## Documentation

- [Software Requirements Specification](docs/SRS.md) — functional specification, UML, identity management.
- [Architecture](docs/ARCHITECTURE.md) — architectural style and Mermaid diagrams.
- [Error Handling](docs/ERROR_HANDLING.md) — RFC 9457 Problem Details design and the [OpenAPI spec](docs/openapi.yaml) of 11 endpoints.
  View the spec in [Swagger Editor](https://editor.swagger.io/?url=https://raw.githubusercontent.com/hrytsykvl/Ticketium/main/docs/openapi.yaml).
- [Error Scenarios](docs/ERROR_SCENARIOS.md) ([PDF](docs/ERROR_SCENARIOS.pdf)) — request, error response and client reaction for every error of every endpoint.
- [Concurrency Patterns](docs/CONCURRENCY_PATTERNS.md) — 7 concurrency flows (sequence diagrams) using concurrency and distributed transaction patterns: distributed lock, saga, TCC, outbox, idempotency and more.
