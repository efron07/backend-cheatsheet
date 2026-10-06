# 9. PostgreSQL 2: performance and DBA skills

## 9.1 Indexes

**What it is:** An index is a sorted lookup structure, like the index at the back of a book. Without one, PostgreSQL reads every row (a **sequential scan**). With one, it jumps straight to the matching rows. Indexes speed up reads but slow down writes slightly and use disk, so add them for real query patterns.

```sql
-- B-tree (default): equality, ranges, sorting
CREATE INDEX idx_batches_medicine_expiry ON batches (medicine_id, expires_at);   -- composite: FEFO lookups
CREATE INDEX idx_sales_created_at ON sales (created_at DESC);

-- Partial: only the rows you query — smaller and faster
CREATE INDEX idx_batches_available ON batches (medicine_id, expires_at) WHERE quantity > 0;
CREATE INDEX idx_medicines_active ON medicines (name) WHERE deleted_at IS NULL;

-- Expression index: for case-insensitive lookups
CREATE INDEX idx_customers_lower_email ON customers (lower(email));

-- GIN: JSONB, arrays, full-text search, trigram (fuzzy) search
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_medicines_name_trgm ON medicines USING gin (name gin_trgm_ops);   -- fast ILIKE '%amox%'

-- Build on a live table without blocking writes
CREATE INDEX CONCURRENTLY idx_sale_items_medicine ON sale_items (medicine_id);
```

| Index type | Good for |
| --- | --- |
| **B-tree** | `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, `LIKE 'abc%'` |
| **GIN** | JSONB containment, arrays, full-text, trigram search |
| **GiST** | Geometry, ranges, nearest-neighbour (PostGIS: nearest branch) |
| **BRIN** | Huge, naturally ordered tables (append-only logs by time) |
| **Hash** | Equality only (rarely needed) |

**Rules:** index foreign keys you join on (Postgres does **not** do this automatically); in composite indexes put equality columns first, then range/sort columns; drop unused indexes (`pg_stat_user_indexes` shows usage).

## 9.2 Reading query plans: `EXPLAIN ANALYZE`

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM batches
WHERE medicine_id = 7 AND quantity > 0 AND expires_at > current_date
ORDER BY expires_at LIMIT 1;
```

| Plan node | Meaning |
| --- | --- |
| `Seq Scan` | Reads the whole table. Fine for small tables; a red flag for big ones |
| `Index Scan` / `Index Only Scan` | Uses an index (only-scan reads no table rows at all) |
| `Bitmap Heap Scan` | Index finds many rows, then fetches them in batches |
| `Nested Loop` / `Hash Join` / `Merge Join` | Join strategies |
| `Sort` with "external merge" | Sort spilled to disk: needs an index or more `work_mem` |

Compare **estimated** vs **actual** rows: big differences mean outdated statistics — run `ANALYZE batches;`.

## 9.3 Transactions, isolation and locking

```sql
BEGIN;
  -- lock the chosen batch row so two cashiers cannot sell the same last units
  SELECT id, quantity FROM batches
  WHERE medicine_id = 7 AND quantity >= 2 AND expires_at > current_date
  ORDER BY expires_at
  LIMIT 1
  FOR UPDATE;                       -- other transactions wanting this row now wait

  UPDATE batches SET quantity = quantity - 2 WHERE id = 42;
  INSERT INTO sales (...) VALUES (...);
  INSERT INTO sale_items (...) VALUES (...);
COMMIT;                             -- or ROLLBACK; on any error
```

**Isolation levels:**

| Level | Prevents | Notes |
| --- | --- | --- |
| **Read Committed** (default) | Dirty reads | Each statement sees data committed before it started. Use `FOR UPDATE` for read-then-write |
| **Repeatable Read** | + non-repeatable reads | Consistent snapshot for the whole transaction; may fail with serialization errors → retry |
| **Serializable** | + all anomalies | Strongest; retry on `40001` errors |

**Locking tips:**

- `FOR UPDATE` locks selected rows; `FOR UPDATE SKIP LOCKED` lets multiple workers take different rows (a simple job queue in SQL).
- Atomic updates avoid read-then-write races: `UPDATE batches SET quantity = quantity - 2 WHERE id = 42 AND quantity >= 2` — if 0 rows changed, stock was insufficient.
- **Deadlocks** happen when two transactions lock rows in opposite order. Always lock in a consistent order (e.g. by `medicine_id`) and keep transactions short. Postgres detects deadlocks and aborts one transaction; retry it.
- **Advisory locks** (`pg_advisory_xact_lock(123)`) coordinate application-level work (only one end-of-day report at a time).

## 9.4 MVCC and VACUUM

**What it is:** PostgreSQL uses **MVCC** (Multi-Version Concurrency Control): an `UPDATE` writes a new row version and marks the old one dead, so readers never block writers. Dead rows must be cleaned up by **VACUUM**; **autovacuum** does this automatically in the background and also refreshes planner statistics (`ANALYZE`).

Watch for **table bloat** on heavily updated tables (batches, sessions): check `pg_stat_user_tables.n_dead_tup` and tune autovacuum for hot tables. Long-running transactions prevent cleanup — never leave a transaction open.

## 9.5 Views, materialized views, functions and triggers

```sql
-- View: a saved query you can SELECT from
CREATE VIEW medicine_stock AS
SELECT m.id, m.name, m.price, COALESCE(SUM(b.quantity) FILTER (WHERE b.expires_at > current_date), 0) AS stock
FROM medicines m LEFT JOIN batches b ON b.medicine_id = m.id
WHERE m.deleted_at IS NULL
GROUP BY m.id;

-- Materialized view: stores the result; refresh on a schedule (fast dashboards)
CREATE MATERIALIZED VIEW daily_sales AS
SELECT created_at::date AS day, payment_method, COUNT(*) AS sales, SUM(total) AS revenue
FROM sales WHERE status = 'paid' GROUP BY 1, 2;
CREATE UNIQUE INDEX ON daily_sales (day, payment_method);
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales;   -- run nightly from a job

-- Trigger: keep updated_at current automatically
CREATE FUNCTION set_updated_at() RETURNS trigger AS $$
BEGIN NEW.updated_at := now(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER medicines_updated_at BEFORE UPDATE ON medicines
FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

Keep most business logic in the application (testable, versioned); use triggers for small, universal rules.

## 9.6 JSONB

**What it is:** Binary JSON stored in a column, queryable and indexable. Good for flexible attributes that differ per row.

```sql
ALTER TABLE medicines ADD COLUMN attributes jsonb NOT NULL DEFAULT '{}';
UPDATE medicines SET attributes = '{"form": "capsule", "strength_mg": 500, "storage": "below 25°C"}' WHERE id = 7;

SELECT name, attributes->>'form' AS form FROM medicines WHERE attributes @> '{"form": "capsule"}';
CREATE INDEX idx_medicines_attributes ON medicines USING gin (attributes);
```

`->` returns JSON, `->>` returns text, `@>` means "contains". Do not put data you filter, join or constrain on every query into JSONB; give it proper columns.

## 9.7 Full-text search

```sql
ALTER TABLE medicines ADD COLUMN search tsvector
  GENERATED ALWAYS AS (to_tsvector('simple', name || ' ' || coalesce(attributes->>'generic', ''))) STORED;
CREATE INDEX idx_medicines_search ON medicines USING gin (search);

SELECT name, ts_rank(search, q) AS rank
FROM medicines, websearch_to_tsquery('simple', 'amoxicillin capsule') q
WHERE search @@ q
ORDER BY rank DESC LIMIT 20;
```

For typo-tolerant search ("amoxicilin"), combine with `pg_trgm` similarity, or use a search engine (Meilisearch, Typesense) for large catalogues.

## 9.8 Roles, privileges and security

```sql
-- The API connects as a least-privileged role, not the owner
CREATE ROLE pharmacare_api LOGIN PASSWORD '...';
GRANT CONNECT ON DATABASE pharmacare TO pharmacare_api;
GRANT USAGE ON SCHEMA public TO pharmacare_api;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO pharmacare_api;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO pharmacare_api;

-- A read-only role for reporting/BI tools
CREATE ROLE reporting_ro LOGIN PASSWORD '...';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO reporting_ro;
```

Other security features: **Row-Level Security** (`ALTER TABLE sales ENABLE ROW LEVEL SECURITY` + policies, e.g. branch isolation), SSL-only connections, never connecting as a superuser from the app.

## 9.9 Backups, restore and high availability

| Method | Use |
| --- | --- |
| **Neon point-in-time restore** | Restore the database (or a branch) to any moment in the retention window |
| `pg_dump` / `pg_restore` | Logical backup of one database; portable between versions |
| `pg_dumpall` | All databases + roles |
| Physical backups + WAL archiving (pgBackRest, Barman) | Self-hosted servers: point-in-time recovery |
| **Streaming replication** | A standby copy for failover and read replicas |
| **Logical replication** | Replicate selected tables, zero-downtime major upgrades |

```bash
pg_dump "$DATABASE_URL_UNPOOLED" -Fc -f pharmacare-$(date +%F).dump      # custom format, compressed
pg_restore -d "$TARGET_URL" --clean --no-owner pharmacare-2026-10-06.dump
```

**Test your restores.** A backup you have never restored is a hope, not a backup.

## 9.10 Connection pooling

Every PostgreSQL connection costs memory on the server; serverless functions and many API instances can open hundreds. A **pooler** (PgBouncer) shares a small set of real connections among many clients. Neon's `-pooler` endpoint runs PgBouncer in **transaction mode**, which does not support session features such as `SET` across statements, `LISTEN/NOTIFY` or session-level prepared statements — use the direct URL for migrations and those features.

In the app, also keep a small client-side pool (`max: 10`) per instance (section 10).

## 9.11 Monitoring and maintenance

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Slowest queries by total time
SELECT query, calls, round(total_exec_time) AS total_ms, round(mean_exec_time, 1) AS avg_ms
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;

-- What is running right now / blocked
SELECT pid, state, wait_event_type, now() - query_start AS running_for, query
FROM pg_stat_activity WHERE state <> 'idle' ORDER BY running_for DESC;

-- Cancel or kill a runaway query
SELECT pg_cancel_backend(12345);
SELECT pg_terminate_backend(12345);

-- Table and index sizes
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) FROM pg_stat_user_tables ORDER BY pg_total_relation_size(relid) DESC;
```

**Settings worth knowing** (self-hosted): `shared_buffers` (\~25% of RAM), `work_mem` (per sort/hash), `maintenance_work_mem`, `effective_cache_size`, `max_connections`, `statement_timeout` (stop runaway queries), `idle_in_transaction_session_timeout`. Neon manages most of these for you.

## 9.12 Scaling PostgreSQL

1. Fix queries and indexes first (most problems are here).
2. Cache hot reads in Redis (section 14).
3. Add **read replicas** for reports and dashboards.
4. **Partition** very large tables by time (`sales` by month) with declarative partitioning.
5. Scale vertically (bigger compute; Neon autoscaling).
6. Shard across databases only at very large scale (Citus).
