# 6. API design 1: REST conventions

## 6.1 What good API design means

**What it is:** An API is a contract between your backend and its clients (Next.js, a mobile app, partners). Good design makes it **predictable** (same patterns everywhere), **hard to misuse**, **stable** (changes don't break clients) and **self-explanatory**.

**REST** (Representational State Transfer) is the most common style: everything is a **resource** (medicines, sales) with a URL, and you act on it with HTTP methods. Its principles: client–server separation, statelessness (each request carries everything needed), cacheability, a uniform interface, and a layered system.

## 6.2 Naming resources and URLs

| Rule | Good | Bad |
| --- | --- | --- |
| Nouns, not verbs | `POST /sales` | `POST /createSale` |
| Plural collections | `/medicines`, `/medicines/7` | `/medicine/7` |
| Lowercase, hyphens | `/stock-adjustments` | `/stockAdjustments`, `/stock_adjustments` |
| Nest only for clear ownership, max one level | `/sales/1052/items` | `/branches/2/sales/1052/items/3/batches` |
| Filters in the query string | `/medicines?category=antibiotics&inStock=true` | `/medicines/antibiotics/in-stock` |
| Actions that are not CRUD: a sub-resource | `POST /sales/1052/void`, `POST /sales/1052/refunds` | `GET /voidSale?id=1052` |
| Use stable identifiers | `/medicines/7` (id) or `/medicines/amoxicillin-500mg` (slug, for public URLs) | Names that change |

## 6.3 Methods and their meaning

| Request | Means | Success |
| --- | --- | --- |
| `GET /medicines` | List | 200 + collection |
| `GET /medicines/7` | Read one | 200, or 404 |
| `POST /medicines` | Create | 201 + `Location` header + the created resource |
| `PUT /medicines/7` | Replace completely | 200 or 204 |
| `PATCH /medicines/7` | Partial update (send only changed fields) | 200 + updated resource |
| `DELETE /medicines/7` | Delete (or soft-delete) | 204 |

**Idempotency:** `GET`, `PUT` and `DELETE` must give the same result if repeated (a retried `DELETE` of an already-deleted item returns 204 or 404, never an error that breaks the client). `POST` is not idempotent; protect money operations with an **Idempotency-Key** (section 7).

## 6.4 Status codes: which one when

| Situation | Status |
| --- | --- |
| Read OK / update OK | 200 |
| Created a sale | 201 |
| Deleted / nothing to return | 204 |
| Accepted for background processing (M-Pesa STK push started) | 202 |
| Body is not valid JSON | 400 |
| No or invalid token | 401 (+ `WWW-Authenticate`) |
| Cashier tries to change a price | 403 |
| Medicine does not exist | 404 |
| Duplicate batch number / not enough stock / version conflict | 409 |
| Field validation failed | 422 |
| Too many requests | 429 (+ `Retry-After`) |
| Unexpected bug | 500 |
| Upstream (M-Pesa) failed / timed out | 502 / 504 |

Never return `200` with `{ "success": false }`: clients, caches and monitoring rely on status codes.

## 6.5 Response format

Pick one shape and use it everywhere.

```json5
// Single resource
{ "data": { "id": 7, "name": "Amoxicillin 500mg", "slug": "amoxicillin-500mg", "price": 500, "stock": 180 } }

// Collection with pagination metadata
{
  "data": [ { "id": 7, "name": "Amoxicillin 500mg" } ],
  "meta": { "page": 2, "limit": 20, "total": 412, "totalPages": 21 },
  "links": { "next": "/v1/medicines?page=3&limit=20", "prev": "/v1/medicines?page=1&limit=20" }
}
```

**Conventions:**

- **camelCase** field names in JSON (matches JavaScript clients). Be consistent.
- Dates as **ISO 8601 strings in UTC**: `"2026-10-06T09:15:00Z"`; dates without time as `"2027-03-31"`.
- Money as **integers in the smallest unit** with a currency: `"price": 500, "currency": "TZS"`.
- IDs: numbers or strings, but consistent; never expose internal secrets in IDs.
- Return only what clients need; never return password hashes, tokens or internal flags.
- Enums as readable strings: `"status": "paid"`, not `2`.

## 6.6 Error format

Use one error shape. The standard **Problem Details** format (RFC 9457) is a good default:

```text
HTTP/1.1 409 Conflict
Content-Type: application/problem+json

{
  "type": "insufficient_stock",
  "title": "Only 3 of Amoxicillin 500mg in stock",
  "status": 409,
  "details": { "medicineId": 7, "available": 3, "requested": 5 },
  "requestId": "2b8f1e0a-..."
}
```

```text
HTTP/1.1 422 Unprocessable Content

{
  "type": "validation_error",
  "title": "Validation failed",
  "status": 422,
  "errors": { "quantity": ["Must be at least 1"], "expiresAt": ["Batch has already expired"] }
}
```

The `errors` object maps straight onto form fields in React Hook Form (Frontend sheet, section 16).

## 6.7 Pagination

| Type | Request | Pros | Cons |
| --- | --- | --- | --- |
| **Offset** | `?page=3&limit=20` (SQL `LIMIT 20 OFFSET 40`) | Simple, jump to any page, show totals | Slow on deep pages; rows shift when data changes |
| **Cursor (keyset)** | `?limit=20&cursor=eyJpZCI6MTA1Mn0` | Fast at any depth, stable under inserts | No "jump to page 37" |

Use offset for admin tables (medicines), cursor for large, growing feeds (sales history, audit logs).

```ts
// Cursor pagination for sales: "give me 20 sales older than the last one I saw"
const rows = await db.select().from(sales)
  .where(cursor ? lt(sales.id, cursor) : undefined)
  .orderBy(desc(sales.id))
  .limit(limit + 1);                                   // fetch one extra to know if there is a next page
const hasMore = rows.length > limit;
const data = rows.slice(0, limit);
res.json({ data, meta: { nextCursor: hasMore ? data.at(-1)!.id : null } });
```

Always cap `limit` (e.g. max 100) so nobody can request a million rows.

## 6.8 Filtering, sorting, searching and field selection

```
GET /v1/medicines?search=amox&categoryId=1&inStock=true&minPrice=200&maxPrice=1000
GET /v1/medicines?sort=-price,name              # minus = descending
GET /v1/sales?from=2026-10-01&to=2026-10-06&paymentMethod=mpesa
GET /v1/medicines?fields=id,name,price          # sparse fieldsets (optional)
GET /v1/sales/1052?include=items,customer       # embed related data (optional)
```

Validate every filter with Zod (section 5.2) and **whitelist** sortable fields — never put user input directly into `ORDER BY`.

## 6.9 Versioning

**Why:** mobile apps installed on phones cannot all update at once; breaking changes need a new version.

| Method | Example |
| --- | --- |
| **URL path** (most common, used here) | `/v1/medicines`, later `/v2/medicines` |
| Header | `Accept: application/vnd.pharmacare.v2+json` |
| Query | `/medicines?version=2` |

**Non-breaking (no new version needed):** adding endpoints, adding optional fields, adding new enum values clients already tolerate. **Breaking:** removing or renaming fields, changing types or meaning, making optional fields required. Deprecate old versions with a timeline and a `Deprecation`/`Sunset` header.

## 6.10 HATEOAS and related ideas

**HATEOAS** (Hypermedia as the Engine of Application State) means responses include links to related actions (`"links": { "void": "/v1/sales/1052/void" }`). Full HATEOAS is rare in practice; pagination `links` are the common, useful part.

**Design checklist:**

- [ ] Resource names are plural nouns; URLs lowercase with hyphens.
- [ ] Correct methods and status codes; no `200` for errors.
- [ ] One response envelope and one error format.
- [ ] Pagination with a capped `limit`; filters and sorting validated.
- [ ] Versioned from day one (`/v1`).
- [ ] Money as integers; dates as ISO 8601 UTC.
- [ ] Documented in OpenAPI (section 7).
