# Ticketium

> Event Ticketing & Seat Reservation System — a platform for organizing events and buying tickets.

**Team:** x53AIBoosted

**Members:**

- Vladyslav Hrytsyk
- Oleksandr Kuzmeniuk
- Danylo Kozlovskyi

Ticketium lets organizers create events and set seat capacities, while buyers browse events, pick seats, and pay. Its core challenge is concurrency: safely handling many users trying to buy the **same seat at the same time**.

## Actors

- **Event Organizer** — creates events and venues, sets seat capacities.
- **Ticket Buyer** — browses events, selects seats, and pays.

## Modules

A modular design with separated responsibilities:

| Module | Responsibility |
|---------|----------------|
| **Identity / Auth** | User registration, login, and token issuance. |
| **Catalog** | Events and venue information. |
| **Booking** | Seat selection and reservation (strong consistency, no double-selling). |
| **Payment** | Payment processing via an external gateway. |

## Concurrency & Distributed Patterns

- **Distributed Lock / Two-Phase Commit** — lock a seat while the buyer is in checkout.
- **Saga Pattern** — roll back a reserved seat if payment fails.
- **Circuit Breaker** — handle timeouts and failures from the external payment gateway.

## Data Model

Five core entities: **User**, **Event**, **Venue**, **Ticket**, **Transaction**.

- **High consistency** for booking data — no double-selling of seats.
- **Streaming** for real-time "seats remaining" updates via WebSockets.

## Analytics Funnel

Event Discovery → Seat Selection → Checkout → Successful Payment.

## Resiliency & Threat Model

- Automated ticket scalping (bots).
- DDoS attacks during high-demand ticket drops.
- Payment gateway failures and timeouts.

## Status

This README describes the intended design. Setup and usage instructions will be added as features are implemented.
