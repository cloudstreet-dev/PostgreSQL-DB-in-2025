# Chapter 7: Performance Optimization - Making PostgreSQL Fly

## Chapter Overview

In this chapter, you'll learn:
- A systematic approach to performance optimization
- Understanding query execution with EXPLAIN
- Index design and optimization strategies
- Query optimization techniques
- Configuration tuning for different workloads
- Connection pooling and resource management
- Monitoring and diagnosing performance issues
- VACUUM and maintenance strategies
- Hardware considerations

By the end of this chapter, you'll know how to make PostgreSQL perform optimally for your specific workload.

## Performance Optimization Methodology

### The Systematic Approach

Before randomly creating indexes or tweaking settings, follow this methodology:

1. **Measure** - Establish baseline performance
2. **Identify** - Find the bottleneck
3. **Analyze** - Understand why it's slow
4. **Optimize** - Apply targeted fixes
5. **Verify** - Confirm improvement
6. **Monitor** - Watch for regression

```sql
-- Step 1: Enable query statistics
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Step 2: Reset statistics for clean baseline
SELECT pg_stat_statements_reset();

-- Step 3: Run your workload

-- Step 4: Find slow queries
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    stddev_exec_time,
    max_exec_time,
    rows,
    100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0) AS cache_hit_ratio
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat_statements%'
ORDER BY mean_exec_time DESC
LIMIT 20;
```

## Understanding EXPLAIN

EXPLAIN is your window into PostgreSQL's query planner.

### Basic EXPLAIN Usage

```sql
-- See the query plan
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;

-- See the plan with actual execution
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 123;

-- Detailed analysis
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, TIMING, FORMAT JSON)
SELECT * FROM orders WHERE customer_id = 123;
```

### Reading EXPLAIN Output

```sql
-- Sample query
EXPLAIN ANALYZE
SELECT c.name, COUNT(o.id) as order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE c.created_at > '2024-01-01'
GROUP BY c.id, c.name
ORDER BY order_count DESC;

-- Typical output:
/*
Sort  (cost=218.55..218.68 rows=50 width=106) (actual time=2.145..2.156 rows=50 loops=1)
  Sort Key: (count(o.id)) DESC
  Sort Method: quicksort  Memory: 33kB
  ->  HashAggregate  (cost=216.38..217.13 rows=50 width=106) (actual time=2.089..2.110 rows=50 loops=1)
        Group Key: c.id, c.name
        ->  Hash Right Join  (cost=14.45..154.70 rows=12336 width=106) (actual time=0.146..1.234 rows=245 loops=1)
              Hash Cond: (o.customer_id = c.id)
              ->  Seq Scan on orders o  (cost=0.00..104.36 rows=12336 width=8) (actual time=0.008..0.234 rows=200 loops=1)
              ->  Hash  (cost=13.70..13.70 rows=60 width=102) (actual time=0.123..0.124 rows=50 loops=1)
                    Buckets: 1024  Batches: 1  Memory Usage: 14kB
                    ->  Seq Scan on customers c  (cost=0.00..13.70 rows=60 width=102) (actual time=0.023..0.089 rows=50 loops=1)
                          Filter: (created_at > '2024-01-01'::date)
Planning Time: 0.234 ms
Execution Time: 2.234 ms
*/
```

### Key Metrics to Watch

```sql
-- Cost: Planner's estimate (arbitrary units)
-- Rows: Estimated rows vs actual
-- Width: Average row size
-- Time: Estimated vs actual time
-- Buffers: Shared buffer hits/reads
-- Loops: How many times node executed

-- Good signs:
-- - Index scans instead of sequential scans (for selective queries)
-- - Hash joins for large joins
-- - Low loops count
-- - High buffer hit ratio

-- Warning signs:
-- - Nested loops with high outer rows
-- - Large sorts in memory
-- - Sequential scans on large tables
-- - Actual rows >> estimated rows
```

## Index Design and Optimization

### Index Types and Use Cases

```sql
-- B-tree (default): Equality and range queries
CREATE INDEX idx_orders_date ON orders(order_date);
SELECT * FROM orders WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31';

-- Hash: Only equality comparisons (rarely used)
CREATE INDEX idx_users_email_hash ON users USING HASH (email);
SELECT * FROM users WHERE email = 'user@example.com';

-- GIN: Arrays, JSONB, full-text search
CREATE INDEX idx_products_tags ON products USING GIN (tags);
SELECT * FROM products WHERE tags @> ARRAY['electronics'];

-- GiST: Geometric data, range types
CREATE INDEX idx_events_period ON events USING GIST (event_period);
SELECT * FROM events WHERE event_period && '[2024-01-01,2024-01-31]'::daterange;

-- BRIN: Large, naturally ordered tables
CREATE INDEX idx_logs_timestamp ON logs USING BRIN (timestamp);
SELECT * FROM logs WHERE timestamp > '2024-01-01';

-- SP-GiST: Partitioned search trees
CREATE INDEX idx_points ON locations USING SPGIST (point);
```

### Multi-Column Indexes

```sql
-- Order matters! Left-most columns can be used alone
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- This uses the index
SELECT * FROM orders WHERE customer_id = 123;
SELECT * FROM orders WHERE customer_id = 123 AND order_date = '2024-01-01';

-- This doesn't use the index efficiently
SELECT * FROM orders WHERE order_date = '2024-01-01';

-- For the last query, you'd need:
CREATE INDEX idx_orders_date_customer ON orders(order_date, customer_id);

-- Or create separate indexes and let PostgreSQL combine them
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_date ON orders(order_date);
```

### Partial Indexes

```sql
-- Index only the rows you query
CREATE INDEX idx_active_users ON users(email)
WHERE status = 'active';

-- Index for recent data
CREATE INDEX idx_recent_orders ON orders(customer_id)
WHERE order_date > CURRENT_DATE - INTERVAL '90 days';

-- Index for non-null values
CREATE INDEX idx_customer_phone ON customers(phone)
WHERE phone IS NOT NULL;

-- Conditional indexes
CREATE INDEX idx_high_value_orders ON orders(customer_id)
WHERE total_amount > 1000;
```

### Expression Indexes

```sql
-- Index on computed values
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- Index on JSON fields
CREATE INDEX idx_profile_age ON users((profile->>'age')::int);
SELECT * FROM users WHERE (profile->>'age')::int > 25;

-- Index on date parts
CREATE INDEX idx_orders_month ON orders(DATE_TRUNC('month', order_date));
SELECT * FROM orders WHERE DATE_TRUNC('month', order_date) = '2024-01-01';

-- Index on concatenated fields
CREATE INDEX idx_full_name ON customers((first_name || ' ' || last_name));
```

### Covering Indexes (Include Columns)

```sql
-- PostgreSQL 11+: Include non-key columns
CREATE INDEX idx_orders_customer_include
ON orders(customer_id)
INCLUDE (order_date, total_amount, status);

-- This query only reads the index, not the table
SELECT order_date, total_amount, status
FROM orders
WHERE customer_id = 123;

-- Check with EXPLAIN - look for "Index Only Scan"
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_date, total_amount, status
FROM orders
WHERE customer_id = 123;
```

### Index Maintenance

```sql
-- Monitor index usage
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Find unused indexes
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND indexrelname NOT LIKE '%_pkey';

-- Find duplicate indexes
WITH index_info AS (
    SELECT
        indrelid,
        indexrelid,
        indkey,
        indisunique,
        indisprimary
    FROM pg_index
)
SELECT
    a.indexrelid::regclass AS index1,
    b.indexrelid::regclass AS index2,
    pg_size_pretty(pg_relation_size(a.indexrelid)) AS size1,
    pg_size_pretty(pg_relation_size(b.indexrelid)) AS size2
FROM index_info a
JOIN index_info b ON a.indrelid = b.indrelid
    AND a.indkey = b.indkey
    AND a.indexrelid < b.indexrelid;

-- Rebuild bloated indexes
REINDEX INDEX CONCURRENTLY idx_orders_customer;

-- Or create new and swap
CREATE INDEX CONCURRENTLY idx_orders_customer_new ON orders(customer_id);
DROP INDEX CONCURRENTLY idx_orders_customer;
ALTER INDEX idx_orders_customer_new RENAME TO idx_orders_customer;
```

## Query Optimization Techniques

### Rewriting Queries for Performance

```sql
-- Use EXISTS instead of IN for large subqueries
-- Slow:
SELECT * FROM orders
WHERE customer_id IN (
    SELECT id FROM customers WHERE country = 'USA'
);

-- Faster:
SELECT * FROM orders o
WHERE EXISTS (
    SELECT 1 FROM customers c
    WHERE c.id = o.customer_id AND c.country = 'USA'
);

-- Use JOIN instead of correlated subqueries
-- Slow:
SELECT
    o.*,
    (SELECT name FROM customers c WHERE c.id = o.customer_id) as customer_name
FROM orders o;

-- Faster:
SELECT o.*, c.name as customer_name
FROM orders o
JOIN customers c ON c.id = o.customer_id;

-- Avoid NOT IN with nullable columns
-- Dangerous (can return no rows if NULL exists):
SELECT * FROM orders
WHERE customer_id NOT IN (SELECT id FROM archived_customers);

-- Safe and faster:
SELECT * FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM archived_customers ac WHERE ac.id = o.customer_id
);
```

### Optimizing Aggregations

```sql
-- Pre-aggregate in CTEs
WITH daily_totals AS (
    SELECT
        DATE_TRUNC('day', order_date) AS day,
        SUM(total_amount) AS daily_total,
        COUNT(*) AS order_count
    FROM orders
    WHERE order_date >= '2024-01-01'
    GROUP BY DATE_TRUNC('day', order_date)
)
SELECT
    day,
    daily_total,
    SUM(daily_total) OVER (ORDER BY day) AS running_total
FROM daily_totals;

-- Use FILTER instead of CASE in aggregations
-- Less efficient:
SELECT
    COUNT(CASE WHEN status = 'completed' THEN 1 END) as completed,
    COUNT(CASE WHEN status = 'pending' THEN 1 END) as pending
FROM orders;

-- More efficient:
SELECT
    COUNT(*) FILTER (WHERE status = 'completed') as completed,
    COUNT(*) FILTER (WHERE status = 'pending') as pending
FROM orders;

-- Parallel aggregation
SET max_parallel_workers_per_gather = 4;
SELECT category_id, COUNT(*), SUM(price)
FROM large_products_table
GROUP BY category_id;
```

### Optimizing Joins

```sql
-- Join order matters - start with most selective table
-- Let PostgreSQL decide:
SELECT *
FROM large_table l
JOIN small_table s ON l.id = s.large_id
WHERE s.status = 'active';  -- Selective filter

-- Force join order if needed:
SET join_collapse_limit = 1;
SELECT *
FROM small_table s
JOIN large_table l ON l.id = s.large_id
WHERE s.status = 'active';

-- Use appropriate join types
-- Nested Loop: Good for small result sets
-- Hash Join: Good for larger sets, equality conditions
-- Merge Join: Good for pre-sorted data

-- Hint PostgreSQL about join size
SET enable_nestloop = off;  -- Temporary, for testing
-- Run query
SET enable_nestloop = on;
```

## Configuration Tuning

### Memory Configuration

```sql
-- Check current settings
SHOW shared_buffers;
SHOW work_mem;
SHOW maintenance_work_mem;
SHOW effective_cache_size;

-- Optimal settings depend on your hardware
-- For a 32GB RAM server:
ALTER SYSTEM SET shared_buffers = '8GB';           -- 25% of RAM
ALTER SYSTEM SET effective_cache_size = '24GB';    -- 75% of RAM
ALTER SYSTEM SET work_mem = '64MB';                -- RAM / (max_connections * 2)
ALTER SYSTEM SET maintenance_work_mem = '2GB';     -- For VACUUM, CREATE INDEX

-- Apply changes
SELECT pg_reload_conf();
-- Some changes require restart:
-- sudo systemctl restart postgresql
```

### Query Planner Configuration

```sql
-- For SSDs (lower random access cost)
ALTER SYSTEM SET random_page_cost = 1.1;  -- Default is 4.0
ALTER SYSTEM SET effective_io_concurrency = 200;  -- Default is 1

-- Enable parallel queries
ALTER SYSTEM SET max_parallel_workers_per_gather = 4;
ALTER SYSTEM SET max_parallel_workers = 8;
ALTER SYSTEM SET parallel_tuple_cost = 0.1;
ALTER SYSTEM SET parallel_setup_cost = 1000;

-- JIT compilation (PostgreSQL 11+)
ALTER SYSTEM SET jit = on;
ALTER SYSTEM SET jit_above_cost = 100000;

-- Statistics target (more = better plans, slower ANALYZE)
ALTER SYSTEM SET default_statistics_target = 100;  -- Default is 100

-- For specific columns needing better statistics:
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 1000;
```

### Write-Ahead Log (WAL) Tuning

```sql
-- Check current WAL settings
SHOW wal_buffers;
SHOW checkpoint_timeout;
SHOW checkpoint_completion_target;
SHOW max_wal_size;

-- Optimize for write-heavy workloads
ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM SET checkpoint_timeout = '15min';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
ALTER SYSTEM SET max_wal_size = '4GB';
ALTER SYSTEM SET min_wal_size = '1GB';

-- For SSDs, can be more aggressive
ALTER SYSTEM SET synchronous_commit = off;  -- Risky but fast
ALTER SYSTEM SET wal_writer_delay = '10ms';
ALTER SYSTEM SET commit_delay = 100;
ALTER SYSTEM SET commit_siblings = 5;
```

## Connection Pooling

### PgBouncer Configuration

```ini
; pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

; Pool modes:
; session - Server connection assigned to client (default)
; transaction - Server connection assigned per transaction
; statement - Server connection assigned per statement
pool_mode = transaction

; Connection limits
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
reserve_pool_timeout = 3

; Timeouts
server_idle_timeout = 600
server_lifetime = 3600
server_connect_timeout = 15
```

### Application-Level Pooling

```python
# Python example with psycopg2
from psycopg2 import pool

# Create connection pool
connection_pool = pool.ThreadedConnectionPool(
    5,   # Min connections
    20,  # Max connections
    host='localhost',
    port=5432,
    database='mydb',
    user='appuser',
    password='password'
)

# Use connection from pool
conn = connection_pool.getconn()
try:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM users")
        results = cur.fetchall()
finally:
    connection_pool.putconn(conn)

# Monitor pool
print(f"Pool size: {connection_pool.maxconn}")
print(f"In use: {len(connection_pool._used)}")
```

## Monitoring Performance

### Key Metrics to Monitor

```sql
-- Database-wide statistics
CREATE VIEW performance_metrics AS
SELECT
    -- Cache hit ratio (should be > 99%)
    (SELECT sum(blks_hit) * 100.0 / NULLIF(sum(blks_hit) + sum(blks_read), 0)
     FROM pg_stat_database) AS cache_hit_ratio,

    -- Transaction rate
    (SELECT sum(xact_commit + xact_rollback) / EXTRACT(epoch FROM (now() - stats_reset))
     FROM pg_stat_database) AS tps,

    -- Active connections
    (SELECT count(*) FROM pg_stat_activity WHERE state != 'idle') AS active_connections,

    -- Longest query
    (SELECT max(EXTRACT(epoch FROM (now() - query_start)))
     FROM pg_stat_activity WHERE state = 'active') AS longest_query_seconds,

    -- Database size
    (SELECT pg_size_pretty(pg_database_size(current_database()))) AS database_size,

    -- Table bloat estimate
    (SELECT round(sum(n_dead_tup)::numeric / NULLIF(sum(n_live_tup), 0) * 100, 2)
     FROM pg_stat_user_tables) AS dead_tuple_percent;

-- Check metrics
SELECT * FROM performance_metrics;

-- Historical query performance
SELECT
    query,
    calls,
    mean_exec_time,
    total_exec_time,
    stddev_exec_time,
    rows,
    100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0) AS hit_ratio
FROM pg_stat_statements
WHERE query !~ '^(COPY|VACUUM|ANALYZE|CREATE|ALTER|DROP)'
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### Lock Monitoring

```sql
-- Current locks
SELECT
    pid,
    usename,
    pg_blocking_pids(pid) AS blocked_by,
    query,
    state,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE state != 'idle';

-- Blocking queries
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

## VACUUM and Maintenance

### Understanding Bloat

```sql
-- Check table bloat
WITH bloat_estimate AS (
    SELECT
        schemaname,
        tablename,
        pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
        n_live_tup,
        n_dead_tup,
        round(n_dead_tup::numeric / NULLIF(n_live_tup, 0) * 100, 2) AS dead_percent
    FROM pg_stat_user_tables
)
SELECT *
FROM bloat_estimate
WHERE n_dead_tup > 1000
ORDER BY dead_percent DESC;

-- Check index bloat
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;
```

### VACUUM Strategy

```sql
-- Manual VACUUM
VACUUM (ANALYZE, VERBOSE) orders;

-- Aggressive VACUUM
VACUUM (FULL, ANALYZE, VERBOSE) orders;  -- Locks table!

-- Configure autovacuum per table
ALTER TABLE high_traffic_table SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_analyze_scale_factor = 0.005,
    autovacuum_vacuum_cost_delay = 2,
    autovacuum_vacuum_cost_limit = 1000
);

-- Monitor autovacuum
SELECT
    schemaname,
    tablename,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    vacuum_count,
    autovacuum_count
FROM pg_stat_user_tables
ORDER BY last_autovacuum NULLS FIRST;
```

## Hardware Considerations

### Storage Performance

```sql
-- Test disk performance
CREATE TABLE io_test (id serial, data text);
INSERT INTO io_test (data) SELECT repeat('x', 1000) FROM generate_series(1, 100000);

-- Sequential read test
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM io_test;

-- Random read test
CREATE INDEX ON io_test(id);
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM io_test WHERE id = ANY(ARRAY(SELECT (random() * 100000)::int FROM generate_series(1, 1000)));

-- Write test
EXPLAIN (ANALYZE, BUFFERS) INSERT INTO io_test (data) SELECT repeat('y', 1000) FROM generate_series(1, 10000);

DROP TABLE io_test;
```

### CPU Optimization

```sql
-- Enable parallel execution
SET max_parallel_workers_per_gather = 4;
SET parallel_tuple_cost = 0.01;
SET parallel_setup_cost = 100;

-- Force parallel execution for testing
SET force_parallel_mode = on;
SET min_parallel_table_scan_size = 0;
SET min_parallel_index_scan_size = 0;

-- Check parallel execution
EXPLAIN (ANALYZE, VERBOSE)
SELECT category, COUNT(*), AVG(price)
FROM large_products
GROUP BY category;
```

## Performance Testing

### Using pgbench

```bash
# Initialize pgbench database
pgbench -i -s 100 benchdb

# Run read-only test
pgbench -c 10 -j 2 -T 60 -S benchdb

# Run read-write test
pgbench -c 10 -j 2 -T 60 benchdb

# Custom script test
cat > custom_test.sql << EOF
\set aid random(1, 100000 * :scale)
\set delta random(-5000, 5000)
BEGIN;
UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
COMMIT;
EOF

pgbench -c 10 -j 2 -T 60 -f custom_test.sql benchdb
```

### Load Testing Strategy

```sql
-- Create test workload procedure
CREATE OR REPLACE PROCEDURE generate_load(duration_seconds INT)
LANGUAGE plpgsql AS $$
DECLARE
    start_time TIMESTAMP := clock_timestamp();
    queries_executed INT := 0;
BEGIN
    WHILE clock_timestamp() < start_time + (duration_seconds || ' seconds')::INTERVAL LOOP
        -- Your typical queries here
        PERFORM * FROM orders WHERE customer_id = (random() * 1000)::INT;
        PERFORM * FROM products WHERE price > random() * 100;

        queries_executed := queries_executed + 2;

        -- Small delay to simulate real load
        PERFORM pg_sleep(0.01);
    END LOOP;

    RAISE NOTICE 'Executed % queries in % seconds',
        queries_executed,
        EXTRACT(epoch FROM (clock_timestamp() - start_time));
END;
$$;

-- Run load test
CALL generate_load(60);
```

## Exercises

### Exercise 7.1: Query Optimization
1. Take a slow query and optimize it using EXPLAIN
2. Create appropriate indexes and measure improvement
3. Rewrite the query for better performance

### Exercise 7.2: Configuration Tuning
1. Baseline your database performance
2. Adjust memory settings for your hardware
3. Measure the performance difference

### Exercise 7.3: Index Strategy
1. Analyze your workload with pg_stat_statements
2. Design an optimal index strategy
3. Implement and verify improvements

### Exercise 7.4: VACUUM Planning
1. Identify tables with high bloat
2. Create a VACUUM strategy
3. Implement autovacuum tuning

## Summary

You've learned comprehensive performance optimization:

✅ **Systematic Approach**: Measure, identify, analyze, optimize, verify
✅ **EXPLAIN Mastery**: Understanding query execution plans
✅ **Index Strategy**: Choosing and maintaining optimal indexes
✅ **Query Optimization**: Rewriting queries for performance
✅ **Configuration Tuning**: Memory, planner, and WAL settings
✅ **Connection Management**: Pooling for scalability
✅ **Monitoring**: Key metrics and what they mean
✅ **Maintenance**: VACUUM and bloat management
✅ **Testing**: Benchmarking and load testing

Performance optimization is iterative. Start with the biggest bottlenecks, measure improvements, and continue optimizing until you meet your performance goals.

## What's Next

Chapter 8 covers security and access control, ensuring your optimized database is also secure from threats and properly controlled.

## Additional Resources

- **EXPLAIN Visualizer**: [explain.depesz.com](https://explain.depesz.com/)
- **PgTune**: [pgtune.leopard.in.ua](https://pgtune.leopard.in.ua/)
- **pg_stat_statements**: [postgresql.org/docs/current/pgstatstatements.html](https://www.postgresql.org/docs/current/pgstatstatements.html)
- **Index Maintenance**: [pghero.dokkuapp.com](https://pghero.dokkuapp.com/)

---

*"The fastest query is the one you don't have to run. The second fastest is the one with the right index."*