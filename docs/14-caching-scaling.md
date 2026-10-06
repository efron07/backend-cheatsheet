# 14. Caching, performance and scaling

## 14.1 Where caching happens

| Layer | What is cached | Tool |
| --- | --- | --- |
| Browser | Responses with `Cache-Control` | HTTP headers |
| CDN / reverse proxy | Public GET responses, static files | Cloudflare, Nginx |
| Next.js | Rendered pages and data (`'use cache'`) | Frontend sheet, section 19 |
| **Application cache** | Expensive query results, computed data | **Redis** |
| Database | Recently used pages in memory, query plans | PostgreSQL itself |

## 14.2 Redis

**What it is:** Redis is an in-memory data store: extremely fast reads/writes (well under a millisecond). Used for caching, sessions, rate-limit counters, queues (BullMQ), pub/sub and locks. Managed options: **Upstash** (serverless, HTTP), Redis Cloud, Railway; or Docker locally. **Valkey** is an open-source fork with the same commands.

```bash
docker run -d --name redis -p 6379:6379 redis:7
npm install ioredis
```

```ts
// src/lib/redis.ts
import { Redis } from 'ioredis';
import { env } from '../config/env.ts';
export const redis = new Redis(env.REDIS_URL, { maxRetriesPerRequest: null });   // null required by BullMQ
```

**Core data structures:**

```bash
SET medicine:amoxicillin-500mg '{"id":7,...}' EX 300     # string with 5-minute expiry
GET medicine:amoxicillin-500mg
INCR rate:login:102.68.1.20                              # counters
HSET cart:user:3 7 2                                     # hash: medicineId → qty
LPUSH / RPOP queue                                       # lists
SADD online:branches dodoma                              # sets
ZADD top-sellers 152 "7"                                 # sorted set: leaderboards
TTL key; DEL key; KEYS is slow — use SCAN in production
```

## 14.3 Caching patterns

**Cache-aside (lazy loading)** — the most common: check the cache; on a miss, load from the database and store it.

```ts
// src/lib/cache.ts
export const cache = {
  async getOrSet<T>(key: string, ttlSeconds: number, load: () => Promise<T>): Promise<T> {
    const hit = await redis.get(key);
    if (hit) return JSON.parse(hit) as T;
    const value = await load();
    if (value !== undefined && value !== null) await redis.set(key, JSON.stringify(value), 'EX', ttlSeconds);
    return value;
  },
  del: (...keys: string[]) => redis.del(...keys),
};

// usage in a service
const medicine = await cache.getOrSet(`medicine:${slug}`, 300, () => medicineRepository.findBySlug(slug));
```

**Invalidation** — "there are only two hard things: cache invalidation and naming things":

- **TTL** — every entry expires (5 minutes). Simple; data may be stale up to the TTL.
- **Delete on write** — when a medicine changes, delete `medicine:<slug>` and list keys (shown in section 5.3).
- **Versioned keys** — `medicines:v42:page:1`; bump the version number to invalidate a whole group at once.
- **Write-through** — update the cache at the same time as the database.

**Pitfalls:**

- **Cache stampede:** a hot key expires and 500 requests hit the database at once. Fix: add random jitter to TTLs, use a short lock so only one request reloads, or serve stale data while refreshing.
- **Never cache per-user private data under shared keys.** Include the user ID in the key, or do not cache it.
- **Never cache stock levels at the point of sale** — the sale transaction must read live data (section 10.6).

## 14.4 What PharmaCare caches

| Data | TTL | Invalidated by |
| --- | --- | --- |
| Public medicine detail by slug | 5 min | Price/detail update |
| Category list | 1 hour | Category change |
| Public catalogue pages | 2 min | Any medicine change (versioned key) |
| Daily dashboard numbers | 1 min | Time |
| M-Pesa OAuth access token | Until \~1 min before expiry | Expiry |
| Live stock at checkout | **Never cached** | — |

## 14.5 Response performance

- **Compression:** `compression()` middleware (or the reverse proxy) gzips/brotlis JSON — often 70–90% smaller.
- **Pagination and field selection** keep payloads small.
- **Avoid N+1 queries** and add indexes (sections 9–10). Most slow APIs are slow databases.
- **Run independent work in parallel** (`Promise.all`).
- **Move slow work out of the request** (queues, section 15): respond `202 Accepted` and finish in the background.
- **Keep connections warm:** reuse DB pools; HTTP keep-alive for outbound calls (Node's `fetch` does this).
- **Measure** with load tests (`autocannon`, k6) and APM traces (section 18) before optimising.

## 14.6 Scaling

**Vertical scaling** = a bigger server (more CPU/RAM). Simple, has a ceiling.

**Horizontal scaling** = more copies of the API behind a **load balancer**. Requires the API to be **stateless**:

- No sessions or caches in process memory — use Redis.
- No files on local disk — use object storage (R2/S3).
- Scheduled jobs run once (a single scheduler or BullMQ's repeatable jobs), not on every copy.
- Use all CPU cores with PM2 cluster mode or multiple containers.

```
            +-------------------+
  clients → |  Load balancer    | → API #1 ─┐
            |  (Nginx / cloud)  | → API #2 ─┼→ Redis, Postgres (Neon), MongoDB (Atlas)
            +-------------------+ → API #3 ─┘
```

**Database scaling** follows section 9.12 (indexes → caching → read replicas → partitioning → sharding).

**Other techniques:** a **CDN** for public GET endpoints and files; **rate limiting** to protect capacity; **circuit breakers** around flaky external services (stop calling M-Pesa for 30 s after repeated failures and fail fast); **graceful degradation** (if MongoDB audit logging is down, still complete sales and log the failure).
