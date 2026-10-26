# Ticketium — Architecture

This document describes the architectural style and design of Ticketium using
**Diagram-as-Code** (Mermaid), which renders natively in GitHub Markdown.

## 1. Architectural Style

Ticketium is delivered as a **cloud-hosted application** with:

- A **Single Page Application (SPA)** front end communicating over REST and WebSockets.
- A **modular backend** (modular monolith) exposing one API, with clear internal module
  boundaries — Identity/Auth, Catalog, Booking, Payment — that can be split into separate
  deployable services later if scale demands it.
- A **relational database** as the system of record for strong booking consistency.
- A **cache / distributed-lock store** for time-limited seat holds, also used as the
  pub/sub backplane that spreads real-time updates across API instances.
- A **transactional outbox** and an internal **message bus** for event-driven side effects
  (notifications, refunds, seat-availability updates), processed by **background workers**
  hosted in every API instance.
- Resilient integration with an **external payment gateway** via a circuit breaker, and an
  **e-mail service** for buyer and organizer notifications.

This style keeps early development simple while preserving a clean path toward a
microservice/serverless split under load.

## 2. System Context

```mermaid
graph TD
    Buyer([Ticket Buyer])
    Organizer([Event Organizer])
    Admin([System Administrator])

    Buyer --> SPA[Web SPA]
    Organizer --> SPA
    Admin --> SPA

    SPA -->|REST / WebSocket| API[Ticketium API]
    API -->|charge / refund| Gateway[[External Payment Gateway]]
    API -->|notifications| Mail[[E-mail Service]]
```

## 3. Container / Component View

```mermaid
graph TD
    subgraph Client
        SPA[Web SPA]
    end

    subgraph Backend[Ticketium API]
        Gateway2[API Gateway / Router]
        Auth[Identity &amp; Auth Module]
        Catalog[Catalog Module]
        Booking[Booking Module]
        Payment[Payment Module]
        Notify[Notification Module]
        RT[Realtime / WebSocket Hub]
        Workers[Background Workers<br/>hold expiry, outbox relay,<br/>saga recovery, capture, refunds]
        Bus{{Message Bus}}
    end

    subgraph Data
        DB[(Relational DB<br/>incl. outbox)]
        Cache[(Cache / Seat-Lock Store<br/>+ Pub/Sub backplane)]
    end

    PG[[Payment Gateway]]
    Mail[[E-mail Service]]

    SPA -->|REST| Gateway2
    SPA <-->|WebSocket| RT
    Gateway2 --> Auth
    Gateway2 --> Catalog
    Gateway2 --> Booking
    Gateway2 --> Payment

    Auth --> DB
    Catalog --> DB
    Booking --> DB
    Booking -->|locks, seat changes| Cache
    Payment -->|confirm / release hold| Booking
    Payment --> DB
    Payment -->|bulkhead + circuit breaker| PG

    Workers --> DB
    Workers --> Cache
    Workers -->|publish outbox events| Bus
    Bus --> Catalog
    Bus --> Payment
    Bus --> Notify
    Bus --> RT
    RT <-->|seat updates| Cache
    Notify --> Mail
```

Modules that change state write an event to the outbox table in the same database
transaction; the outbox relay publishes it to the message bus, and each subscriber reacts on
its own (e.g. tickets e-mailed, capacity updated, refunds started). Seat changes reach
browsers through one path: they are published to the pub/sub backplane (holds and releases
by Booking, sales by the Realtime Hub's handler of the bus event), and every instance's hub
pushes them to its own WebSocket connections. The Payment module changes holds and seat
locks only through the Booking module. See [Concurrency Patterns](./CONCURRENCY_PATTERNS.md)
for the flows.

## 4. Data Model

Six core entities: **User**, **Event**, **Venue**, **Hold**, **Ticket**, **Transaction**.

- A **Hold** is the durable record of a checkout reservation with status `Active`, `Paid`,
  `Released` or `Expired`. The seat-lock store guarantees that a seat belongs to only one
  hold at a time; the database keeps the hold's lifecycle, so the API can tell an expired
  hold (`410`) from a paid one (`409`). A released hold keeps its record for audit but no
  longer exists for the API (`404`).
- An **Event** sells the seats of its venue's seat map, so its capacity is the size of that
  map; `seats_remaining` is maintained by the Catalog. `Cancelled` and `SoldOut` are stored
  in `status`; `Scheduled`, `OnSale`, `SalesClosed` and `Started` follow from `sales_open_at`,
  `sales_close_at`, `starts_at` and the database clock, so no job has to flip them.
  `version` is the event's `ETag`.
- A **Transaction** moves `Authorized → Captured`; when its event is cancelled it becomes
  `Voided` (not yet captured) or `RefundPending → Refunded`. `gateway_auth_id` is the
  gateway reference used to capture, void and refund. A **Ticket** is `Valid`, and becomes
  `Cancelled` and then `Refunded` if its event is cancelled.
- Money values are stored as an amount plus an ISO 4217 currency code, matching the
  `Money` object of the API.
- Supporting tables of the concurrency patterns (`checkout_sagas`, `idempotency_keys`,
  `outbox`, `processed_messages`, refund checkpoints) are described in
  [Concurrency Patterns](./CONCURRENCY_PATTERNS.md) and omitted from the diagram.

```mermaid
erDiagram
    USER ||--o{ TRANSACTION : makes
    USER ||--o{ EVENT : organizes
    VENUE ||--o{ EVENT : hosts
    EVENT ||--o{ TICKET : has
    USER ||--o{ HOLD : places
    EVENT ||--o{ HOLD : has
    HOLD ||--o| TRANSACTION : "paid by"
    TRANSACTION ||--|{ TICKET : covers

    USER {
        uuid id PK
        string email
        string password_hash
        string role
    }
    VENUE {
        uuid id PK
        string name
        int capacity
        json seat_map
    }
    EVENT {
        uuid id PK
        uuid venue_id FK
        uuid organizer_id FK
        string title
        datetime starts_at
        datetime sales_open_at
        datetime sales_close_at
        int seats_remaining
        decimal price
        string currency
        string status
        datetime cancelled_at
        uuid cancelled_by FK
        string cancellation_reason
        int version
    }
    HOLD {
        uuid id PK
        uuid event_id FK
        uuid user_id FK
        json seat_labels
        string status
        datetime expires_at
        datetime created_at
    }
    TICKET {
        uuid id PK
        uuid event_id FK
        uuid transaction_id FK
        string seat_label
        string status
        decimal price
        string currency
    }
    TRANSACTION {
        uuid id PK
        uuid user_id FK
        uuid hold_id FK
        decimal amount
        string currency
        string status
        string gateway_auth_id
        datetime created_at
    }
```

## 5. Key Flow — Concurrency-safe Seat Booking

The booking flow is the heart of the system. A seat is **held** for a limited time during
checkout; a **saga** compensates by releasing the hold when payment fails or times out.
A payment is executed as **authorize, then capture**: the gateway first authorizes the amount,
the booking is confirmed, and only then is the payment captured (in the background), so a
failed checkout voids an authorization instead of refunding money (see
[Concurrency Patterns S2](./CONCURRENCY_PATTERNS.md#2-s2-paying-for-a-hold-checkout-saga)).

```mermaid
sequenceDiagram
    actor Buyer
    participant API
    participant Booking
    participant Cache as Seat-Lock Store
    participant Payment
    participant PG as Payment Gateway

    Buyer->>API: Select seats
    API->>Booking: Request hold
    Booking->>Cache: Acquire locks (seats, TTL)
    alt Seats free
        Cache-->>Booking: Locks acquired
        Booking-->>API: Hold confirmed
        API-->>Buyer: Hold (expiresAt)
        Buyer->>API: Submit payment
        API->>Payment: Pay for hold
        Payment->>PG: Authorize (timeout + circuit breaker)
        alt Authorized
            PG-->>Payment: Authorized
            Payment->>Booking: Confirm booking
            Booking->>Cache: Promote locks to sold
            Booking-->>Payment: Tickets issued
            Payment-->>API: Transaction
            API-->>Buyer: Tickets
            Payment->>PG: Capture (background, retried)
        else Declined, gateway error or timeout
            PG-->>Payment: Declined / error / no answer
            Payment-)PG: Void authorization, if any
            Payment->>Booking: Compensate
            Booking->>Cache: Release locks
            Booking-->>Payment: Seats released
            Payment-->>API: Payment failed
            API-->>Buyer: Payment failed, select seats again
        end
    else Seat already held
        Cache-->>Booking: Lock denied
        Booking-->>API: Seat unavailable
        API-->>Buyer: Seat unavailable
    end
```

## 6. Distributed & Resiliency Patterns

| Pattern                      | Where / Why |
|------------------------------|-------------|
| Distributed Lock / Seat Hold | Lock a seat during checkout to prevent double-selling. |
| Saga (compensation)          | Release the held seat if payment fails or expires. |
| Try/Confirm/Cancel           | Reserve seats (hold) and money (authorization) before confirming either. |
| Idempotency Key              | Make payment retries safe: one charge per checkout. |
| Transactional Outbox         | Publish side effects (e-mails, capacity, refunds) reliably after a commit. |
| Circuit Breaker + Bulkhead   | Protect against payment-gateway failures and cap concurrent gateway calls. |
| Real-time Streaming          | Push "seats remaining" updates to clients over WebSockets. |

These and the other patterns used (balking, barrier, readers-writer lock, reactor and more)
are applied in detail, with sequence diagrams, in
[Concurrency Patterns](./CONCURRENCY_PATTERNS.md).

## 7. Evolution Path

The modular boundaries allow extracting high-load modules (e.g., Booking) into independent
services, and moving event-driven work (hold expiry, notifications) to **serverless**
functions, without changing the public API contract.
