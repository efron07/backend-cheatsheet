# Backend Cheat Sheet — Node.js, Express, PostgreSQL & MongoDB

## How to use this sheet

This sheet builds the **backend** for the PharmaCare React/Next.js frontend. It follows the roadmap.sh Backend, Node.js, Express, API Design, PostgreSQL DBA and MongoDB roadmaps, in the order you need them:

1. **Fundamentals** (section 1) — what a backend does, HTTP, the terminal.
2. **Node.js** (sections 2–3) — the runtime that runs JavaScript on the server.
3. **Express** (sections 4–5) — the web framework that turns Node into an API.
4. **API design** (sections 6–7) — how to design endpoints people enjoy using.
5. **PostgreSQL on Neon** (sections 8–10) — the main relational database, plus DBA skills.
6. **MongoDB** (sections 11–12) — a document database for flexible data.
7. **Production skills** (sections 13–20) — auth, caching, jobs, security, testing, observability, architecture, deployment.
8. **Frontend connection** (section 21) and **quick reference** (section 22).

Every topic uses the same pattern as the Laravel and Frontend sheets: **what it is**, **why it matters**, **how to do it**, and a **PharmaCare example**. Targets: Node.js 22/24 LTS, Express 5, TypeScript, PostgreSQL 17+ on Neon, MongoDB 8 with Mongoose, Zod 4.

## The PharmaCare backend architecture

```
  Customers (browser)          Staff (POS, dashboard)          Mobile app
          \                           |                          /
           \                          |                         /
            +------------  Next.js frontend (Vercel) ----------+
                                      |
                      HTTPS JSON  (Server Components / Actions)
                                      |
                      +---------------v----------------+
                      |   PharmaCare API  (Node.js +   |
                      |   Express + TypeScript)        |
                      |   routes → controllers →        |
                      |   services → repositories       |
                      +---+----------+----------+------+
                          |          |          |
            +-------------v--+  +----v-----+  +-v---------------+
            | PostgreSQL     |  | MongoDB  |  | Redis           |
            | (Neon)         |  | (Atlas)  |  | cache + BullMQ  |
            | medicines,     |  | audit    |  | job queues      |
            | batches, sales,|  | logs,    |  +-----------------+
            | users, money   |  | events,  |
            +----------------+  | reviews  |       M-Pesa, SMS, email
                                +----------+       (external services)
```

## Why two databases?

| Data | Database | Why |
| --- | --- | --- |
| Users, medicines, batches, sales, payments, suppliers | **PostgreSQL (Neon)** | Money and stock need strict structure, relations, constraints and transactions |
| Audit logs ("who changed this price"), activity events, customer reviews, supplier catalogue imports with varying fields | **MongoDB (Atlas)** | Flexible document shapes, high write volume, easy to evolve |
| Cache, sessions, job queues | **Redis** | Very fast in-memory key-value store |

Rule of thumb: **if it involves money or stock, it goes in PostgreSQL.** Many real systems use only PostgreSQL (its `JSONB` columns cover flexible data). Using both here teaches you both.

## The API we are building

| Method | Endpoint | Who | Purpose |
| --- | --- | --- | --- |
| POST | `/v1/auth/login` | Staff | Log in, receive tokens |
| GET | `/v1/medicines` | Public | List/search medicines (paginated) |
| GET | `/v1/medicines/:slug` | Public | One medicine with stock |
| POST | `/v1/medicines` | Pharmacist, admin | Create a medicine |
| PATCH | `/v1/medicines/:id` | Pharmacist, admin | Update price, details |
| POST | `/v1/batches` | Pharmacist, admin | Receive stock |
| POST | `/v1/sales` | Cashier+ | Make a sale (deducts stock) |
| GET | `/v1/sales/:id` | Cashier+ | Sale with items |
| POST | `/v1/payments/mpesa/callback` | M-Pesa | Payment webhook |
| GET | `/v1/reports/daily` | Pharmacist, admin | Sales summary |
| GET | `/v1/audit-logs` | Admin | MongoDB audit trail |

Money is stored as whole TZS integers everywhere (no floats).

## Sections

| Part | Pages |
| --- | --- |
| **Foundations** | [1 Backend fundamentals](01-fundamentals.md) |
| **Node.js** | [2 Basics](02-node-basics.md) · [3 Internals](03-node-internals.md) |
| **Express** | [4 Basics](04-express-basics.md) · [5 Production patterns](05-express-production.md) |
| **API design** | [6 REST conventions](06-api-design-rest.md) · [7 Docs, auth styles, webhooks](07-api-design-advanced.md) |
| **PostgreSQL (Neon)** | [8 SQL](08-postgres-sql.md) · [9 DBA](09-postgres-dba.md) · [10 From Node.js](10-postgres-node.md) |
| **MongoDB** | [11 Basics & modelling](11-mongodb-basics.md) · [12 Aggregation & Mongoose](12-mongodb-advanced.md) |
| **Production** | [13 Auth](13-auth.md) · [14 Caching & scaling](14-caching-scaling.md) · [15 Jobs & real-time](15-jobs-realtime.md) · [16 Security](16-security.md) · [17 Testing](17-testing.md) · [18 Observability](18-observability.md) · [19 Architecture](19-architecture.md) · [20 Deployment](20-deployment.md) |
| **Full stack** | [21 Connecting Next.js](21-nextjs-integration.md) |
| **Reference** | [22 Quick reference](22-quick-reference.md) |

Tip: press `/` or `S` on any page to search the whole sheet.
