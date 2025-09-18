# Chapter 6: Performance and Optimization - Making It Fly

"PostgreSQL is slow" is like saying "cars are slow" after driving one with the parking brake on. Let's learn how to tune PostgreSQL for speed.

Performance optimization is part science, part art, and part detective work. This chapter gives you the tools for all three.

## Understanding How PostgreSQL Thinks

Before optimizing, understand how PostgreSQL processes queries:

1. **Parser**: Checks SQL syntax
2. **Rewriter**: Applies rules and views
3. **Planner/Optimizer**: Decides HOW to execute
4. **Executor**: Actually runs the query

The planner is where the magic happens. It considers multiple ways to execute your query and picks the (hopefully) fastest one.

## EXPLAIN: X-Ray Vision for Queries

EXPLAIN shows you what PostgreSQL is thinking. EXPLAIN ANALYZE shows you what actually happened.

```sql
-- See the plan
EXPLAIN SELECT * FROM books WHERE price > 20;

-- See the plan with actual execution
EXPLAIN ANALYZE SELECT * FROM books WHERE price > 20;

-- More detail
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM books WHERE price > 20;

-- All the details
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, TIMING)
SELECT b.title, a.name
FROM books b
JOIN authors a ON b.author_id = a.id
WHERE b.price > 20;
```

### Reading EXPLAIN Output

```sql
EXPLAIN ANALYZE
SELECT * FROM books WHERE price > 20;

-- Output:
-- Seq Scan on books  (cost=0.00..25.88 rows=423 width=32) (actual time=0.015..0.201 rows=412 loops=1)
--   Filter: (price > 20)
--   Rows Removed by Filter: 1088
-- Planning Time: 0.075 ms
-- Execution Time: 0.234 ms
```

Key metrics:
- **cost**: Estimated cost (startup..total)
- **rows**: Estimated rows returned
- **width**: Average row size in bytes
- **actual time**: Real execution time (ms)
- **loops**: How many times this step ran

## Indexes: The Performance Multiplier

Indexes are like a book's table of contents. Without them, PostgreSQL reads every page (row) to find what you want.

### B-Tree Indexes (The Default)

```sql
-- Create an index
CREATE INDEX idx_books_price ON books(price);

-- Now this is fast
SELECT * FROM books WHERE price = 19.99;

-- Multi-column index (order matters!)
CREATE INDEX idx_books_author_year ON books(author_id, publication_year);

-- This uses the index
SELECT * FROM books WHERE author_id = 5 AND publication_year = 1950;

-- This also uses it (left-most prefix)
SELECT * FROM books WHERE author_id = 5;

-- This doesn't (not left-most)
SELECT * FROM books WHERE publication_year = 1950;
```

### Partial Indexes

Index only the rows you actually query:

```sql
-- Only index active users
CREATE INDEX idx_active_users ON users(email)
WHERE status = 'active';

-- Only index recent orders
CREATE INDEX idx_recent_orders ON orders(customer_id)
WHERE order_date > CURRENT_DATE - INTERVAL '30 days';

-- Only index expensive items
CREATE INDEX idx_premium_products ON products(name)
WHERE price > 100;
```

### Expression Indexes

Index computed values:

```sql
-- Index on lowercase email for case-insensitive searches
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Now this is fast
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';

-- Index on JSON fields
CREATE INDEX idx_products_brand ON products((details->>'brand'));

-- Date truncation for date range queries
CREATE INDEX idx_orders_month ON orders(DATE_TRUNC('month', order_date));
```

### Special Index Types

```sql
-- GIN: For full-text search and arrays
CREATE INDEX idx_articles_search ON articles USING GIN (search_vector);
CREATE INDEX idx_tags ON posts USING GIN (tags);

-- GiST: For geometric data and full-text
CREATE INDEX idx_locations ON stores USING GIST (coordinates);

-- BRIN: For large, naturally ordered tables
CREATE INDEX idx_logs_timestamp ON logs USING BRIN (timestamp);

-- Hash: For equality comparisons only
CREATE INDEX idx_users_id_hash ON users USING HASH (id);
```

### Index Maintenance

```sql
-- See all indexes on a table
\di books*

-- Check index usage
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,  -- Number of index scans
    idx_tup_read,  -- Rows read via index
    idx_tup_fetch  -- Rows fetched via index
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;

-- Find unused indexes
SELECT
    indexname,
    idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND indexname NOT LIKE '%_pkey';

-- Rebuild indexes (locks table!)
REINDEX TABLE books;

-- Concurrent rebuild (doesn't lock)
CREATE INDEX CONCURRENTLY idx_books_price_new ON books(price);
DROP INDEX idx_books_price;
ALTER INDEX idx_books_price_new RENAME TO idx_books_price;
```

## Query Optimization Patterns

### Use Covering Indexes

```sql
-- Include extra columns in index to avoid table lookups
CREATE INDEX idx_orders_covering ON orders(customer_id)
INCLUDE (order_date, total_amount);

-- This query only reads the index
SELECT order_date, total_amount
FROM orders
WHERE customer_id = 123;
```

### JOIN Optimization

```sql
-- Bad: Cartesian product then filter
SELECT *
FROM orders o, customers c
WHERE o.customer_id = c.id;

-- Good: Explicit JOIN
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- Push conditions into JOIN
-- Bad:
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'USA' AND o.total > 100;

-- Better:
SELECT *
FROM orders o
JOIN (SELECT * FROM customers WHERE country = 'USA') c
ON o.customer_id = c.id
WHERE o.total > 100;
```

### EXISTS vs IN vs JOIN

```sql
-- Often fastest: EXISTS
SELECT name FROM authors a
WHERE EXISTS (
    SELECT 1 FROM books b
    WHERE b.author_id = a.id AND b.price > 20
);

-- Sometimes slower: IN
SELECT name FROM authors
WHERE id IN (
    SELECT author_id FROM books WHERE price > 20
);

-- Can be slow with duplicates: JOIN
SELECT DISTINCT a.name
FROM authors a
JOIN books b ON a.id = b.author_id
WHERE b.price > 20;
```

### Pagination Performance

```sql
-- Bad for large offsets
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 20 OFFSET 10000;  -- Reads 10,020 rows!

-- Better: Keyset pagination
SELECT * FROM posts
WHERE created_at < '2024-01-01 00:00:00'
ORDER BY created_at DESC
LIMIT 20;

-- Best: Cursor-based
SELECT * FROM posts
WHERE id > 12345  -- Last ID from previous page
ORDER BY id
LIMIT 20;
```

## Configuration Tuning

### Memory Settings

Edit `postgresql.conf`:

```conf
# Shared memory for caching (25% of RAM is good start)
shared_buffers = 4GB

# Memory for each query operation
work_mem = 16MB  # Increase for complex queries

# Memory for maintenance operations
maintenance_work_mem = 512MB

# Connection memory
effective_cache_size = 12GB  # Total memory available for caching
```

### Query Planner Settings

```conf
# Help planner make better decisions
random_page_cost = 1.1  # Lower for SSDs (default 4.0)
effective_io_concurrency = 200  # Higher for SSDs

# Parallel query execution
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
parallel_tuple_cost = 0.1
parallel_setup_cost = 1000.0

# JIT compilation (PostgreSQL 11+)
jit = on
jit_above_cost = 100000
```

### Connection Pool

```conf
# Don't create too many connections
max_connections = 100  # Use connection pooling instead

# Better: Use PgBouncer or Pgpool
# Each connection uses ~10MB RAM
```

## Monitoring and Diagnostics

### Enable Statistics Collection

```sql
-- Track query statistics
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- In postgresql.conf:
-- shared_preload_libraries = 'pg_stat_statements'
-- pg_stat_statements.max = 10000
-- pg_stat_statements.track = all
```

### Find Slow Queries

```sql
-- Top 10 slowest queries
SELECT
    query,
    mean_exec_time,
    calls,
    total_exec_time,
    min_exec_time,
    max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Queries using most total time
SELECT
    query,
    total_exec_time,
    calls,
    mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Currently running queries
SELECT
    pid,
    now() - query_start AS duration,
    query,
    state
FROM pg_stat_activity
WHERE state != 'idle'
AND query NOT ILIKE '%pg_stat_activity%'
ORDER BY duration DESC;

-- Kill a long-running query
SELECT pg_cancel_backend(pid);  -- Gentle cancel
SELECT pg_terminate_backend(pid);  -- Force kill
```

### Table and Index Statistics

```sql
-- Table sizes
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Table bloat (dead rows)
SELECT
    schemaname,
    tablename,
    n_live_tup,
    n_dead_tup,
    n_dead_tup::float / NULLIF(n_live_tup, 0) AS dead_ratio
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY dead_ratio DESC;

-- Cache hit ratio (should be > 99%)
SELECT
    sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) AS cache_hit_ratio
FROM pg_statio_user_tables;
```

## VACUUM and Maintenance

PostgreSQL needs regular maintenance to stay fast.

### Understanding VACUUM

```sql
-- Manual VACUUM (marks dead rows as reusable)
VACUUM books;

-- VACUUM and update statistics
VACUUM ANALYZE books;

-- Full VACUUM (rewrites table, locks it!)
VACUUM FULL books;

-- See autovacuum activity
SELECT
    schemaname,
    tablename,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables;
```

### Autovacuum Tuning

```conf
# In postgresql.conf
autovacuum = on
autovacuum_max_workers = 4
autovacuum_naptime = 30s

# Per-table settings
ALTER TABLE high_churn_table SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_analyze_scale_factor = 0.05
);
```

## Partitioning Large Tables

Split huge tables into smaller, manageable pieces:

```sql
-- Range partitioning for time-series data
CREATE TABLE orders_2024 (
    id SERIAL,
    order_date DATE NOT NULL,
    customer_id INTEGER,
    total DECIMAL(10,2)
) PARTITION BY RANGE (order_date);

-- Create partitions
CREATE TABLE orders_2024_q1 PARTITION OF orders_2024
FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE orders_2024_q2 PARTITION OF orders_2024
FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- Automatic routing
INSERT INTO orders_2024 (order_date, customer_id, total)
VALUES ('2024-03-15', 123, 99.99);  -- Goes to Q1 partition

-- List partitioning for categories
CREATE TABLE products_by_category (
    id SERIAL,
    category TEXT,
    name TEXT,
    price DECIMAL(10,2)
) PARTITION BY LIST (category);

CREATE TABLE products_electronics PARTITION OF products_by_category
FOR VALUES IN ('laptop', 'phone', 'tablet');

CREATE TABLE products_clothing PARTITION OF products_by_category
FOR VALUES IN ('shirt', 'pants', 'shoes');
```

## Query Optimization Checklist

Before declaring "the database is slow":

1. **Run EXPLAIN ANALYZE** - What's the actual plan?
2. **Check indexes** - Are they being used? Do you need new ones?
3. **Update statistics** - Run ANALYZE on the table
4. **Check table bloat** - Maybe VACUUM is needed
5. **Look at configuration** - Is work_mem too low?
6. **Monitor connections** - Too many? Need pooling?
7. **Check disk I/O** - Is the disk the bottleneck?
8. **Review the query** - Can it be rewritten?

## Common Performance Killers

### 1. Missing Indexes
```sql
-- Symptom: Seq Scan on large table
-- Fix: Add appropriate index
CREATE INDEX idx_column ON table(column);
```

### 2. Implicit Type Conversions
```sql
-- Bad: Forces type conversion, can't use index
SELECT * FROM users WHERE id = '123';  -- id is integer

-- Good: Correct type
SELECT * FROM users WHERE id = 123;
```

### 3. NOT IN with NULLs
```sql
-- Bad: NOT IN with potential NULLs
SELECT * FROM orders
WHERE customer_id NOT IN (SELECT id FROM archived_customers);

-- Good: NOT EXISTS
SELECT * FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM archived_customers ac WHERE ac.id = o.customer_id
);
```

### 4. Wildcard at Start of LIKE
```sql
-- Bad: Can't use index
SELECT * FROM products WHERE name LIKE '%phone%';

-- Better: Full-text search
SELECT * FROM products
WHERE to_tsvector('english', name) @@ to_tsquery('phone');
```

## Performance Monitoring Dashboard

Create a simple monitoring view:

```sql
CREATE VIEW performance_dashboard AS
WITH cache_hit AS (
    SELECT
        sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) AS ratio
    FROM pg_statio_user_tables
),
connections AS (
    SELECT
        count(*) AS total,
        count(*) FILTER (WHERE state = 'active') AS active
    FROM pg_stat_activity
),
database_size AS (
    SELECT pg_database_size(current_database()) AS bytes
)
SELECT
    to_char(cache_hit.ratio * 100, '990.99%') AS cache_hit_ratio,
    connections.total AS total_connections,
    connections.active AS active_connections,
    pg_size_pretty(database_size.bytes) AS database_size,
    (SELECT count(*) FROM pg_stat_activity WHERE wait_event IS NOT NULL) AS waiting_queries,
    (SELECT count(*) FROM pg_stat_activity WHERE state = 'idle in transaction') AS idle_in_transaction
FROM cache_hit, connections, database_size;

-- Check it
SELECT * FROM performance_dashboard;
```

## What You've Learned

You now know how to:
- Use EXPLAIN to understand query plans
- Create effective indexes
- Optimize queries for speed
- Configure PostgreSQL for performance
- Monitor database health
- Maintain tables with VACUUM
- Partition large tables
- Identify and fix performance problems

Remember: premature optimization is the root of all evil, but a slow database is pretty evil too.

## What's Next

The final chapter brings it all together with real-world patterns and best practices. We'll cover architecture decisions, security, backup strategies, and how to sleep soundly knowing your database won't wake you at 3 AM.

*"There are two types of DBAs: those who have experienced data loss, and those who will. The smart ones are prepared for when they join the first group."*