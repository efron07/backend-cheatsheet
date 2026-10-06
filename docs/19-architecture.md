# 19. Architecture and system design

## 19.1 Monolith, modular monolith, microservices

| Architecture | What it is | Pros | Cons |
| --- | --- | --- | --- |
| **Monolith** | One deployable app | Simple to build, test, deploy | Can become tangled as it grows |
| **Modular monolith** | One deployable, strict internal modules (medicines, sales, payments) talking through clear interfaces | Simple ops + clean boundaries; easy to split later | Requires discipline |
| **Microservices** | Many small services, each owning its data, communicating over HTTP or messages | Teams and services scale independently | Network failures, distributed data, much more ops |
| **Serverless functions** | Functions run on demand (Vercel, AWS Lambda, Cloudflare Workers) | No servers, scale to zero | Cold starts, time limits, DB connection limits |

**Recommendation for PharmaCare:** a **modular monolith** (the feature folders in section 4.3). Split out a service only when there is a clear reason (a different scaling profile, a separate team, a different security boundary like payments).

## 19.2 Layered and clean architecture

Dependencies point inward: HTTP and databases are details; business rules are the core.

```
  Routes / Controllers (HTTP)      Jobs / Webhooks        CLI scripts
               \                          |                   /
                +----------->  Services (business rules)  <---+
                                          |
                         Repositories / gateways (interfaces)
                                          |
                 PostgreSQL     MongoDB     Redis     M-Pesa     SMS
```

Benefits: test business rules without HTTP or a database; swap the SMS provider or database driver without touching the rules. Related ideas: **hexagonal architecture** (ports and adapters), **domain-driven design** (bounded contexts like Sales, Inventory, Payments with their own language).

## 19.3 Backend-for-Frontend (BFF)

**What it is:** A server layer built for one specific frontend, shaping data exactly as that UI needs. In PharmaCare, **Next.js Server Components and Server Actions act as the BFF**: they call the Express API (which serves all clients: web, mobile, partners), combine responses, keep tokens server-side and return only what the page needs. Section 21 shows it.

## 19.4 Design patterns you will use

| Pattern | Where in PharmaCare |
| --- | --- |
| **Repository** | Database access behind `medicineRepository` |
| **Service layer** | Business rules in `saleService` |
| **Dependency injection** | Passing repositories/gateways into services for testability |
| **Adapter / gateway** | `PaymentGateway` interface with M-Pesa and Airtel implementations |
| **Strategy** | Different pricing/discount rules selected at runtime |
| **Observer / pub-sub** | `bus.emit('sale.completed')` → listeners |
| **Middleware / chain of responsibility** | Express middleware |
| **Factory** | `createApp()`, `createMedicineService(deps)` |
| **Singleton** | One DB pool, one Redis client per process |
| **Outbox** | Reliable event publishing (below) |
| **Saga** | Multi-step flows with compensating actions (refund if dispatch fails) |
| **CQRS** | Separate read models (materialised daily sales) from write models |

## 19.5 Message brokers and event-driven architecture

**What it is:** Services communicate by publishing **events** ("SaleCompleted") to a **broker**; other services subscribe. The producer does not wait for or even know about consumers.

| Tool | Model | Use |
| --- | --- | --- |
| **Redis + BullMQ** | Job queues | Background work inside one system (section 15) |
| **RabbitMQ** | Message queues with routing (exchanges) | Commands/events between services |
| **Apache Kafka / Redpanda** | Durable, replayable event log | High-volume event streams, analytics pipelines |
| **NATS** | Lightweight pub/sub | Low-latency messaging |
| Cloud: SQS/SNS, Google Pub/Sub, Azure Service Bus | Managed | No broker to operate |

**The outbox pattern** — how to publish events reliably: write the event to an `outbox` table **in the same database transaction** as the sale; a separate process reads unpublished rows and pushes them to the queue/broker, marking them sent. This guarantees you never commit a sale without its event, or send an event for a rolled-back sale.

**Delivery guarantees:** most systems give **at-least-once** delivery, so consumers must be **idempotent** (dedupe by event ID).

## 19.6 Data consistency and the CAP theorem

**CAP theorem:** in a distributed system, during a network **P**artition you must choose between **C**onsistency (everyone sees the same data) and **A**vailability (every request gets an answer).

- **Strong consistency** — PostgreSQL in one region: stock levels are always exact. Required for money and stock.
- **Eventual consistency** — replicas, caches, search indexes and other services catch up after a short delay. Fine for catalogue pages, reports and audit logs.

Design which data needs which, and tell users when they might see slightly old data ("Stock updated 1 minute ago").

## 19.7 System design building blocks

| Block | Purpose |
| --- | --- |
| DNS + CDN | Route users, cache static/public content near them |
| Load balancer | Spread traffic over API instances, health-check them |
| Reverse proxy (Nginx, Caddy, Traefik) | TLS, compression, routing, rate limiting |
| API servers (stateless) | Business logic, scale horizontally |
| Cache (Redis) | Fast reads, sessions, rate limits |
| Primary database + replicas | Source of truth; replicas for reads |
| Object storage (R2/S3) | Files |
| Queues/brokers + workers | Asynchronous and scheduled work |
| Search engine (Meilisearch, Typesense, OpenSearch) | Fast, typo-tolerant search |
| Observability stack | Logs, metrics, traces, alerts |

**Back-of-the-envelope estimate for PharmaCare:** 10 branches × 3 POS × \~300 sales/day = \~9,000 sales/day ≈ 0.1 sales/second on average, maybe 5/second at peak. One well-indexed PostgreSQL instance and two API instances handle this comfortably. Do the maths before adding complexity.

## 19.8 Multi-tenancy (if PharmaCare becomes SaaS for many pharmacies)

| Model | How | Trade-off |
| --- | --- | --- |
| Shared tables + `tenant_id` column | Every query filters by tenant (enforce with Row-Level Security) | Cheapest, needs strict discipline |
| Schema per tenant | `pharmacy_a.sales`, `pharmacy_b.sales` | More isolation, harder migrations |
| Database per tenant | Separate DB (easy with Neon projects/branches) | Strongest isolation, more to manage |
