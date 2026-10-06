# 22. Quick reference

## 22.1 Commands

| Command | Does | Section |
| --- | --- | --- |
| `nvm install --lts` / `nvm use 22` | Install / switch Node version | 2.2 |
| `npm run dev` (`tsx watch --env-file=.env src/server.ts`) | Run the API with reload | 2.5 |
| `node --inspect dist/server.js` | Debug | 3.8 |
| `npx drizzle-kit generate` / `migrate` / `studio` | Migrations / DB browser | 10.4 |
| `psql "$DATABASE_URL_UNPOOLED"` | PostgreSQL shell | 8.3 |
| `neonctl branches create --name x` | New Neon branch | 8.2 |
| `pg_dump -Fc -f db.dump "$URL"` / `pg_restore` | Backup / restore Postgres | 9.9 |
| `mongosh "$MONGODB_URI"` | MongoDB shell | 11.2 |
| `mongodump` / `mongorestore` | Backup / restore MongoDB | 12.7 |
| `docker compose up -d` / `logs -f api` / `down` | Local stack | 20.2 |
| `docker build -t api .` / `docker run -p 4000:4000 api` | Build / run image | 20.1 |
| `npx vitest run --coverage` | Tests | 17.2 |
| `npx autocannon -c 50 -d 20 URL` | Load test | 3.8 |
| `pm2 start dist/server.js -i max` / `pm2 reload` | Run / zero-downtime reload on a VPS | 20.6 |
| `npm audit` | Dependency vulnerabilities | 16.7 |
| `npx openapi-typescript URL -o api.d.ts` | Typed client for the frontend | 21.7 |

## 22.2 Express skeleton

```ts
const app = express();
app.set('trust proxy', 1);
app.use(pinoHttp({ logger }), helmet(), cors({ origin: env.CORS_ORIGINS, credentials: true }), compression(),
        express.json({ limit: '1mb' }), cookieParser(), apiLimiter);
app.get('/health/live', (_req, res) => res.json({ status: 'ok' }));
app.use('/v1', apiRouter);
app.use(notFound);
app.use(errorHandler);           // last

router.post('/', authenticate, authorize('pharmacist', 'admin'), validate({ body: Schema }), controller.create);
```

## 22.3 Status codes cheat

`200` OK · `201` created · `202` accepted (async) · `204` no content · `400` bad JSON · `401` not logged in · `403` not allowed · `404` not found · `409` conflict (stock, duplicate, version) · `422` validation · `429` rate limited · `500` bug · `502/504` upstream failed/timed out.

## 22.4 SQL cheat

```sql
SELECT m.name, SUM(b.quantity) AS stock FROM medicines m
LEFT JOIN batches b ON b.medicine_id = m.id AND b.expires_at > current_date
WHERE m.deleted_at IS NULL GROUP BY m.id HAVING SUM(b.quantity) < 20 ORDER BY stock;

INSERT INTO categories (name) VALUES ('Vitamins') ON CONFLICT (name) DO NOTHING RETURNING *;
UPDATE batches SET quantity = quantity - 2 WHERE id = 42 AND quantity >= 2;          -- atomic decrement
SELECT ... FOR UPDATE;                                                               -- lock rows in a transaction
WITH x AS (...) SELECT ...;                                                          -- CTE
SUM(total) OVER (ORDER BY day)                                                       -- running total
CREATE INDEX CONCURRENTLY idx ON batches (medicine_id, expires_at) WHERE quantity > 0;
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

## 22.5 Drizzle cheat

```ts
await db.select().from(medicines).where(and(eq(medicines.categoryId, 1), ilike(medicines.name, '%amox%'))).orderBy(asc(medicines.name)).limit(20);
await db.query.medicines.findFirst({ where: eq(medicines.slug, slug), with: { category: true } });
await db.insert(medicines).values(data).returning();
await db.update(medicines).set({ price: 550 }).where(eq(medicines.id, 7)).returning();
await db.delete(batches).where(eq(batches.id, 9));
await db.transaction(async (tx) => { /* tx.select().for('update') ... */ });
await db.execute(sql`SELECT ... WHERE id = ${id}`);    // parameterised raw SQL
```

## 22.6 MongoDB / Mongoose cheat

```js
db.col.find({ field: { $gte: 4 }, tags: { $in: ['a'] } }, { name: 1 }).sort({ at: -1 }).limit(20);
db.col.updateOne({ _id }, { $set: { a: 1 }, $inc: { n: 1 }, $push: { list: x } }, { upsert: true });
db.col.aggregate([{ $match: {} }, { $group: { _id: '$k', total: { $sum: 1 } } }, { $sort: { total: -1 } }]);
db.col.createIndex({ a: 1, b: -1 });  db.col.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });
```

```ts
await Model.create(doc);
await Model.find(filter).sort({ at: -1 }).limit(20).lean();
await Model.updateOne(filter, { $set: patch });
await Model.aggregate(pipeline);
```

## 22.7 Redis and BullMQ cheat

```ts
await redis.set(key, JSON.stringify(v), 'EX', 300);  const v = JSON.parse((await redis.get(key)) ?? 'null');
await redis.del(key);  await redis.incr(`count:${id}`);
await queue.add('job-name', { id }, { jobId: `unique-${id}`, attempts: 5, backoff: { type: 'exponential', delay: 10_000 } });
new Worker('queue', async (job) => { /* idempotent work */ }, { connection: redis, concurrency: 10 });
await queue.upsertJobScheduler('daily', { pattern: '0 21 * * *', tz: 'Africa/Dar_es_Salaam' }, { name: 'daily-report' });
```

## 22.8 Decision guide

| Question | Answer |
| --- | --- |
| Postgres or Mongo for this data? | Money, stock, relations, constraints → **Postgres**. Flexible, high-volume logs/events/content → **Mongo** (or Postgres JSONB) |
| Pooled or direct Neon URL? | App → **pooled** (`-pooler`). Migrations, `LISTEN/NOTIFY`, admin → **direct** |
| Do this in the request or a job? | Under \~200 ms and needed for the response → request. Slow, external, retryable → **job** |
| How do I prevent double charges? | **Idempotency-Key** + unique constraints + idempotent webhook handling |
| How do I stop overselling? | Transaction + `SELECT ... FOR UPDATE` + `CHECK (quantity >= 0)` |
| Cache it? | Public, read-heavy, tolerates seconds of staleness → Redis with TTL + delete on write. Stock at checkout → never |
| Session or JWT? | Next.js BFF: tokens in HttpOnly cookies on the web domain; API: 15-min JWT + rotating refresh token |
| Monolith or microservices? | **Modular monolith** until there is a concrete reason to split |
| Where to deploy? | PaaS (Railway/Render/Fly) to start; VPS + Docker/Coolify for cost control; keep API in the same region as Neon/Atlas |
