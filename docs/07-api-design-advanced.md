# 7. API design 2: documentation, security styles, reliability and alternatives

## 7.1 Documenting with OpenAPI

**What it is:** **OpenAPI** (formerly Swagger) is a standard file (YAML/JSON) that describes every endpoint, parameter, request body, response and error. From it you get interactive docs, generated client SDKs, mock servers and contract tests.

**Generate it from your Zod schemas** so docs never drift from the code:

```bash
npm install @asteasolutions/zod-to-openapi swagger-ui-express
npm install -D @types/swagger-ui-express
```

```ts
// src/docs/openapi.ts
import { OpenAPIRegistry, OpenApiGeneratorV31, extendZodWithOpenApi } from '@asteasolutions/zod-to-openapi';
import { z } from 'zod';
extendZodWithOpenApi(z);

export const registry = new OpenAPIRegistry();

const Medicine = registry.register('Medicine', z.object({
  id: z.number().openapi({ example: 7 }),
  name: z.string().openapi({ example: 'Amoxicillin 500mg' }),
  price: z.number().int().openapi({ example: 500, description: 'Whole TZS' }),
  stock: z.number().int(),
}));

registry.registerPath({
  method: 'get',
  path: '/v1/medicines/{slug}',
  tags: ['Medicines'],
  request: { params: z.object({ slug: z.string() }) },
  responses: {
    200: { description: 'The medicine', content: { 'application/json': { schema: z.object({ data: Medicine }) } } },
    404: { description: 'Not found' },
  },
});

export const openApiDocument = new OpenApiGeneratorV31(registry.definitions).generateDocument({
  openapi: '3.1.0',
  info: { title: 'PharmaCare API', version: '1.0.0' },
  servers: [{ url: 'https://api.pharmacare.co.tz' }],
});
```

```ts
// app.ts
import swaggerUi from 'swagger-ui-express';
app.get('/openapi.json', (_req, res) => res.json(openApiDocument));
app.use('/docs', swaggerUi.serve, swaggerUi.setup(openApiDocument));   // interactive docs at /docs
```

The Next.js frontend can then generate a typed client with `openapi-typescript` + `openapi-fetch` (section 21). Alternative docs UIs: Scalar, Redoc.

## 7.2 API authentication styles

| Style | How it works | Best for |
| --- | --- | --- |
| **Session cookie** | Server stores the session (Redis/DB); browser sends a cookie | Same-site web apps |
| **JWT access token** (Bearer) | Signed token with user id and role, short-lived; sent in `Authorization` header | SPAs, mobile apps, service-to-service |
| **Refresh token** | Long-lived, stored as HttpOnly cookie or securely on device; used only to get new access tokens | Keeping users logged in safely |
| **API key** | A secret string per client, sent in a header | Server-to-server partners, internal jobs |
| **OAuth 2.0 / OpenID Connect** | Delegated login via Google, Microsoft, or your own auth server | "Login with Google", third-party apps |
| **HMAC signatures** | Request body signed with a shared secret | Webhooks |
| **mTLS** | Both sides present certificates | Banks, high-security integrations |

PharmaCare uses **short-lived JWT access tokens + rotating refresh tokens in HttpOnly cookies** (section 13).

## 7.3 Idempotency keys (critical for payments)

**What it is:** A client sends a unique `Idempotency-Key` header with a `POST`. If the same key arrives again (the cashier double-clicks, or the network retries after a timeout), the server returns the **original** result instead of creating a second sale or charging twice.

```ts
// src/middleware/idempotency.ts
import type { RequestHandler } from 'express';
import { redis } from '../lib/redis.ts';

export const idempotent: RequestHandler = async (req, res, next) => {
  const key = req.get('idempotency-key');
  if (!key) return res.status(400).json({ type: 'bad_request', title: 'Idempotency-Key header required', status: 400 });

  const cacheKey = `idem:${res.locals.user.id}:${key}`;
  const saved = await redis.get(cacheKey);
  if (saved === 'processing') return res.status(409).json({ type: 'in_progress', title: 'Request already in progress', status: 409 });
  if (saved) {
    const { status, body } = JSON.parse(saved);
    return res.status(status).json(body);                     // replay the original response
  }

  await redis.set(cacheKey, 'processing', 'EX', 60, 'NX');    // lock
  const originalJson = res.json.bind(res);
  res.json = (body) => {
    void redis.set(cacheKey, JSON.stringify({ status: res.statusCode, body }), 'EX', 60 * 60 * 24);
    return originalJson(body);
  };
  next();
};

saleRouter.post('/', authenticate, authorize('cashier', 'pharmacist', 'admin'), idempotent, validate({ body: CreateSaleSchema }), controller.create);
```

The frontend generates the key once per checkout: `crypto.randomUUID()`.

## 7.4 Rate limiting

Limit requests per client to protect against abuse, brute-force login attempts and runaway scripts. Return `429` with `Retry-After`. Use a shared store (Redis) when running several API instances. Implementation in section 16.

## 7.5 HTTP caching for APIs

```ts
// Public, rarely-changing data: let browsers and CDNs cache
res.set('Cache-Control', 'public, max-age=60, stale-while-revalidate=300');

// Private data: never cache in shared caches
res.set('Cache-Control', 'private, no-store');
```

**ETags** let clients ask "has this changed?" and get a cheap `304 Not Modified`. Express sets weak ETags on responses automatically; clients send `If-None-Match`.

## 7.6 Concurrency control: optimistic locking

Two pharmacists edit the same medicine at once; the second save silently overwrites the first. Prevent it with a `version` column (or `updatedAt`):

```ts
// Client sends the version it loaded: PATCH /v1/medicines/7  { "price": 550, "version": 4 }
const updated = await db.update(medicines)
  .set({ price: input.price, version: sql`${medicines.version} + 1` })
  .where(and(eq(medicines.id, id), eq(medicines.version, input.version)))
  .returning();
if (updated.length === 0) throw new AppError(409, 'This medicine was changed by someone else. Reload and try again.', 'VERSION_CONFLICT');
```

The HTTP-standard equivalent uses `ETag` + `If-Match` headers and returns `412 Precondition Failed`.

## 7.7 Webhooks

**What it is:** A webhook is an HTTP request **another system sends to you** when something happens: M-Pesa calls `/v1/payments/mpesa/callback` when a customer pays.

**Receiving webhooks safely:**

1. **Verify** the sender: signature (HMAC of the raw body with a shared secret), a secret in the URL or header, and/or an IP allowlist.
2. **Respond fast** (`200` within a few seconds); do the real work in a background job.
3. **Be idempotent:** providers retry, so the same event may arrive twice. Store processed event IDs.
4. **Log** every payload for troubleshooting.

```ts
import { createHmac, timingSafeEqual } from 'node:crypto';

function verifySignature(rawBody: Buffer, signature: string, secret: string) {
  const expected = createHmac('sha256', secret).update(rawBody).digest('hex');
  return signature.length === expected.length && timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
}
```

**Sending webhooks** (e.g. notifying a partner clinic of a filled prescription): sign the payload, retry with exponential backoff, and let receivers verify the signature.

## 7.8 REST vs GraphQL vs gRPC vs real-time

| Style | How | Strengths | Use when |
| --- | --- | --- | --- |
| **REST** | Resources + HTTP methods, JSON | Simple, cacheable, universal | Default for public and frontend APIs |
| **GraphQL** | One endpoint; the client asks for exactly the fields it needs | No over/under-fetching; strongly typed schema | Many different clients with different data needs |
| **gRPC** | Binary Protocol Buffers over HTTP/2 | Very fast, typed contracts, streaming | Internal service-to-service calls |
| **tRPC / oRPC** | Call backend functions with end-to-end TS types | Zero schema duplication in TS monorepos | Next.js + Node in one TypeScript repo |
| **WebSockets** | Persistent two-way connection | Real-time both directions | Live POS updates, chat |
| **Server-Sent Events (SSE)** | Server pushes over a normal HTTP response | Simple one-way real-time | Live stock levels, payment status |
| **Webhooks** | Server calls client's URL on events | Event notifications between systems | M-Pesa callbacks |

Real-time implementation: section 15.

## 7.9 API testing and exploration tools

| Tool | Use |
| --- | --- |
| **curl** / **HTTPie** | Quick calls from the terminal |
| **Bruno** | Open-source API client; collections stored as files in your Git repo |
| **Postman** / **Insomnia** | Popular API clients, collections, environments |
| **VS Code REST Client** / `.http` files | Requests next to your code |
| **Hurl** | Plain-text HTTP tests that run in CI |
| **Supertest** | Automated API tests in code (section 17) |

```bash
curl -i http://localhost:4000/v1/medicines?search=amox
curl -X POST http://localhost:4000/v1/sales \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"paymentMethod":"cash","items":[{"medicineId":7,"qty":2}]}'
```
