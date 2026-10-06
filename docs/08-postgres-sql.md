# 8. PostgreSQL 1: concepts, Neon and SQL

## 8.1 Relational database concepts

**What it is:** A relational database (RDBMS) stores data in **tables** (relations) of **rows** and **columns**. Tables link through **keys**. SQL (Structured Query Language) is how you define, read and change the data. PostgreSQL is a powerful open-source RDBMS known for correctness, rich features (JSONB, full-text search, extensions) and standards compliance.

| Term | Meaning | PharmaCare example |
| --- | --- | --- |
| Table | A set of rows with the same columns | `medicines` |
| Row (tuple) | One record | Amoxicillin 500mg |
| Column (attribute) | One field with a type | `price integer` |
| Primary key (PK) | Unique identifier of a row | `medicines.id` |
| Foreign key (FK) | A column pointing to another table's PK | `batches.medicine_id → medicines.id` |
| Constraint | A rule the database enforces | `price >= 0`, `batch_no UNIQUE` |
| Schema | A namespace grouping tables | `public`, `reporting` |
| Index | A structure that speeds up lookups | Index on `medicines(name)` |
| Transaction | A group of changes that succeed or fail together | Sale + stock deduction |

**ACID** — what makes relational databases safe for money:

- **Atomicity** — all of a transaction happens, or none of it.
- **Consistency** — constraints always hold.
- **Isolation** — concurrent transactions don't see each other's half-finished work.
- **Durability** — once committed, data survives a crash (write-ahead log).

**Normalisation:** store each fact once. Customer phone numbers live in `customers`, not copied into every sale. 1NF = atomic values, no repeating groups; 2NF/3NF = every non-key column depends on the whole key and nothing but the key. Denormalise deliberately only for measured performance needs (e.g. storing `sale_items.price` at the time of sale is **correct** — it is a historical fact, not a duplicate).

## 8.2 Neon: serverless PostgreSQL

**What it is:** Neon is managed PostgreSQL that separates storage from compute. Key features:

- **Autoscaling and scale-to-zero** — compute grows with load and can suspend when idle (cheap dev databases; the first query after idle has a short cold start).
- **Branching** — instant copy-on-write copies of your database: a branch per developer, per pull request, or for testing a risky migration on real-shaped data.
- **Connection pooling** — a built-in PgBouncer endpoint (hostname with `-pooler`) for many short connections.
- **Point-in-time restore** — restore to any moment within your history window.
- **Serverless driver** — `@neondatabase/serverless` queries over HTTP or WebSockets, ideal for serverless/edge functions.

**Setup:**

1. Create a project at neon.tech, pick a region close to your API server (e.g. Europe Frankfurt for East Africa traffic, or match your host's region).
2. Create a database `pharmacare` and copy two connection strings:
   - **Pooled** (`...-pooler...`) → for the app: `DATABASE_URL`.
   - **Direct** (no `-pooler`) → for migrations and admin tools: `DATABASE_URL_UNPOOLED`.
3. Always connect with `sslmode=require`.

```bash
# Neon CLI
npm i -g neonctl
neonctl auth
neonctl branches create --name feature/stock-transfers          # instant branch from main
neonctl connection-string feature/stock-transfers               # its URL
neonctl branches delete feature/stock-transfers
```

**Local alternative:** run PostgreSQL in Docker for offline development:

```bash
docker run -d --name pg -e POSTGRES_PASSWORD=secret -e POSTGRES_DB=pharmacare -p 5432:5432 postgres:17
```

## 8.3 `psql` — the command-line client

```bash
psql "$DATABASE_URL_UNPOOLED"
```

| Command | Does |
| --- | --- |
| `\l` | List databases |
| `\c pharmacare` | Connect to a database |
| `\dt` | List tables |
| `\d medicines` | Describe a table (columns, indexes, FKs) |
| `\di` | List indexes |
| `\du` | List roles |
| `\x` | Toggle expanded (vertical) output |
| `\timing` | Show query time |
| `\i file.sql` | Run a SQL file |
| `\q` | Quit |

GUI tools: Neon's SQL editor, DBeaver, TablePlus, pgAdmin, Drizzle Studio (`npx drizzle-kit studio`).

## 8.4 Data types

| Type | Use for | Example |
| --- | --- | --- |
| `integer` / `bigint` | Whole numbers, money in TZS, counts | `price integer` |
| `bigint generated always as identity` | Auto-increment primary keys | `id` |
| `uuid` | Public or distributed IDs | `gen_random_uuid()` |
| `numeric(12,2)` | Exact decimals (if a currency needs cents) | Never `real`/`double` for money |
| `text` / `varchar(n)` | Strings (`text` is fine; add a `CHECK` for length) | `name text` |
| `boolean` | Flags | `requires_prescription` |
| `date` | Calendar dates | `expires_at date` |
| `timestamptz` | Points in time (**always** with time zone; stored as UTC) | `created_at timestamptz` |
| `jsonb` | Flexible structured data, indexable | `meta jsonb` |
| `text[]` | Arrays | `tags text[]` |
| enum (`CREATE TYPE`) or `text` + `CHECK` | Fixed set of values | `role` |
| `inet`, `interval`, `bytea`, `tsvector` | IPs, durations, binary, full-text search |  |

## 8.5 The PharmaCare schema (DDL)

```sql
CREATE TYPE user_role AS ENUM ('admin', 'pharmacist', 'cashier');
CREATE TYPE payment_method AS ENUM ('cash', 'mpesa', 'airtel', 'card');
CREATE TYPE sale_status AS ENUM ('pending', 'paid', 'voided');

CREATE TABLE users (
  id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name          text NOT NULL,
  email         text NOT NULL UNIQUE,
  password_hash text NOT NULL,
  role          user_role NOT NULL DEFAULT 'cashier',
  is_active     boolean NOT NULL DEFAULT true,
  created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE categories (
  id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name text NOT NULL UNIQUE
);

CREATE TABLE medicines (
  id                    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  category_id           bigint NOT NULL REFERENCES categories(id) ON DELETE RESTRICT,
  name                  text NOT NULL CHECK (length(name) BETWEEN 2 AND 150),
  slug                  text NOT NULL UNIQUE,
  price                 integer NOT NULL CHECK (price >= 0),          -- whole TZS
  requires_prescription boolean NOT NULL DEFAULT false,
  reorder_level         integer NOT NULL DEFAULT 20 CHECK (reorder_level >= 0),
  version               integer NOT NULL DEFAULT 1,                   -- optimistic locking
  created_at            timestamptz NOT NULL DEFAULT now(),
  updated_at            timestamptz NOT NULL DEFAULT now(),
  deleted_at            timestamptz                                   -- soft delete
);

CREATE TABLE batches (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  medicine_id bigint NOT NULL REFERENCES medicines(id) ON DELETE CASCADE,
  batch_no    text NOT NULL UNIQUE,
  quantity    integer NOT NULL CHECK (quantity >= 0),   -- the DB itself refuses negative stock
  cost_price  integer NOT NULL CHECK (cost_price >= 0),
  expires_at  date NOT NULL,
  received_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE customers (
  id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name  text NOT NULL,
  phone text UNIQUE
);

CREATE TABLE sales (
  id             bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  receipt_no     text NOT NULL UNIQUE,
  cashier_id     bigint NOT NULL REFERENCES users(id),
  customer_id    bigint REFERENCES customers(id),
  total          integer NOT NULL CHECK (total >= 0),
  payment_method payment_method NOT NULL,
  status         sale_status NOT NULL DEFAULT 'pending',
  created_at     timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE sale_items (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  sale_id     bigint NOT NULL REFERENCES sales(id) ON DELETE CASCADE,
  medicine_id bigint NOT NULL REFERENCES medicines(id),
  batch_id    bigint NOT NULL REFERENCES batches(id),
  qty         integer NOT NULL CHECK (qty > 0),
  price       integer NOT NULL CHECK (price >= 0)    -- price AT THE TIME of sale
);
```

Change tables later with `ALTER TABLE` (but in a real project, through **migrations**, section 10):

```sql
ALTER TABLE medicines ADD COLUMN barcode text UNIQUE;
ALTER TABLE medicines ALTER COLUMN reorder_level SET DEFAULT 30;
ALTER TABLE medicines RENAME COLUMN name TO display_name;
ALTER TABLE medicines DROP COLUMN barcode;
```

## 8.6 Changing data (DML)

```sql
INSERT INTO categories (name) VALUES ('Antibiotics'), ('Painkillers') RETURNING id, name;

INSERT INTO medicines (category_id, name, slug, price)
VALUES (1, 'Amoxicillin 500mg', 'amoxicillin-500mg', 500)
RETURNING *;

UPDATE medicines SET price = 550, updated_at = now() WHERE id = 7 RETURNING price;

DELETE FROM batches WHERE quantity = 0 AND expires_at < current_date - interval '1 year';

-- Upsert: insert, or update if the unique key already exists
INSERT INTO categories (name) VALUES ('Vitamins')
ON CONFLICT (name) DO NOTHING;

INSERT INTO medicines (category_id, name, slug, price) VALUES (1, 'Amoxicillin 500mg', 'amoxicillin-500mg', 520)
ON CONFLICT (slug) DO UPDATE SET price = EXCLUDED.price, updated_at = now();
```

**Always write a `WHERE` on `UPDATE` and `DELETE`.** Run it as a `SELECT` first to see which rows it touches.

## 8.7 Querying

```sql
-- Filter, sort, limit
SELECT id, name, price
FROM medicines
WHERE deleted_at IS NULL
  AND price BETWEEN 200 AND 1000
  AND name ILIKE '%amox%'                 -- case-insensitive match
  AND category_id IN (1, 2)
ORDER BY price DESC, name
LIMIT 20 OFFSET 0;

-- NULL handling
SELECT name, COALESCE(phone, 'no phone') FROM customers WHERE phone IS NOT NULL;

-- CASE expression
SELECT name, CASE WHEN price < 500 THEN 'budget' WHEN price < 5000 THEN 'standard' ELSE 'premium' END AS band
FROM medicines;
```

**Joins:**

```sql
-- INNER JOIN: medicines with their category
SELECT m.name, c.name AS category
FROM medicines m
JOIN categories c ON c.id = m.category_id;

-- LEFT JOIN: every medicine, with total stock (0 if no batches)
SELECT m.id, m.name, COALESCE(SUM(b.quantity), 0) AS stock
FROM medicines m
LEFT JOIN batches b ON b.medicine_id = m.id AND b.expires_at > current_date
GROUP BY m.id, m.name;
```

| Join | Returns |
| --- | --- |
| `INNER JOIN` | Only rows with a match on both sides |
| `LEFT JOIN` | All rows from the left table, matches or NULLs from the right |
| `RIGHT JOIN` | The reverse (rarely used) |
| `FULL JOIN` | All rows from both sides |
| `CROSS JOIN` | Every combination |

**Aggregates and grouping:**

```sql
-- Daily sales by payment method
SELECT date_trunc('day', created_at) AS day, payment_method, COUNT(*) AS sales, SUM(total) AS revenue
FROM sales
WHERE status = 'paid' AND created_at >= now() - interval '7 days'
GROUP BY 1, 2
ORDER BY 1 DESC;

-- Low stock: HAVING filters after grouping
SELECT m.name, SUM(b.quantity) AS stock, m.reorder_level
FROM medicines m JOIN batches b ON b.medicine_id = m.id
WHERE b.expires_at > current_date
GROUP BY m.id
HAVING SUM(b.quantity) <= m.reorder_level;
```

**Subqueries and CTEs (`WITH`):**

```sql
-- Medicines that never sold
SELECT name FROM medicines m
WHERE NOT EXISTS (SELECT 1 FROM sale_items si WHERE si.medicine_id = m.id);

-- CTE: readable multi-step query — top 10 sellers this month and their share of revenue
WITH monthly AS (
  SELECT si.medicine_id, SUM(si.qty * si.price) AS revenue
  FROM sale_items si JOIN sales s ON s.id = si.sale_id
  WHERE s.status = 'paid' AND s.created_at >= date_trunc('month', now())
  GROUP BY si.medicine_id
)
SELECT m.name, mo.revenue, ROUND(100.0 * mo.revenue / SUM(mo.revenue) OVER (), 1) AS pct
FROM monthly mo JOIN medicines m ON m.id = mo.medicine_id
ORDER BY mo.revenue DESC
LIMIT 10;
```

**Window functions** — calculations across related rows without collapsing them:

```sql
-- Running total of revenue per day, and each cashier's rank
SELECT date_trunc('day', created_at)::date AS day,
       SUM(total) AS revenue,
       SUM(SUM(total)) OVER (ORDER BY date_trunc('day', created_at)) AS running_total
FROM sales WHERE status = 'paid'
GROUP BY 1;

SELECT cashier_id, SUM(total) AS revenue,
       RANK() OVER (ORDER BY SUM(total) DESC) AS rank
FROM sales WHERE created_at >= date_trunc('month', now())
GROUP BY cashier_id;

-- FEFO: the batch to sell first for each medicine
SELECT * FROM (
  SELECT b.*, ROW_NUMBER() OVER (PARTITION BY medicine_id ORDER BY expires_at) AS rn
  FROM batches b WHERE quantity > 0 AND expires_at > current_date
) x WHERE rn = 1;
```

Other useful window functions: `LAG`/`LEAD` (compare with the previous/next row: day-over-day change), `DENSE_RANK`, `NTILE`, `FIRST_VALUE`.

**Set operations:** `UNION` / `UNION ALL` (combine results), `INTERSECT`, `EXCEPT`.
