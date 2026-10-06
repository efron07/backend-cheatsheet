# 10. PostgreSQL from Node.js: drivers, Drizzle and transactions

## 10.1 Choosing a driver and query layer

| Option | What it is | When |
| --- | --- | --- |
| `pg` (node-postgres) | The classic PostgreSQL driver with a connection pool | Long-running Express servers (Docker, VPS, Railway) |
| `postgres` (postgres.js) | Fast modern driver with tagged-template SQL | Same, alternative style |
| `@neondatabase/serverless` | Neon driver over HTTP (one-shot queries) or WebSockets (`Pool`, full transactions) | Serverless/edge functions, Vercel |
| **Drizzle ORM** | Type-safe, SQL-like query builder + migrations | **Used here**: you write near-SQL with full types |
| **Prisma** | Schema-first ORM with generated client | Popular alternative; great DX |
| **Kysely** | Type-safe SQL query builder (no ORM) | If you want pure SQL building |
| Raw SQL | `pool.query('SELECT ... WHERE id = $1', [id])` | Reports, complex queries |

**Important:** Neon's HTTP driver cannot run interactive transactions (read, decide, then write). For the sale flow (section 10.5), use `pg` or the Neon WebSocket `Pool`.

## 10.2 Setup: Drizzle + node-postgres on Neon

```bash
npm install drizzle-orm pg
npm install -D drizzle-kit @types/pg
```

```ts
// src/db/postgres.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import { env } from '../config/env.ts';
import * as schema from './schema.ts';

export const pool = new Pool({
  connectionString: env.DATABASE_URL,     // Neon POOLED url (-pooler), sslmode=require
  max: 10,                                 // connections per API instance
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
});

export const db = drizzle(pool, { schema, casing: 'snake_case' });   // camelCase in TS ↔ snake_case in SQL

// For serverless instead:
// import { neon } from '@neondatabase/serverless';
// import { drizzle } from 'drizzle-orm/neon-http';
// export const db = drizzle(neon(env.DATABASE_URL), { schema });
```

## 10.3 Schema in TypeScript

```ts
// src/db/schema.ts
import { pgTable, pgEnum, bigint, text, integer, boolean, date, timestamp, index, check, uniqueIndex } from 'drizzle-orm/pg-core';
import { relations, sql } from 'drizzle-orm';

export const userRole = pgEnum('user_role', ['admin', 'pharmacist', 'cashier']);
export const paymentMethod = pgEnum('payment_method', ['cash', 'mpesa', 'airtel', 'card']);
export const saleStatus = pgEnum('sale_status', ['pending', 'paid', 'voided']);

const id = () => bigint({ mode: 'number' }).primaryKey().generatedAlwaysAsIdentity();
const timestamps = {
  createdAt: timestamp({ withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp({ withTimezone: true }).notNull().defaultNow().$onUpdate(() => new Date()),
};

export const users = pgTable('users', {
  id: id(),
  name: text().notNull(),
  email: text().notNull().unique(),
  passwordHash: text().notNull(),
  role: userRole().notNull().default('cashier'),
  isActive: boolean().notNull().default(true),
  ...timestamps,
});

export const categories = pgTable('categories', { id: id(), name: text().notNull().unique() });

export const medicines = pgTable('medicines', {
  id: id(),
  categoryId: bigint({ mode: 'number' }).notNull().references(() => categories.id, { onDelete: 'restrict' }),
  name: text().notNull(),
  slug: text().notNull().unique(),
  price: integer().notNull(),
  requiresPrescription: boolean().notNull().default(false),
  reorderLevel: integer().notNull().default(20),
  version: integer().notNull().default(1),
  deletedAt: timestamp({ withTimezone: true }),
  ...timestamps,
}, (t) => [
  check('price_non_negative', sql`${t.price} >= 0`),
  index('idx_medicines_category').on(t.categoryId),
]);

export const batches = pgTable('batches', {
  id: id(),
  medicineId: bigint({ mode: 'number' }).notNull().references(() => medicines.id, { onDelete: 'cascade' }),
  batchNo: text().notNull().unique(),
  quantity: integer().notNull(),
  costPrice: integer().notNull(),
  expiresAt: date({ mode: 'string' }).notNull(),
  receivedAt: timestamp({ withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  check('quantity_non_negative', sql`${t.quantity} >= 0`),
  index('idx_batches_fefo').on(t.medicineId, t.expiresAt).where(sql`${t.quantity} > 0`),
]);

export const sales = pgTable('sales', {
  id: id(),
  receiptNo: text().notNull().unique(),
  cashierId: bigint({ mode: 'number' }).notNull().references(() => users.id),
  customerId: bigint({ mode: 'number' }),
  total: integer().notNull(),
  paymentMethod: paymentMethod().notNull(),
  status: saleStatus().notNull().default('pending'),
  createdAt: timestamp({ withTimezone: true }).notNull().defaultNow(),
}, (t) => [index('idx_sales_created_at').on(t.createdAt)]);

export const saleItems = pgTable('sale_items', {
  id: id(),
  saleId: bigint({ mode: 'number' }).notNull().references(() => sales.id, { onDelete: 'cascade' }),
  medicineId: bigint({ mode: 'number' }).notNull().references(() => medicines.id),
  batchId: bigint({ mode: 'number' }).notNull().references(() => batches.id),
  qty: integer().notNull(),
  price: integer().notNull(),
});

// Relations (for the relational query API: db.query.medicines.findMany({ with: { category: true } }))
export const medicineRelations = relations(medicines, ({ one, many }) => ({
  category: one(categories, { fields: [medicines.categoryId], references: [categories.id] }),
  batches: many(batches),
}));
export const saleRelations = relations(sales, ({ many, one }) => ({
  items: many(saleItems),
  cashier: one(users, { fields: [sales.cashierId], references: [users.id] }),
}));
export const saleItemRelations = relations(saleItems, ({ one }) => ({
  sale: one(sales, { fields: [saleItems.saleId], references: [sales.id] }),
  medicine: one(medicines, { fields: [saleItems.medicineId], references: [medicines.id] }),
}));

export type Medicine = typeof medicines.$inferSelect;
export type NewMedicine = typeof medicines.$inferInsert;
```

## 10.4 Migrations

```ts
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit';
export default defineConfig({
  schema: './src/db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  casing: 'snake_case',
  dbCredentials: { url: process.env.DATABASE_URL_UNPOOLED! },   // direct URL for DDL
});
```

```bash
npx drizzle-kit generate --name add_barcode    # diff schema.ts → new SQL file in ./drizzle
npx drizzle-kit migrate                         # apply pending migrations
npx drizzle-kit studio                          # browse data in a web UI
npx drizzle-kit push                            # sync schema without migration files (prototyping only)
```

**Migration rules:**

- Commit migration files; review the generated SQL before applying.
- Never edit a migration that has run in production; add a new one.
- Make changes **backward-compatible** for zero-downtime deploys: add a nullable column → deploy code that writes it → backfill → make it `NOT NULL`. Rename = add new + copy + switch + drop old later.
- Test risky migrations on a **Neon branch** of production first.
- Run migrations as a separate deploy step, not on every app start (section 20).

**Seeding:** a `scripts/seed.ts` that inserts categories, an admin user and sample medicines; run with `npx tsx scripts/seed.ts`.

## 10.5 Repositories and queries

```ts
// src/modules/medicines/medicine.repository.ts
import { and, asc, desc, eq, gt, ilike, isNull, sql, count } from 'drizzle-orm';
import { db } from '../../db/postgres.ts';
import { medicines, batches, categories } from '../../db/schema.ts';

const SORTS = {
  name: asc(medicines.name), '-name': desc(medicines.name),
  price: asc(medicines.price), '-price': desc(medicines.price),
} as const;                                                 // whitelist — never interpolate user input

const stockExpr = sql<number>`coalesce(sum(${batches.quantity}) filter (where ${batches.expiresAt} > current_date), 0)::int`;

export const medicineRepository = {
  async list(q: ListMedicinesQuery) {
    const where = and(
      isNull(medicines.deletedAt),
      q.search ? ilike(medicines.name, `%${q.search}%`) : undefined,   // parameterised: safe from SQL injection
      q.categoryId ? eq(medicines.categoryId, q.categoryId) : undefined,
    );

    const [rows, [{ total }]] = await Promise.all([
      db.select({
          id: medicines.id, name: medicines.name, slug: medicines.slug, price: medicines.price,
          requiresPrescription: medicines.requiresPrescription, category: categories.name, stock: stockExpr,
        })
        .from(medicines)
        .innerJoin(categories, eq(categories.id, medicines.categoryId))
        .leftJoin(batches, eq(batches.medicineId, medicines.id))
        .where(where)
        .groupBy(medicines.id, categories.name)
        .having(q.inStock ? gt(stockExpr, 0) : undefined)
        .orderBy(SORTS[q.sort])
        .limit(q.limit)
        .offset((q.page - 1) * q.limit),
      db.select({ total: count() }).from(medicines).where(where),
    ]);

    return { data: rows, meta: { page: q.page, limit: q.limit, total, totalPages: Math.ceil(total / q.limit) } };
  },

  findBySlug: (slug: string) =>
    db.query.medicines.findFirst({
      where: and(eq(medicines.slug, slug), isNull(medicines.deletedAt)),
      with: { category: true },
    }),

  findById: (id: number) => db.query.medicines.findFirst({ where: eq(medicines.id, id) }),

  create: async (data: NewMedicine) => (await db.insert(medicines).values(data).returning())[0]!,

  update: async (id: number, data: Partial<NewMedicine>) =>
    (await db.update(medicines).set(data).where(eq(medicines.id, id)).returning())[0]!,

  softDelete: (id: number) => db.update(medicines).set({ deletedAt: new Date() }).where(eq(medicines.id, id)),
};
```

**Avoid N+1 queries:** do not loop over 50 medicines and query each one's batches. Use a join, `with: { batches: true }`, or one `WHERE medicine_id IN (...)` query.

**Raw SQL when it is clearer** (still parameterised):

```ts
const result = await db.execute(sql`
  SELECT created_at::date AS day, SUM(total)::int AS revenue
  FROM sales WHERE status = 'paid' AND created_at >= ${from} GROUP BY 1 ORDER BY 1`);
```

## 10.6 The sale transaction (stock, FEFO, locking)

The most important function in PharmaCare: everything succeeds together or nothing happens.

```ts
// src/modules/sales/sale.service.ts
import { and, asc, eq, gt, inArray } from 'drizzle-orm';
import { db } from '../../db/postgres.ts';
import { batches, medicines, sales, saleItems } from '../../db/schema.ts';
import { AppError, InsufficientStockError } from '../../lib/errors.ts';
import { bus } from '../../events/bus.ts';

export async function createSale(input: CreateSaleInput, cashier: AuthUser) {
  const sale = await db.transaction(async (tx) => {
    // 1. Load medicines and their CURRENT prices from the DB (never trust client prices)
    const ids = input.items.map((i) => i.medicineId).sort((a, b) => a - b);   // consistent lock order
    const meds = await tx.select().from(medicines).where(inArray(medicines.id, ids));
    const byId = new Map(meds.map((m) => [m.id, m]));

    const lines: { medicineId: number; batchId: number; qty: number; price: number }[] = [];

    for (const item of input.items.toSorted((a, b) => a.medicineId - b.medicineId)) {
      const med = byId.get(item.medicineId);
      if (!med || med.deletedAt) throw new AppError(404, `Medicine ${item.medicineId} not found`, 'NOT_FOUND');
      if (med.requiresPrescription && !input.prescriptionId) throw new AppError(422, `${med.name} needs a prescription`, 'PRESCRIPTION_REQUIRED');

      // 2. Lock available batches, earliest expiry first (FEFO)
      const available = await tx.select().from(batches)
        .where(and(eq(batches.medicineId, med.id), gt(batches.quantity, 0), gt(batches.expiresAt, new Date().toISOString().slice(0, 10))))
        .orderBy(asc(batches.expiresAt))
        .for('update');

      let remaining = item.qty;
      for (const batch of available) {
        if (remaining === 0) break;
        const take = Math.min(batch.quantity, remaining);
        await tx.update(batches).set({ quantity: batch.quantity - take }).where(eq(batches.id, batch.id));
        lines.push({ medicineId: med.id, batchId: batch.id, qty: take, price: med.price });
        remaining -= take;
      }
      if (remaining > 0) throw new InsufficientStockError(med.name, item.qty - remaining);   // whole transaction rolls back
    }

    // 3. Record the sale and its lines
    const total = lines.reduce((s, l) => s + l.qty * l.price, 0);
    const [created] = await tx.insert(sales).values({
      receiptNo: await nextReceiptNo(tx),
      cashierId: cashier.id,
      customerId: input.customerId,
      total,
      paymentMethod: input.paymentMethod,
      status: input.paymentMethod === 'cash' ? 'paid' : 'pending',
    }).returning();

    await tx.insert(saleItems).values(lines.map((l) => ({ ...l, saleId: created!.id })));
    return { ...created!, items: lines };
  });

  // 4. After COMMIT: side effects (audit log, low-stock checks, M-Pesa STK push via a queue)
  bus.emit('sale.completed', sale);
  return sale;
}
```

**Why each step matters:**

- Prices come from the database, not the request.
- `FOR UPDATE` stops two cashiers selling the same last boxes.
- Locking medicines in sorted order avoids deadlocks.
- The `CHECK (quantity >= 0)` constraint is a final safety net.
- Side effects run **after** commit, so a rolled-back sale never sends an SMS.

## 10.7 Prisma in one glance (alternative)

```text
model Medicine {
  id         Int      @id @default(autoincrement())
  name       String
  slug       String   @unique
  price      Int
  categoryId Int
  category   Category @relation(fields: [categoryId], references: [id])
  batches    Batch[]
  @@map("medicines")
}
```

```ts
const meds = await prisma.medicine.findMany({ where: { name: { contains: 'amox', mode: 'insensitive' } }, include: { category: true } });
await prisma.$transaction(async (tx) => { /* ... */ });
```

```bash
npx prisma migrate dev --name init; npx prisma generate; npx prisma studio
```
