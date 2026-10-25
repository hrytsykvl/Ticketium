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
- A **cache / distributed-lock store** for time-limited seat holds.
- Resilient integration with an **external payment gateway** via a circuit breaker.

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
        RT[Realtime / WebSocket Hub]
    end

    subgraph Data
        DB[(Relational DB)]
        Cache[(Cache / Seat-Lock Store)]
    end

    PG[[Payment Gateway]]

    SPA -->|REST| Gateway2
    SPA <-->|WebSocket| RT
    Gateway2 --> Auth
    Gateway2 --> Catalog
    Gateway2 --> Booking
    Gateway2 --> Payment

    Auth --> DB
    Catalog --> DB
    Booking --> DB
    Booking --> Cache
    Booking --> RT
    Payment -->|circuit breaker| PG
```

## 4. Data Model

Five core entities: **User**, **Event**, **Venue**, **Ticket**, **Transaction**.

```mermaid
erDiagram
    USER ||--o{ TRANSACTION : makes
    USER ||--o{ EVENT : organizes
    VENUE ||--o{ EVENT : hosts
    EVENT ||--o{ TICKET : has
    TRANSACTION |o--|{ TICKET : covers

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
        datetime sales_close_at
        int capacity
        decimal price
        string status
    }
    TICKET {
        uuid id PK
        uuid event_id FK
        uuid transaction_id FK
        string seat_label
        string status
        decimal price
    }
    TRANSACTION {
        uuid id PK
        uuid user_id FK
        decimal amount
        string status
        datetime created_at
    }
```

## 5. Key Flow — Concurrency-safe Seat Booking

The booking flow is the heart of the system. A seat is **held** for a limited time during
checkout; a **saga** compensates by releasing the hold when payment fails or times out.

```mermaid
sequenceDiagram
    actor Buyer
    participant API
    participant Booking
    participant Cache as Seat-Lock Store
    participant Payment
    participant PG as Payment Gateway

    Buyer->>API: Select seat
    API->>Booking: Request hold
    Booking->>Cache: Acquire lock (seat, TTL)
    alt Seat free
        Cache-->>Booking: Lock acquired
        Booking-->>API: Hold confirmed
        Buyer->>API: Submit payment
        API->>Payment: Charge
        Payment->>PG: Process (timeout + circuit breaker)
        alt Payment success
            PG-->>Payment: Approved
            Payment->>Booking: Confirm booking
            Booking->>Cache: Promote lock to sold
            Booking-->>API: Ticket issued
        else Payment failure / timeout
            PG-->>Payment: Declined / timeout
            Payment->>Booking: Compensate
            Booking->>Cache: Release lock
            Booking-->>API: Seat released
        end
    else Seat already held
        Cache-->>Booking: Lock denied
        Booking-->>API: Seat unavailable
    end
```

## 6. Distributed & Resiliency Patterns

| Pattern                      | Where / Why |
|------------------------------|-------------|
| Distributed Lock / Seat Hold | Lock a seat during checkout to prevent double-selling. |
| Saga (compensation)          | Release the held seat if payment fails or expires. |
| Circuit Breaker              | Protect against payment-gateway timeouts and failures. |
| Real-time Streaming          | Push "seats remaining" updates to clients over WebSockets. |

Each pattern is applied in detail, with sequence diagrams, in
[Concurrency Patterns](./CONCURRENCY_PATTERNS.md).

## 7. Evolution Path

The modular boundaries allow extracting high-load modules (e.g., Booking) into independent
services, and moving event-driven work (hold expiry, notifications) to **serverless**
functions, without changing the public API contract.
