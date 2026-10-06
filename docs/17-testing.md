# 17. Testing the backend

## 17.1 What to test at each level

| Level | Tests | Speed | PharmaCare examples |
| --- | --- | --- | --- |
| **Unit** | Pure functions and services with fake dependencies | Milliseconds | Receipt number format, FEFO allocation, permission map, Zod schemas |
| **Integration** | HTTP route → middleware → service → **real** test database | Seconds | `POST /v1/sales` deducts stock; cashier gets 403 on price change |
| **Contract** | The API matches its OpenAPI spec | Seconds | Response shapes don't drift |
| **End-to-end** | Frontend + API + DB together | Minutes | Playwright checkout flow (Frontend sheet, section 26) |
| **Load** | Behaviour under traffic | Minutes | 50 cashiers selling at once (k6, autocannon) |

The highest-value tests for an API are **integration tests against a real database**: they catch SQL mistakes, constraint violations and auth gaps that mocks hide.

## 17.2 Vitest setup

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'node',
    setupFiles: ['./tests/setup.ts'],
    fileParallelism: false,          // integration tests share one test DB
    coverage: { provider: 'v8', include: ['src/**'] },
  },
});
```

```bash
npx vitest                 # watch mode
npx vitest run             # once (CI)
npx vitest run --coverage
```

## 17.3 Unit tests

```ts
// src/modules/sales/allocate.ts — pure FEFO allocation logic, easy to test
export function allocateFefo(batches: { id: number; quantity: number; expiresAt: string }[], qty: number) {
  const sorted = batches.filter((b) => b.quantity > 0).toSorted((a, b) => a.expiresAt.localeCompare(b.expiresAt));
  const lines: { batchId: number; qty: number }[] = [];
  let remaining = qty;
  for (const b of sorted) {
    if (remaining === 0) break;
    const take = Math.min(b.quantity, remaining);
    lines.push({ batchId: b.id, qty: take });
    remaining -= take;
  }
  return { lines, shortBy: remaining };
}
```

```ts
// src/modules/sales/allocate.test.ts
import { describe, it, expect } from 'vitest';
import { allocateFefo } from './allocate.ts';

describe('allocateFefo', () => {
  const batches = [
    { id: 1, quantity: 5, expiresAt: '2027-06-30' },
    { id: 2, quantity: 3, expiresAt: '2027-01-31' },   // expires first
  ];

  it('takes from the earliest-expiring batch first', () => {
    expect(allocateFefo(batches, 2).lines).toEqual([{ batchId: 2, qty: 2 }]);
  });

  it('spans batches when one is not enough', () => {
    expect(allocateFefo(batches, 6).lines).toEqual([{ batchId: 2, qty: 3 }, { batchId: 1, qty: 3 }]);
  });

  it('reports shortage', () => {
    expect(allocateFefo(batches, 10).shortBy).toBe(2);
  });
});
```

**Mocks and fakes** for services with external dependencies:

```ts
import { vi } from 'vitest';
const sms = { send: vi.fn().mockResolvedValue({ id: 'msg_1' }) };
await sendReceipt({ saleId: 1, phone: '+255754000000' }, { sms, saleRepository: fakeRepo });
expect(sms.send).toHaveBeenCalledWith('+255754000000', expect.stringContaining('RCP-'));
```

## 17.4 Integration tests with Supertest

**What it is:** Supertest sends real HTTP requests to your Express app in memory (no port needed), so you test routing, middleware, validation, auth, the service and the database together.

**Test database options:**

| Option | How |
| --- | --- |
| **Neon branch per CI run** | Create a branch from a seeded base, run tests, delete it (fast, real Neon) |
| **Docker / Testcontainers** | Start a throwaway `postgres:17` container for the test run |
| **PGlite** | PostgreSQL compiled to WebAssembly, in-process (very fast; check extension support) |
| **mongodb-memory-server** | In-memory MongoDB for Mongoose tests |

```ts
// tests/setup.ts
import { beforeAll, beforeEach, afterAll } from 'vitest';
import { migrate } from 'drizzle-orm/node-postgres/migrator';
import { sql } from 'drizzle-orm';
import { db, pool } from '../src/db/postgres.ts';

beforeAll(async () => {
  await migrate(db, { migrationsFolder: './drizzle' });   // DATABASE_URL points to the TEST database
});

beforeEach(async () => {
  await db.execute(sql`TRUNCATE sale_items, sales, batches, medicines, categories, refresh_tokens, users RESTART IDENTITY CASCADE`);
});

afterAll(async () => { await pool.end(); });
```

```ts
// tests/factories.ts — create test data quickly
export async function createUser(role: Role = 'cashier') {
  const [u] = await db.insert(users).values({ name: `Test ${role}`, email: `${role}-${crypto.randomUUID()}@test.tz`, passwordHash: await argon2.hash('password123'), role }).returning();
  return { user: u!, token: await signAccessToken(u!) };
}

export async function createMedicineWithStock(stock: number, price = 500) {
  const [cat] = await db.insert(categories).values({ name: `Cat ${crypto.randomUUID()}` }).returning();
  const [med] = await db.insert(medicines).values({ categoryId: cat!.id, name: 'Amoxicillin 500mg', slug: `amox-${crypto.randomUUID()}`, price }).returning();
  await db.insert(batches).values({ medicineId: med!.id, batchNo: `B-${crypto.randomUUID()}`, quantity: stock, costPrice: 300, expiresAt: '2030-01-01' });
  return med!;
}
```

```ts
// tests/sales.test.ts
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import { createApp } from '../src/app.ts';

const app = createApp();

describe('POST /v1/sales', () => {
  it('creates a cash sale and deducts stock', async () => {
    const { token } = await createUser('cashier');
    const med = await createMedicineWithStock(10, 500);

    const res = await request(app)
      .post('/v1/sales')
      .set('Authorization', `Bearer ${token}`)
      .set('Idempotency-Key', crypto.randomUUID())
      .send({ paymentMethod: 'cash', items: [{ medicineId: med.id, qty: 3 }] });

    expect(res.status).toBe(201);
    expect(res.body.data.total).toBe(1500);
    const [batch] = await db.select().from(batches).where(eq(batches.medicineId, med.id));
    expect(batch!.quantity).toBe(7);
  });

  it('returns 409 and changes nothing when stock is insufficient', async () => {
    const { token } = await createUser('cashier');
    const med = await createMedicineWithStock(2);

    const res = await request(app).post('/v1/sales').set('Authorization', `Bearer ${token}`).set('Idempotency-Key', crypto.randomUUID())
      .send({ paymentMethod: 'cash', items: [{ medicineId: med.id, qty: 5 }] });

    expect(res.status).toBe(409);
    expect(res.body.type).toBe('insufficient_stock');
    const [batch] = await db.select().from(batches).where(eq(batches.medicineId, med.id));
    expect(batch!.quantity).toBe(2);          // transaction rolled back
  });

  it('rejects unauthenticated requests', async () => {
    await request(app).post('/v1/sales').send({}).expect(401);
  });

  it('validates input', async () => {
    const { token } = await createUser('cashier');
    const res = await request(app).post('/v1/sales').set('Authorization', `Bearer ${token}`).set('Idempotency-Key', crypto.randomUUID())
      .send({ paymentMethod: 'bitcoin', items: [] });
    expect(res.status).toBe(422);
    expect(res.body.errors).toHaveProperty('paymentMethod');
  });
});

describe('PATCH /v1/medicines/:id', () => {
  it('forbids cashiers from changing prices', async () => {
    const { token } = await createUser('cashier');
    const med = await createMedicineWithStock(5);
    await request(app).patch(`/v1/medicines/${med.id}`).set('Authorization', `Bearer ${token}`).send({ price: 1 }).expect(403);
  });
});
```

**Concurrency test** — prove two parallel sales cannot oversell:

```ts
it('never oversells under concurrent requests', async () => {
  const med = await createMedicineWithStock(5);
  const { token } = await createUser('cashier');
  const sell = () => request(app).post('/v1/sales').set('Authorization', `Bearer ${token}`).set('Idempotency-Key', crypto.randomUUID())
    .send({ paymentMethod: 'cash', items: [{ medicineId: med.id, qty: 3 }] });

  const results = await Promise.all([sell(), sell()]);
  expect(results.map((r) => r.status).toSorted()).toEqual([201, 409]);   // only one succeeds
});
```

## 17.5 Mocking external HTTP calls

Use **MSW** (`msw/node`) or **nock** to fake M-Pesa and SMS providers in tests, including failures and timeouts:

```ts
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

const server = setupServer(
  http.post('https://mpesa.example/stkpush', () => HttpResponse.json({ requestId: 'req_123', responseCode: '0' })),
);
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));   // fail if code calls anything unmocked
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

## 17.6 Tests in CI

```yaml
# .github/workflows/ci.yml (excerpt)
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17
        env: { POSTGRES_PASSWORD: test, POSTGRES_DB: pharmacare_test }
        ports: ['5432:5432']
        options: --health-cmd pg_isready --health-interval 5s --health-retries 10
      redis:
        image: redis:7
        ports: ['6379:6379']
    env:
      DATABASE_URL: postgresql://postgres:test@localhost:5432/pharmacare_test
      REDIS_URL: redis://localhost:6379
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm run typecheck && npm run lint
      - run: npx vitest run --coverage
```

**What to always test:** every money path (sale, void, refund, payment callback), every permission rule, validation errors, and every bug you fix (write the failing test first).
