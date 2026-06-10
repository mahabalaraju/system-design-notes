# SQL & Performance Tuning — Complete Notes

> Advanced reference covering SQL fundamentals, query optimization, indexing strategies, execution plans, and database-specific tuning for PostgreSQL and MySQL.

---

## Table of Contents

- [SQL Fundamentals](#sql-fundamentals)
  - [DDL — Data Definition Language](#ddl--data-definition-language)
  - [DML — Data Manipulation Language](#dml--data-manipulation-language)
  - [Constraints](#constraints)
  - [Joins](#joins)
  - [Subqueries & CTEs](#subqueries--ctes)
  - [Window Functions](#window-functions)
  - [Aggregations & GROUP BY](#aggregations--group-by)
  - [Set Operations](#set-operations)
- [Indexes](#indexes)
  - [B-Tree Index](#b-tree-index)
  - [Hash Index](#hash-index)
  - [Composite Index & Column Order](#composite-index--column-order)
  - [Covering Index](#covering-index)
  - [Partial Index](#partial-index)
  - [Expression Index](#expression-index)
  - [Full-Text Index](#full-text-index)
  - [Index Internals](#index-internals)
  - [When NOT to Index](#when-not-to-index)
- [Query Execution Plan](#query-execution-plan)
  - [EXPLAIN & EXPLAIN ANALYZE](#explain--explain-analyze)
  - [Reading a Plan](#reading-a-plan)
  - [Scan Types](#scan-types)
  - [Join Algorithms](#join-algorithms)
- [Query Optimization](#query-optimization)
  - [WHERE Clause Optimization](#where-clause-optimization)
  - [JOIN Optimization](#join-optimization)
  - [Pagination Optimization](#pagination-optimization)
  - [N+1 in Raw SQL](#n1-in-raw-sql)
  - [EXISTS vs IN vs JOIN](#exists-vs-in-vs-join)
  - [Avoiding Full Table Scans](#avoiding-full-table-scans)
- [Schema Design for Performance](#schema-design-for-performance)
  - [Normalization vs Denormalization](#normalization-vs-denormalization)
  - [Partitioning](#partitioning)
  - [Data Types & Storage](#data-types--storage)
- [Transactions & Locking](#transactions--locking)
  - [Isolation Levels Review](#isolation-levels-review)
  - [Row-Level Locking](#row-level-locking)
  - [Deadlocks](#deadlocks)
  - [MVCC Internals](#mvcc-internals)
- [PostgreSQL Specific](#postgresql-specific)
  - [VACUUM & AUTOVACUUM](#vacuum--autovacuum)
  - [Table Bloat](#table-bloat)
  - [Statistics & Planner](#statistics--planner)
  - [pg_stat Views](#pg_stat-views)
  - [Useful Extensions](#useful-extensions)
- [MySQL / InnoDB Specific](#mysql--innodb-specific)
  - [InnoDB Buffer Pool](#innodb-buffer-pool)
  - [Clustered Index](#clustered-index)
  - [InnoDB-Specific Tuning](#innodb-specific-tuning)
- [Connection Pooling](#connection-pooling)
- [Slow Query Identification](#slow-query-identification)
- [Performance Tuning Checklist](#performance-tuning-checklist)
- [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## SQL Fundamentals

### DDL — Data Definition Language

```sql
-- Create table with all common options
CREATE TABLE orders (
    id           BIGSERIAL       PRIMARY KEY,
    user_id      BIGINT          NOT NULL,
    status       VARCHAR(20)     NOT NULL DEFAULT 'PENDING',
    total        NUMERIC(12, 2)  NOT NULL CHECK (total >= 0),
    created_at   TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT chk_status CHECK (status IN ('PENDING','CONFIRMED','SHIPPED','DELIVERED','CANCELLED'))
);

-- Alter table
ALTER TABLE orders ADD COLUMN notes TEXT;
ALTER TABLE orders DROP COLUMN notes;
ALTER TABLE orders ALTER COLUMN status SET NOT NULL;
ALTER TABLE orders ALTER COLUMN total TYPE NUMERIC(15, 2);
ALTER TABLE orders RENAME COLUMN total TO amount;
ALTER TABLE orders ADD CONSTRAINT uk_order_ref UNIQUE (reference_no);

-- Indexes
CREATE INDEX idx_orders_user    ON orders(user_id);
CREATE INDEX idx_orders_status  ON orders(status);
CREATE UNIQUE INDEX uk_orders_ref ON orders(reference_no);
CREATE INDEX CONCURRENTLY idx_orders_created ON orders(created_at DESC); -- no table lock

-- Drop
DROP TABLE orders;
DROP TABLE IF EXISTS orders CASCADE; -- drops dependent objects too
TRUNCATE orders;                     -- fast delete all rows (no WHERE)
TRUNCATE orders RESTART IDENTITY;   -- reset sequences too
```

---

### DML — Data Manipulation Language

```sql
-- INSERT
INSERT INTO orders (user_id, status, total) VALUES (1, 'PENDING', 99.99);

-- Bulk insert
INSERT INTO orders (user_id, status, total)
VALUES (1, 'PENDING', 10.00),
       (2, 'PENDING', 20.00),
       (3, 'CONFIRMED', 30.00);

-- Insert from select
INSERT INTO order_archive SELECT * FROM orders WHERE created_at < NOW() - INTERVAL '1 year';

-- Upsert (PostgreSQL)
INSERT INTO products (id, name, price)
VALUES (1, 'Widget', 9.99)
ON CONFLICT (id) DO UPDATE
    SET name  = EXCLUDED.name,
        price = EXCLUDED.price,
        updated_at = NOW();

-- Upsert (MySQL)
INSERT INTO products (id, name, price) VALUES (1, 'Widget', 9.99)
ON DUPLICATE KEY UPDATE name = VALUES(name), price = VALUES(price);

-- UPDATE
UPDATE orders SET status = 'CONFIRMED', updated_at = NOW() WHERE id = 42;

-- UPDATE with JOIN (PostgreSQL)
UPDATE orders o
SET    status = 'FLAGGED'
FROM   users u
WHERE  o.user_id = u.id
  AND  u.is_suspended = true;

-- UPDATE with JOIN (MySQL)
UPDATE orders o
JOIN   users u ON o.user_id = u.id
SET    o.status = 'FLAGGED'
WHERE  u.is_suspended = true;

-- DELETE
DELETE FROM orders WHERE status = 'CANCELLED' AND created_at < NOW() - INTERVAL '90 days';

-- DELETE with JOIN (PostgreSQL)
DELETE FROM order_items oi
USING  orders o
WHERE  oi.order_id = o.id
  AND  o.status = 'CANCELLED';

-- RETURNING (PostgreSQL) — get affected rows
INSERT INTO orders (user_id, total) VALUES (1, 50.00) RETURNING id, created_at;
UPDATE orders SET status = 'SHIPPED' WHERE id = 42 RETURNING *;
DELETE FROM orders WHERE id = 42 RETURNING id;
```

---

### Constraints

```sql
-- Primary key
id BIGSERIAL PRIMARY KEY

-- Foreign key with actions
FOREIGN KEY (user_id) REFERENCES users(id)
    ON DELETE CASCADE    -- delete child when parent deleted
    ON DELETE SET NULL   -- set FK to NULL when parent deleted
    ON DELETE RESTRICT   -- prevent parent delete if children exist (default)
    ON UPDATE CASCADE    -- propagate PK change to FK

-- Unique
UNIQUE (email)
UNIQUE (user_id, product_id)  -- composite unique

-- Check
CHECK (age BETWEEN 0 AND 150)
CHECK (start_date < end_date)
CHECK (status IN ('A', 'B', 'C'))

-- Not null
column_name TYPE NOT NULL

-- Deferrable constraints (PostgreSQL) — check at commit, not at statement
CONSTRAINT fk_order FOREIGN KEY (order_id) REFERENCES orders(id)
    DEFERRABLE INITIALLY DEFERRED;
```

---

### Joins

```sql
-- INNER JOIN — only matching rows
SELECT o.id, u.email, o.total
FROM   orders o
JOIN   users  u ON o.user_id = u.id;

-- LEFT JOIN — all from left, matching from right (NULL if no match)
SELECT u.id, u.email, COUNT(o.id) AS order_count
FROM   users  u
LEFT   JOIN orders o ON o.user_id = u.id
GROUP  BY u.id, u.email;

-- RIGHT JOIN — all from right, matching from left (rare — rewrite as LEFT JOIN)
-- FULL OUTER JOIN — all rows from both sides
SELECT u.id, o.id
FROM   users u
FULL   OUTER JOIN orders o ON o.user_id = u.id;

-- CROSS JOIN — cartesian product
SELECT a.name, b.name FROM categories a CROSS JOIN categories b;

-- SELF JOIN — join table to itself
SELECT e.name AS employee, m.name AS manager
FROM   employees e
LEFT   JOIN employees m ON e.manager_id = m.id;

-- Multiple joins
SELECT o.id, u.email, p.name, oi.quantity
FROM   orders      o
JOIN   users       u  ON o.user_id   = u.id
JOIN   order_items oi ON oi.order_id = o.id
JOIN   products    p  ON oi.product_id = p.id
WHERE  o.status = 'CONFIRMED'
ORDER  BY o.created_at DESC;
```

---

### Subqueries & CTEs

```sql
-- Scalar subquery
SELECT id, total, (SELECT AVG(total) FROM orders) AS avg_total FROM orders;

-- Correlated subquery — executes once per row (often slow)
SELECT u.id, u.email
FROM   users u
WHERE  (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) > 5;

-- EXISTS — stops at first match (faster than COUNT)
SELECT u.id, u.email
FROM   users u
WHERE  EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.total > 100);

-- IN with subquery
SELECT * FROM products
WHERE  id IN (SELECT product_id FROM order_items WHERE quantity > 10);

-- CTE (Common Table Expression) — named, reusable, readable
WITH high_value_users AS (
    SELECT user_id, SUM(total) AS lifetime_value
    FROM   orders
    WHERE  status = 'DELIVERED'
    GROUP  BY user_id
    HAVING SUM(total) > 1000
),
recent_orders AS (
    SELECT user_id, COUNT(*) AS recent_count
    FROM   orders
    WHERE  created_at > NOW() - INTERVAL '30 days'
    GROUP  BY user_id
)
SELECT u.email, hv.lifetime_value, ro.recent_count
FROM   users u
JOIN   high_value_users hv ON hv.user_id = u.id
LEFT   JOIN recent_orders  ro ON ro.user_id = u.id
ORDER  BY hv.lifetime_value DESC;

-- Recursive CTE — org charts, trees, paths
WITH RECURSIVE subordinates AS (
    -- anchor: start with the root manager
    SELECT id, name, manager_id, 0 AS depth
    FROM   employees
    WHERE  manager_id IS NULL

    UNION ALL

    -- recursive: find direct reports
    SELECT e.id, e.name, e.manager_id, s.depth + 1
    FROM   employees  e
    JOIN   subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates ORDER BY depth, name;
```

---

### Window Functions

Window functions compute a value across a set of rows related to the current row **without collapsing rows** (unlike GROUP BY).

```sql
-- Syntax
function() OVER (
    PARTITION BY col   -- group (optional)
    ORDER BY col       -- ordering within the group
    ROWS/RANGE BETWEEN ... AND ...  -- frame (optional)
)

-- ROW_NUMBER — unique sequential number
SELECT id, user_id, total,
       ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
FROM   orders;

-- RANK & DENSE_RANK — rank with/without gaps on ties
SELECT product_id, SUM(quantity) AS units_sold,
       RANK()       OVER (ORDER BY SUM(quantity) DESC) AS rank,
       DENSE_RANK() OVER (ORDER BY SUM(quantity) DESC) AS dense_rank
FROM   order_items
GROUP  BY product_id;

-- LAG & LEAD — access previous/next row
SELECT id, total, created_at,
       LAG(total, 1)  OVER (ORDER BY created_at) AS prev_order_total,
       LEAD(total, 1) OVER (ORDER BY created_at) AS next_order_total,
       total - LAG(total, 1) OVER (ORDER BY created_at) AS delta
FROM   orders
WHERE  user_id = 42;

-- FIRST_VALUE & LAST_VALUE
SELECT id, user_id, total,
       FIRST_VALUE(total) OVER (PARTITION BY user_id ORDER BY created_at) AS first_order_total,
       LAST_VALUE(total)  OVER (
           PARTITION BY user_id ORDER BY created_at
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS last_order_total
FROM   orders;

-- Running total (cumulative sum)
SELECT id, total,
       SUM(total) OVER (ORDER BY created_at ROWS UNBOUNDED PRECEDING) AS running_total
FROM   orders;

-- Moving average (last 7 rows)
SELECT id, total,
       AVG(total) OVER (ORDER BY created_at ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7
FROM   orders;

-- NTILE — divide rows into N buckets
SELECT id, total,
       NTILE(4) OVER (ORDER BY total) AS quartile
FROM   orders;

-- PERCENT_RANK & CUME_DIST
SELECT id, total,
       PERCENT_RANK() OVER (ORDER BY total)  AS pct_rank,
       CUME_DIST()    OVER (ORDER BY total)  AS cum_dist
FROM   orders;

-- Practical: get latest order per user
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
) ranked
WHERE rn = 1;
```

---

### Aggregations & GROUP BY

```sql
-- Basic aggregates
SELECT
    status,
    COUNT(*)                    AS total_orders,
    COUNT(DISTINCT user_id)     AS unique_customers,
    SUM(total)                  AS revenue,
    AVG(total)                  AS avg_order_value,
    MIN(total)                  AS min_order,
    MAX(total)                  AS max_order,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY total) AS median_order  -- PostgreSQL
FROM  orders
WHERE created_at >= DATE_TRUNC('month', NOW())
GROUP BY status
HAVING COUNT(*) > 10           -- filter AFTER aggregation
ORDER BY revenue DESC;

-- GROUPING SETS — multiple groupings in one query
SELECT category, status, SUM(total)
FROM  orders o JOIN products p ON ...
GROUP BY GROUPING SETS (
    (category, status),  -- subtotal by category+status
    (category),          -- subtotal by category
    (status),            -- subtotal by status
    ()                   -- grand total
);

-- ROLLUP — hierarchical subtotals
GROUP BY ROLLUP (year, month, day);

-- CUBE — all combinations
GROUP BY CUBE (category, status, region);

-- FILTER (PostgreSQL) — conditional aggregation
SELECT
    COUNT(*) FILTER (WHERE status = 'DELIVERED') AS delivered,
    COUNT(*) FILTER (WHERE status = 'CANCELLED') AS cancelled,
    SUM(total) FILTER (WHERE created_at > NOW() - INTERVAL '7 days') AS weekly_revenue
FROM orders;
```

---

### Set Operations

```sql
-- UNION — combine, remove duplicates
SELECT email FROM customers
UNION
SELECT email FROM prospects;

-- UNION ALL — combine, keep duplicates (faster)
SELECT product_id FROM order_items
UNION ALL
SELECT product_id FROM wishlist_items;

-- INTERSECT — rows in both
SELECT user_id FROM buyers
INTERSECT
SELECT user_id FROM newsletter_subscribers;

-- EXCEPT / MINUS — rows in first but not second
SELECT user_id FROM all_users
EXCEPT
SELECT user_id FROM banned_users;
```

---

## Indexes

### B-Tree Index

Default index type. Stores sorted key values in a balanced tree. Supports `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `LIKE 'prefix%'`, `IS NULL`, `ORDER BY`.

```sql
CREATE INDEX idx_orders_user_id   ON orders(user_id);
CREATE INDEX idx_orders_created   ON orders(created_at DESC);
CREATE UNIQUE INDEX uk_users_email ON users(email);

-- Concurrent build — no table lock (PostgreSQL)
CREATE INDEX CONCURRENTLY idx_orders_status ON orders(status);
```

---

### Hash Index

Stores hash of key. Only supports `=`. Faster than B-Tree for pure equality lookups but no range support.

```sql
-- PostgreSQL
CREATE INDEX idx_sessions_token ON sessions USING HASH (token);
```

---

### Composite Index & Column Order

A composite index on `(a, b, c)` can be used for:
- `WHERE a = ?`
- `WHERE a = ? AND b = ?`
- `WHERE a = ? AND b = ? AND c = ?`
- `WHERE a = ? ORDER BY b`
- `ORDER BY a, b, c` (for sort)

But **NOT** for:
- `WHERE b = ?` (skips leading column)
- `WHERE c = ?` (skips leading columns)

```sql
-- Index for: WHERE user_id = ? AND status = ? ORDER BY created_at DESC
CREATE INDEX idx_orders_user_status_date
ON orders(user_id, status, created_at DESC);

-- Column order rule:
-- 1. Equality columns first (=)
-- 2. Range/inequality columns after (>, <, BETWEEN)
-- 3. ORDER BY columns last

-- Good: WHERE user_id = ? AND created_at > ?
CREATE INDEX ON orders(user_id, created_at);

-- Bad order: WHERE user_id = ? AND created_at > ?
-- CREATE INDEX ON orders(created_at, user_id); -- created_at is range, user_id won't be used
```

---

### Covering Index

An index that contains all columns needed by a query — the DB reads only the index, never touches the table (index-only scan).

```sql
-- Query: SELECT id, status, total FROM orders WHERE user_id = 1
-- Covering index: includes all selected columns
CREATE INDEX idx_orders_covering
ON orders(user_id) INCLUDE (status, total, created_at);
-- PostgreSQL INCLUDE — non-key columns stored in leaf, not in tree
-- MySQL: just list them all in the index key

-- Verify: EXPLAIN shows "Index Only Scan"
```

---

### Partial Index

Index only a subset of rows — smaller, faster, more targeted.

```sql
-- Only index pending orders (hot data)
CREATE INDEX idx_pending_orders ON orders(user_id, created_at)
WHERE status = 'PENDING';

-- Only index non-null values
CREATE INDEX idx_orders_coupon ON orders(coupon_id)
WHERE coupon_id IS NOT NULL;

-- Only active users
CREATE INDEX idx_active_users_email ON users(email)
WHERE is_active = true;
```

The query's `WHERE` clause must match the partial index condition for the DB to use it.

---

### Expression Index

Index the result of a function or expression.

```sql
-- Case-insensitive email search
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
-- Query must use: WHERE LOWER(email) = LOWER(?)

-- Index on extracted JSON field
CREATE INDEX idx_users_meta_city ON users((metadata->>'city'));

-- Index on date part
CREATE INDEX idx_orders_year_month ON orders(DATE_TRUNC('month', created_at));

-- Index on computed value
CREATE INDEX idx_products_discounted ON products((price * 0.9))
WHERE on_sale = true;
```

---

### Full-Text Index

```sql
-- PostgreSQL
ALTER TABLE articles ADD COLUMN search_vector TSVECTOR;
UPDATE articles SET search_vector =
    TO_TSVECTOR('english', COALESCE(title,'') || ' ' || COALESCE(body,''));

CREATE INDEX idx_articles_fts ON articles USING GIN(search_vector);

-- Query
SELECT title FROM articles
WHERE search_vector @@ TO_TSQUERY('english', 'java & concurrency')
ORDER BY TS_RANK(search_vector, TO_TSQUERY('english', 'java & concurrency')) DESC;

-- Auto-update trigger
CREATE TRIGGER tsvector_update BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION
    TSVECTOR_UPDATE_TRIGGER(search_vector, 'pg_catalog.english', title, body);

-- MySQL FULLTEXT
CREATE FULLTEXT INDEX ft_articles ON articles(title, body);
SELECT * FROM articles
WHERE MATCH(title, body) AGAINST ('java concurrency' IN NATURAL LANGUAGE MODE);
```

---

### Index Internals

```
B-Tree structure:
  Root node
    ├── Internal nodes (keys + pointers to children)
    └── Leaf nodes (keys + heap pointers / row data)
         └── linked list of leaf nodes (for range scans)

Page size: typically 8KB (PostgreSQL) / 16KB (MySQL InnoDB)
Height: log_b(n) — B-Tree of order b with n entries
  1M rows → height ~3–4 (very few I/Os for lookup)

Index lookup cost:
  - Seek: O(log n) — traverse tree
  - Range scan: O(log n + k) — k = rows returned
  - Full scan: O(n/b) — sequential leaf reads
```

**Index overhead:**
- Every write (INSERT/UPDATE/DELETE) must also update all indexes
- Each index adds ~10–30% overhead to write throughput
- Index storage is typically 10–30% of table size

---

### When NOT to Index

- Very low-cardinality columns (`status` with 3 values, `boolean`) — full table scan may be faster
- Columns in small tables (< 1000 rows) — sequential scan is faster
- Columns only used in `SELECT *` but never in `WHERE`/`JOIN`/`ORDER BY`
- Tables with very high write:read ratio
- Columns that are never filtered/sorted

---

## Query Execution Plan

### EXPLAIN & EXPLAIN ANALYZE

```sql
-- EXPLAIN — shows estimated plan (no execution)
EXPLAIN SELECT * FROM orders WHERE user_id = 1;

-- EXPLAIN ANALYZE — executes query, shows actual vs estimated
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, u.email, o.total
FROM   orders o
JOIN   users  u ON o.user_id = u.id
WHERE  o.status = 'PENDING'
ORDER  BY o.created_at DESC
LIMIT  20;

-- MySQL
EXPLAIN FORMAT=JSON SELECT ...;
EXPLAIN ANALYZE SELECT ...;  -- MySQL 8.0+
```

---

### Reading a Plan

```
Gather  (cost=1000.00..5432.10 rows=250 width=64) (actual time=12.3..45.6 rows=243 loops=1)
  ->  Parallel Seq Scan on orders  (cost=0..4100.00 rows=104) (actual time=0.1..30.2 rows=243)
        Filter: (status = 'PENDING')
        Rows Removed by Filter: 9757

Hash Join  (cost=200..1500 rows=500 width=80) (actual time=5.2..18.3 rows=487 loops=1)
  Hash Cond: (o.user_id = u.id)
  ->  Index Scan using idx_orders_status on orders o
        (cost=0.43..1100.00 rows=500 width=40) (actual time=0.05..8.1 rows=500 loops=1)
        Index Cond: (status = 'PENDING')
  ->  Hash  (cost=150.00..150.00 rows=4000 width=40) (actual time=3.1..3.1 rows=4000 loops=1)
        Buckets: 4096  Batches: 1
        ->  Seq Scan on users u
```

**Key fields:**
- `cost=start..total` — estimated cost in arbitrary planner units
- `rows=N` — estimated rows
- `actual time=start..end` — real execution time in ms
- `loops=N` — how many times this node was executed
- `Rows Removed by Filter` — wasted work (index may help here)
- Large gap between `rows=estimated` and `actual rows` → stale statistics → run `ANALYZE`

---

### Scan Types

| Scan | When used | Cost |
|---|---|---|
| **Seq Scan** | No usable index, small table, large fraction of rows | O(n) — reads entire table |
| **Index Scan** | Selective query with index; fetches heap for each row | O(log n + k) |
| **Index Only Scan** | Covering index — all cols in index; no heap access | O(log n + k) — fastest |
| **Bitmap Index Scan** | Medium selectivity; combines multiple indexes | O(log n + k) — batched heap access |
| **Bitmap Heap Scan** | Second step after Bitmap Index Scan | Sorted heap access |
| **TID Scan** | Access by physical row location (ctid) | Rare |

```
Rule of thumb:
  < 5% of rows → Index Scan
  5–20% of rows → Bitmap Index Scan
  > 20% of rows → Seq Scan (planner chooses this)
```

---

### Join Algorithms

**Nested Loop Join:**
```
For each row in outer table:
    For each matching row in inner table:
        output pair

Cost: O(outer × inner) — good for small outer, indexed inner
Best when: small result set, inner side has index on join key
```

**Hash Join:**
```
Phase 1 — Build: hash the smaller table into memory hash table
Phase 2 — Probe: scan larger table, probe hash for matches

Cost: O(n + m) — very efficient for large unsorted tables
Best when: no useful index, equi-join, tables fit in memory
```

**Merge Join:**
```
Both inputs must be sorted on join key
Scan both sorted inputs simultaneously

Cost: O(n log n + m log m) — sort cost + O(n + m) merge
Best when: inputs already sorted (index), large equi-join
```

```sql
-- Force a join type (PostgreSQL) — for testing
SET enable_hashjoin = off;
SET enable_mergejoin = off;
SET enable_nestloop = off;
```

---

## Query Optimization

### WHERE Clause Optimization

```sql
-- ❌ Function on indexed column — disables index
WHERE YEAR(created_at) = 2024
WHERE UPPER(email) = 'USER@EXAMPLE.COM'
WHERE user_id + 0 = 42

-- ✅ Rewrite to use index
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
WHERE email = 'user@example.com'          -- store emails lowercased
WHERE user_id = 42

-- ❌ Leading wildcard — full table scan
WHERE name LIKE '%widget%'

-- ✅ Prefix match — uses index
WHERE name LIKE 'widget%'

-- ✅ For contains — use full-text search
WHERE search_vector @@ TO_TSQUERY('widget')

-- ❌ OR on different columns — each side is a separate scan
WHERE status = 'PENDING' OR user_id = 42

-- ✅ Rewrite with UNION ALL (each uses its own index)
SELECT * FROM orders WHERE status = 'PENDING'
UNION ALL
SELECT * FROM orders WHERE user_id = 42 AND status != 'PENDING';

-- ❌ NOT IN with subquery — can't use index well, NULL issues
WHERE user_id NOT IN (SELECT id FROM banned_users)

-- ✅ NOT EXISTS — cleaner, handles NULLs, faster
WHERE NOT EXISTS (SELECT 1 FROM banned_users WHERE id = user_id)

-- ❌ Implicit type cast disables index
WHERE user_id = '42'  -- user_id is INT, '42' is string → cast applied

-- ✅ Match types
WHERE user_id = 42
```

---

### JOIN Optimization

```sql
-- Join order matters for nested loop — put smaller table first
-- The planner usually handles this, but you can hint it

-- ❌ Missing index on FK column — full scan on order_items for each order
SELECT o.id, oi.product_id
FROM   orders o
JOIN   order_items oi ON oi.order_id = o.id;
-- ✅ Always index FK columns
CREATE INDEX idx_order_items_order_id ON order_items(order_id);

-- ❌ Joining on expressions
WHERE o.user_id = CAST(u.external_id AS BIGINT)
-- ✅ Normalize types at schema level

-- Reduce rows before joining
-- ❌ Join everything, then filter
SELECT u.email, o.total
FROM   users u
JOIN   orders o ON o.user_id = u.id
WHERE  o.total > 100 AND u.country = 'IN';

-- ✅ Filter early (subquery/CTE), then join
WITH big_orders AS (
    SELECT user_id, total FROM orders WHERE total > 100
)
SELECT u.email, bo.total
FROM   big_orders bo
JOIN   users u ON u.id = bo.user_id AND u.country = 'IN';
```

---

### Pagination Optimization

```sql
-- ❌ OFFSET pagination — gets slower with higher offsets
-- DB must read and discard all rows before the offset
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
-- Cost: O(offset + limit) — reads 10,020 rows, returns 20

-- ✅ Keyset / Cursor pagination — constant cost regardless of page
-- Use the last seen value as the cursor
SELECT * FROM orders
WHERE  created_at < :last_seen_created_at  -- from previous page's last row
   OR  (created_at = :last_seen_created_at AND id < :last_seen_id)
ORDER  BY created_at DESC, id DESC
LIMIT  20;
-- Cost: O(log n + limit) — uses index regardless of page depth

-- ✅ For random access pagination with large OFFSET — fetch IDs first
SELECT o.*
FROM   orders o
JOIN  (
    SELECT id FROM orders
    ORDER  BY created_at DESC
    LIMIT  20 OFFSET 10000
) ids ON o.id = ids.id;
-- ID subquery is index-only; join fetches only 20 full rows

-- Count optimization — avoid COUNT(*) on filtered large tables
-- ❌ Exact count every time
SELECT COUNT(*) FROM orders WHERE user_id = 42;

-- ✅ Cache counts in a summary table
-- ✅ Use approximate count for display
SELECT reltuples::BIGINT FROM pg_class WHERE relname = 'orders'; -- pg fast estimate
```

---

### N+1 in Raw SQL

```sql
-- ❌ Application-side N+1
-- 1 query for users, then N queries for orders
SELECT * FROM users WHERE active = true;     -- 1 query
-- loop: SELECT * FROM orders WHERE user_id = ?  -- N queries

-- ✅ JOIN — single query
SELECT u.id, u.email, o.id AS order_id, o.total
FROM   users u
JOIN   orders o ON o.user_id = u.id
WHERE  u.active = true;

-- ✅ Two-query batch approach (for large datasets)
SELECT id FROM users WHERE active = true;   -- get IDs
SELECT * FROM orders WHERE user_id IN (...); -- batch fetch
```

---

### EXISTS vs IN vs JOIN

```sql
-- EXISTS — stops at first match, handles NULLs correctly
-- Best for: "does at least one matching row exist?"
SELECT u.id FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- IN — materializes subquery result, then probes
-- Best for: small subquery result, no NULLs in subquery
SELECT u.id FROM users u
WHERE u.id IN (SELECT user_id FROM orders WHERE total > 100);

-- IN vs NOT IN — NULL gotcha:
-- NOT IN returns no rows if subquery contains ANY NULL
-- Always use NOT EXISTS for exclusion queries

-- JOIN — most flexible, can return columns from both sides
-- Best for: need data from both tables
SELECT DISTINCT u.id FROM users u
JOIN orders o ON o.user_id = u.id;
-- Note: use DISTINCT or GROUP BY to avoid row multiplication

-- Semi-join (EXISTS) = JOIN + DISTINCT in most cases
-- Planner often rewrites IN → semi-join internally anyway
```

---

### Avoiding Full Table Scans

```sql
-- Check if index is being used
EXPLAIN SELECT * FROM orders WHERE status = 'PENDING';
-- If "Seq Scan" appears — index missing or not selective enough

-- Force index (MySQL)
SELECT * FROM orders FORCE INDEX (idx_orders_status) WHERE status = 'PENDING';

-- PostgreSQL — disable seq scan to test index path
SET enable_seqscan = off;
EXPLAIN SELECT * FROM orders WHERE status = 'PENDING';
SET enable_seqscan = on; -- reset

-- Common reasons planner ignores an index:
-- 1. Low selectivity (status has few distinct values)
-- 2. Stale statistics → run ANALYZE orders;
-- 3. Cost model miscalibration → SET random_page_cost = 1.1; (for SSD)
-- 4. Function on indexed column
-- 5. Implicit type cast
```

---

## Schema Design for Performance

### Normalization vs Denormalization

```
3NF (Normalized):
  ✅ No data duplication
  ✅ Easy updates
  ❌ More JOINs required
  ❌ Slower reads for complex queries

Denormalized:
  ✅ Fewer JOINs — faster reads
  ✅ Better for analytics / reporting
  ❌ Data duplication
  ❌ Update anomalies
  ❌ More storage

Strategy:
  OLTP (transactional): normalize to 3NF
  OLAP (analytics): denormalize, use star/snowflake schema
  Hybrid: normalize core, denormalize reporting tables / materialized views
```

**Materialized Views (PostgreSQL):**

```sql
-- Pre-compute expensive query result
CREATE MATERIALIZED VIEW mv_user_stats AS
SELECT
    u.id,
    u.email,
    COUNT(o.id)   AS total_orders,
    SUM(o.total)  AS lifetime_value,
    MAX(o.created_at) AS last_order_date
FROM   users u
LEFT   JOIN orders o ON o.user_id = u.id AND o.status = 'DELIVERED'
GROUP  BY u.id, u.email;

CREATE UNIQUE INDEX ON mv_user_stats(id); -- needed for CONCURRENTLY refresh

-- Refresh (full rebuild)
REFRESH MATERIALIZED VIEW mv_user_stats;

-- Refresh without locking reads
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_user_stats;

-- Query reads from cache — no joins, no aggregation
SELECT * FROM mv_user_stats WHERE lifetime_value > 1000;
```

---

### Partitioning

Split large tables into smaller physical partitions. Query only scans relevant partitions (partition pruning).

```sql
-- Range partitioning (PostgreSQL) — by date
CREATE TABLE orders (
    id         BIGSERIAL,
    user_id    BIGINT NOT NULL,
    total      NUMERIC(12,2),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- Default partition for unmatched rows
CREATE TABLE orders_default PARTITION OF orders DEFAULT;

-- Indexes on partitioned table (applied to all partitions)
CREATE INDEX ON orders(user_id);
CREATE INDEX ON orders(created_at);

-- Query automatically prunes to relevant partition
EXPLAIN SELECT * FROM orders WHERE created_at >= '2024-01-01';
-- Shows: Seq Scan on orders_2024 (not orders_2023, orders_2025)

-- List partitioning
CREATE TABLE users PARTITION BY LIST (country_code);
CREATE TABLE users_in PARTITION OF users FOR VALUES IN ('IN');
CREATE TABLE users_us PARTITION OF users FOR VALUES IN ('US');

-- Hash partitioning — even distribution
CREATE TABLE events PARTITION BY HASH (user_id);
CREATE TABLE events_0 PARTITION OF events FOR VALUES WITH (modulus 4, remainder 0);
CREATE TABLE events_1 PARTITION OF events FOR VALUES WITH (modulus 4, remainder 1);
CREATE TABLE events_2 PARTITION OF events FOR VALUES WITH (modulus 4, remainder 2);
CREATE TABLE events_3 PARTITION OF events FOR VALUES WITH (modulus 4, remainder 3);
```

**When to partition:**
- Table > 100M rows or > 100GB
- Queries consistently filter by the partition key
- Need to drop old data fast (`DROP TABLE orders_2020` vs `DELETE`)

---

### Data Types & Storage

```sql
-- Use smallest type that fits the data
-- INT (4B) instead of BIGINT (8B) for small counts
-- SMALLINT (2B) for status codes, ratings
-- NUMERIC(p,s) for money — never FLOAT (rounding errors)

-- PostgreSQL type sizes
SMALLINT      2 bytes   -32768 to 32767
INT           4 bytes   ~±2.1B
BIGINT        8 bytes   ~±9.2 × 10^18
REAL          4 bytes   6 decimal digits (inexact)
DOUBLE        8 bytes   15 decimal digits (inexact)
NUMERIC(p,s)  variable  exact decimal — use for money
CHAR(n)       n bytes   fixed-length, blank-padded (rarely use)
VARCHAR(n)    variable  up to n chars
TEXT          variable  unlimited — same performance as VARCHAR in Postgres
BOOLEAN       1 byte
DATE          4 bytes
TIMESTAMP     8 bytes   no timezone
TIMESTAMPTZ   8 bytes   UTC internally — PREFER this
UUID          16 bytes  use gen_random_uuid()
JSONB         variable  binary JSON — indexable, prefer over JSON
JSON          variable  text JSON — stored as-is, slower

-- UUID vs BIGSERIAL as PK
-- UUID: globally unique, distributed-friendly, no sequence contention
--       BUT random inserts fragment B-Tree index → table/index bloat
-- BIGSERIAL: sequential → B-Tree insertion at end → minimal fragmentation
-- UUIDv7: time-ordered UUID — best of both worlds (PostgreSQL 17+ / extension)
```

---

## Transactions & Locking

### Isolation Levels Review

```sql
-- Set isolation level for current transaction
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED; -- default PostgreSQL
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Set default for session
SET default_transaction_isolation = 'repeatable read';
```

---

### Row-Level Locking

```sql
-- SELECT FOR UPDATE — exclusive lock, blocks other writers & FOR UPDATE readers
BEGIN;
SELECT * FROM inventory WHERE product_id = 42 FOR UPDATE;
-- other transactions block on this row until COMMIT/ROLLBACK
UPDATE inventory SET stock = stock - 1 WHERE product_id = 42;
COMMIT;

-- SELECT FOR SHARE — shared lock, blocks writers but not other readers
SELECT * FROM products WHERE id = 42 FOR SHARE;

-- SKIP LOCKED — skip rows already locked (queue processing pattern)
SELECT * FROM job_queue
WHERE  status = 'PENDING'
ORDER  BY created_at
LIMIT  1
FOR    UPDATE SKIP LOCKED;
-- Multiple workers can each grab a different row without blocking each other

-- NOWAIT — fail immediately if lock unavailable (instead of waiting)
SELECT * FROM orders WHERE id = 42 FOR UPDATE NOWAIT;
-- Throws: ERROR: could not obtain lock on row in relation "orders"

-- Lock escalation — avoid locking too many rows
-- Lock at the right granularity: row > page > table
-- Table-level locks
LOCK TABLE orders IN EXCLUSIVE MODE;    -- blocks all reads and writes
LOCK TABLE orders IN SHARE MODE;        -- blocks writes, allows reads
LOCK TABLE orders IN ROW EXCLUSIVE MODE;-- only blocks conflicting row locks
```

---

### Deadlocks

```sql
-- Classic deadlock
-- T1: locks row A, waits for row B
-- T2: locks row B, waits for row A
-- DB detects cycle → kills one transaction → other proceeds

-- Prevention: always acquire locks in consistent order
-- ✅ Both transactions lock by ascending ID
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- locks id=1 first, then id=2 — consistent order

-- Detect deadlock frequency
SELECT * FROM pg_stat_activity WHERE wait_event_type = 'Lock';
SELECT COUNT(*) FROM pg_locks WHERE NOT granted;

-- PostgreSQL deadlock log
-- log_lock_waits = on  (in postgresql.conf)
-- deadlock_timeout = 1s (default)
```

---

### MVCC Internals

Multi-Version Concurrency Control — readers don't block writers, writers don't block readers.

```
PostgreSQL MVCC:
  Each row version has:
    xmin — transaction ID that created this version
    xmax — transaction ID that deleted/updated this version (0 if current)

  Reader sees a row version if:
    xmin committed AND (xmax = 0 OR xmax not yet committed)

  Writer creates NEW row version; marks old version with xmax
  → Dead tuples accumulate → VACUUM cleans them up

MySQL InnoDB MVCC:
  Uses undo log to reconstruct older versions
  Read view created at transaction start (REPEATABLE READ)
  or at each statement start (READ COMMITTED)
```

---

## PostgreSQL Specific

### VACUUM & AUTOVACUUM

```sql
-- Dead tuples accumulate from UPDATEs and DELETEs (MVCC)
-- VACUUM removes dead tuples and frees space for reuse
-- VACUUM FULL reclaims disk space (rewrites table — locks!)

-- Manual vacuum
VACUUM orders;                    -- removes dead tuples, no table lock
VACUUM ANALYZE orders;            -- vacuum + update statistics
VACUUM FULL orders;               -- reclaims disk space (EXCLUSIVE LOCK — use carefully)

-- Check dead tuple count
SELECT relname,
       n_live_tup,
       n_dead_tup,
       ROUND(n_dead_tup::NUMERIC / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
       last_autovacuum,
       last_autoanalyze
FROM   pg_stat_user_tables
ORDER  BY n_dead_tup DESC;

-- Autovacuum tuning (postgresql.conf)
autovacuum = on
autovacuum_vacuum_scale_factor = 0.01     -- trigger at 1% dead tuples (default 0.2 = 20%)
autovacuum_analyze_scale_factor = 0.005  -- trigger analyze at 0.5% changes
autovacuum_vacuum_cost_delay = 2ms       -- reduce if autovacuum too slow
autovacuum_max_workers = 5               -- increase for many tables

-- For high-write tables: lower thresholds so autovacuum runs more aggressively
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 1000
);
```

---

### Table Bloat

```sql
-- Check table and index bloat
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_indexes_size(schemaname||'.'||tablename)) AS indexes_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Reclaim space without downtime
-- 1. pg_repack extension (rewrites table without EXCLUSIVE LOCK)
-- 2. CLUSTER orders USING idx_orders_created; (rewrites + reorders — needs lock)
-- 3. pg_squeeze extension
```

---

### Statistics & Planner

```sql
-- Update statistics (tell planner about data distribution)
ANALYZE orders;
ANALYZE;  -- all tables

-- Check statistics
SELECT attname, n_distinct, correlation, most_common_vals
FROM   pg_stats
WHERE  tablename = 'orders'
  AND  attname   = 'status';

-- correlation: 1.0 = perfectly ordered, 0 = random, -1 = reverse ordered
-- Planner uses correlation to decide: high correlation → index scan; low → seq scan

-- Increase statistics target for columns with poor estimates
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;  -- default 100
ANALYZE orders;

-- Planner cost settings (tune for SSD)
SET random_page_cost = 1.1;   -- default 4.0 (HDD). SSD: 1.0–1.5
SET effective_cache_size = '12GB';  -- OS + DB cache available (~75% of RAM)
SET work_mem = '64MB';        -- per sort/hash operation (× connections × ops)
SET shared_buffers = '4GB';   -- DB cache (~25% of RAM)

-- Force planner to re-estimate joins
SET join_collapse_limit = 1;  -- disable join reordering (for debugging)
```

---

### pg_stat Views

```sql
-- Active queries
SELECT pid, now() - query_start AS duration, state, query
FROM   pg_stat_activity
WHERE  state != 'idle'
ORDER  BY duration DESC;

-- Kill long-running query
SELECT pg_cancel_backend(pid);   -- graceful (sends SIGINT)
SELECT pg_terminate_backend(pid); -- forceful (sends SIGTERM)

-- Lock waits
SELECT
    blocked.pid     AS blocked_pid,
    blocked.query   AS blocked_query,
    blocking.pid    AS blocking_pid,
    blocking.query  AS blocking_query
FROM  pg_stat_activity blocked
JOIN  pg_stat_activity blocking
      ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE NOT blocked.granted;  -- actually: WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0

-- Index usage
SELECT
    indexrelname AS index_name,
    idx_scan     AS scans,
    idx_tup_read AS tuples_read,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM   pg_stat_user_indexes
WHERE  schemaname = 'public'
ORDER  BY idx_scan DESC;

-- Unused indexes (candidates for removal)
SELECT indexrelname, idx_scan
FROM   pg_stat_user_indexes
WHERE  idx_scan = 0
  AND  indexrelname NOT LIKE 'pk_%'
  AND  indexrelname NOT LIKE 'uk_%';

-- Cache hit ratio (should be > 99%)
SELECT
    SUM(heap_blks_hit) / (SUM(heap_blks_hit) + SUM(heap_blks_read)) AS cache_hit_ratio
FROM pg_statio_user_tables;

-- Table stats
SELECT relname, seq_scan, idx_scan,
       seq_scan::FLOAT / NULLIF(seq_scan + idx_scan, 0) AS seq_scan_ratio
FROM   pg_stat_user_tables
ORDER  BY seq_scan DESC;
-- High seq_scan_ratio on large tables → missing index
```

---

### Useful Extensions

```sql
-- pg_stat_statements — track query performance statistics
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top slow queries by total time
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM   pg_stat_statements
ORDER  BY total_exec_time DESC
LIMIT  20;

-- pg_trgm — trigram similarity for fuzzy search + LIKE optimization
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_products_name_trgm ON products USING GIN(name gin_trgm_ops);
SELECT * FROM products WHERE name ILIKE '%widget%';  -- now uses GIN index

-- uuid-ossp / pgcrypto — UUID generation
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
SELECT uuid_generate_v4();

-- pg_partman — automatic partition management
-- pgvector — vector similarity search (AI/ML use cases)
-- timescaledb — time-series optimization
```

---

## MySQL / InnoDB Specific

### InnoDB Buffer Pool

The in-memory cache for table and index data. Most critical performance parameter.

```ini
# my.cnf
innodb_buffer_pool_size = 12G       # 70-80% of available RAM
innodb_buffer_pool_instances = 8    # 1 per GB, max 64
innodb_buffer_pool_chunk_size = 128M
```

```sql
-- Buffer pool hit ratio (should be > 99%)
SELECT
    (1 - (innodb_buffer_pool_reads / innodb_buffer_pool_read_requests)) * 100
    AS buffer_pool_hit_pct
FROM information_schema.INNODB_METRICS
WHERE name IN ('buffer_pool_reads', 'buffer_pool_read_requests');

-- Or
SHOW STATUS LIKE 'Innodb_buffer_pool%';
```

---

### Clustered Index

InnoDB stores table data in the **clustered index** (primary key order). Secondary indexes store the PK value as a pointer to the row.

```
Implications:
  - PK lookup: traverse B-Tree once → leaf has full row data
  - Secondary index lookup: traverse secondary B-Tree → get PK → traverse PK B-Tree again
    (double lookup unless covering index)
  - Sequential PK inserts (BIGINT AUTO_INCREMENT): efficient — appends to end of B-Tree
  - Random UUID PKs: inserts scattered throughout B-Tree → frequent page splits → fragmentation
```

```sql
-- Best PK for InnoDB: sequential (AUTO_INCREMENT BIGINT)
CREATE TABLE orders (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    ...
);

-- Secondary index covering — avoids double lookup
-- Query: SELECT status, total FROM orders WHERE user_id = 1
CREATE INDEX idx_orders_user_covering ON orders(user_id, status, total);
-- InnoDB: leaf of this index has (user_id, status, total, id) → no second lookup needed
```

---

### InnoDB-Specific Tuning

```ini
# my.cnf
innodb_log_file_size = 2G           # larger = fewer checkpoints = better write throughput
innodb_flush_log_at_trx_commit = 1  # 1=safe(default), 2=risk 1s data loss, 0=risk on crash
innodb_flush_method = O_DIRECT      # bypass OS cache for data files (avoid double buffering)
innodb_io_capacity = 2000           # IOPS available (SSD: 2000-20000, HDD: 200)
innodb_io_capacity_max = 4000
innodb_read_io_threads = 4
innodb_write_io_threads = 8

# Query cache (removed in MySQL 8.0 — caused scalability issues)
# Use application-level caching (Redis) instead
```

```sql
-- Slow query log
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;          -- log queries > 1 second
SET GLOBAL log_queries_not_using_indexes = ON;

-- EXPLAIN output
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE user_id = 1\G

-- Table statistics
ANALYZE TABLE orders;  -- update index statistics

-- Show index cardinality
SHOW INDEX FROM orders;

-- InnoDB status
SHOW ENGINE INNODB STATUS\G  -- shows lock waits, deadlocks, buffer pool info
```

---

## Connection Pooling

```
Without pooling:
  Each request: TCP connect → auth → query → TCP close
  ~50–100ms overhead per connection

With pooling:
  Connection established once, reused across requests
  ~0.1ms overhead per borrow
```

```ini
# HikariCP (Java — fastest pool)
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=2
spring.datasource.hikari.connection-timeout=30000      # 30s — throw if no conn available
spring.datasource.hikari.idle-timeout=600000           # 10min — close idle connections
spring.datasource.hikari.max-lifetime=1800000          # 30min — recycle connections
spring.datasource.hikari.leak-detection-threshold=60000 # warn if conn held > 60s

# PgBouncer (PostgreSQL connection pooler — sits between app and DB)
# Transaction mode: each SQL transaction gets a connection — most efficient
# Session mode: connection held for entire client session
# Statement mode: connection returned after each statement (limited SQL support)
```

**Pool sizing formula:**
```
connections = (core_count × 2) + effective_spindle_count

For a 4-core server with SSD:
  = (4 × 2) + 1 = 9 ≈ 10 connections

More connections ≠ more throughput
Too many connections → context switching + memory → worse performance
```

---

## Slow Query Identification

```sql
-- PostgreSQL — find slow queries using pg_stat_statements
SELECT
    LEFT(query, 100) AS query_snippet,
    calls,
    ROUND(total_exec_time::NUMERIC / calls, 2) AS avg_ms,
    ROUND(total_exec_time::NUMERIC, 2) AS total_ms,
    rows / calls AS avg_rows,
    shared_blks_hit + shared_blks_read AS total_blocks,
    ROUND(shared_blks_hit::NUMERIC /
          NULLIF(shared_blks_hit + shared_blks_read, 0) * 100, 2) AS cache_hit_pct
FROM   pg_stat_statements
WHERE  calls > 100
ORDER  BY avg_ms DESC
LIMIT  20;

-- Reset statistics
SELECT pg_stat_statements_reset();

-- MySQL — slow query log analysis with pt-query-digest
-- pt-query-digest /var/log/mysql/slow.log

-- Find queries doing full table scans
SELECT query, full_scan, exec_count
FROM   sys.statements_with_full_table_scans
ORDER  BY exec_count DESC;

-- Find queries with no index usage
SELECT * FROM sys.statements_with_errors_or_warnings
WHERE no_index_used_pct > 50;
```

---

## Performance Tuning Checklist

### Query Level
- [ ] Run `EXPLAIN ANALYZE` — understand the plan
- [ ] Check for Seq Scans on large tables → add index
- [ ] Check estimated vs actual rows → run `ANALYZE` if big gap
- [ ] Remove functions on indexed columns in `WHERE`
- [ ] Replace `OFFSET` pagination with keyset pagination
- [ ] Replace `NOT IN` with `NOT EXISTS`
- [ ] Replace correlated subqueries with JOINs or CTEs
- [ ] Use covering indexes for frequent read queries
- [ ] Use `EXISTS` instead of `COUNT(*)` for existence checks
- [ ] Push filters as early as possible (filter before join)

### Index Level
- [ ] Index all FK columns
- [ ] Index all `WHERE`, `JOIN ON`, `ORDER BY` columns
- [ ] Check composite index column order (equality → range → sort)
- [ ] Remove unused indexes (`pg_stat_user_indexes` where `idx_scan = 0`)
- [ ] Use partial indexes for sparse queries
- [ ] Use expression indexes for function-based filters
- [ ] Use covering indexes (INCLUDE) for read-heavy tables

### Schema Level
- [ ] Use `TIMESTAMPTZ` not `TIMESTAMP`
- [ ] Use `NUMERIC` not `FLOAT` for money
- [ ] Use `BIGSERIAL`/`AUTO_INCREMENT` not UUID for PK (unless distributed)
- [ ] Partition tables > 100M rows
- [ ] Use materialized views for expensive recurring queries
- [ ] Archive old data to separate tables / partitions

### Configuration Level (PostgreSQL)
- [ ] `shared_buffers` = 25% of RAM
- [ ] `effective_cache_size` = 75% of RAM
- [ ] `work_mem` tuned per workload (careful — per operation × connections)
- [ ] `random_page_cost = 1.1` for SSD
- [ ] `autovacuum_vacuum_scale_factor` lowered for high-write tables
- [ ] `pg_stat_statements` enabled
- [ ] Connection pool (PgBouncer) in front of DB

### Connection Level
- [ ] Use connection pool (HikariCP, PgBouncer)
- [ ] Set pool size based on core count, not arbitrarily high
- [ ] Set `connection-timeout` (don't wait forever)
- [ ] Set `max-lifetime` to recycle stale connections

---

## Quick Reference Cheat Sheet

### Index type selection

```
Equality only (=)                   → Hash Index
Range, sort, LIKE prefix            → B-Tree (default)
All needed columns in index         → Covering Index (INCLUDE)
Subset of rows                      → Partial Index
Function result                     → Expression Index
LIKE '%text%' / full text           → GIN + pg_trgm / Full-Text Index
Array, JSONB, GIS                   → GIN / GiST Index
```

### Join algorithm selection (planner does this — know it for tuning)

```
Small outer + indexed inner         → Nested Loop
Large unsorted equi-join            → Hash Join
Large sorted or indexed equi-join   → Merge Join
```

### Scan type reading

```
Seq Scan          → no index or low selectivity
Index Scan        → selective, fetches heap rows
Index Only Scan   → covering index — fastest
Bitmap Index Scan → medium selectivity, batches heap access
```

### When to use what

```
Exact lookup            → B-Tree equality, Hash
Range query             → B-Tree range
Sort performance        → B-Tree on ORDER BY col
Avoid heap access       → Covering index
Sparse data             → Partial index
Case-insensitive search → Expression index on LOWER(col)
Fuzzy / LIKE '%x%'      → GIN + pg_trgm
Pagination at scale     → Keyset pagination
Expensive recurring query → Materialized view
Table > 100M rows       → Partitioning
High concurrent reads   → Connection pool + read replicas
```

### Key pg_stat queries

```sql
-- Slow queries        → pg_stat_statements ORDER BY avg_exec_time DESC
-- Unused indexes      → pg_stat_user_indexes WHERE idx_scan = 0
-- Table bloat         → pg_stat_user_tables WHERE n_dead_tup is high
-- Lock waits          → pg_stat_activity WHERE wait_event_type = 'Lock'
-- Cache hit ratio     → pg_statio_user_tables
-- Active connections  → pg_stat_activity WHERE state != 'idle'
```

---

*References: PostgreSQL Docs (postgresql.org/docs) | Use The Index, Luke (use-the-index-luke.com) | High Performance MySQL — Schwartz, Zaitsev, Tkachenko | The Art of PostgreSQL — Fontaine*
