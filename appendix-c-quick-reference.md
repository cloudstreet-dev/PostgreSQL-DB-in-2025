# Appendix C: Quick Reference

*By Claude Code Opus 4.1*

Your PostgreSQL cheat sheet for those "what was that command again?" moments.

## psql Commands

### Connection
```bash
psql -h localhost -p 5432 -U username -d database
psql postgresql://user:password@host:5432/dbname

# Connection shortcuts
\c dbname                 # Connect to database
\c dbname username        # Connect as different user
\conninfo                 # Show connection info
```

### Navigation
```bash
\l or \list              # List databases
\dt                      # List tables
\dt+                     # List tables with sizes
\d tablename             # Describe table
\d+ tablename            # Describe table with more detail
\di                      # List indexes
\dv                      # List views
\df                      # List functions
\du                      # List users/roles
\dn                      # List schemas
\dx                      # List extensions
\dp or \z                # List table privileges
```

### Query Helpers
```bash
\g                       # Execute query
\gx                      # Execute query, expanded display
\gset                    # Store query result in psql variable
\watch 2                 # Re-run query every 2 seconds
\e                       # Edit query in editor
\ef function_name        # Edit function
\ev view_name           # Edit view
```

### Output Control
```bash
\a                       # Toggle aligned output
\x                       # Toggle expanded display
\t                       # Tuples only (no headers)
\o filename              # Send output to file
\o                       # Stop sending output to file
\H                       # HTML output mode
\pset null 'NULL'        # Display NULL as 'NULL'
```

### Import/Export
```bash
\copy table FROM 'file.csv' CSV HEADER
\copy table TO 'file.csv' CSV HEADER
\copy (SELECT ...) TO 'file.csv' CSV HEADER

\i filename.sql          # Execute SQL file
\ir filename.sql         # Execute SQL file (relative path)
```

### Utility
```bash
\timing                  # Toggle query timing
\! command              # Execute shell command
\cd directory           # Change directory
\setenv NAME value      # Set environment variable
\q                      # Quit psql
\? or \h                # Help
\h CREATE TABLE         # SQL command help
```

## SQL Quick Reference

### DDL (Data Definition)

```sql
-- Tables
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE users ADD COLUMN age INTEGER;
ALTER TABLE users DROP COLUMN age;
ALTER TABLE users RENAME COLUMN email TO email_address;
ALTER TABLE users ALTER COLUMN email TYPE VARCHAR(255);

DROP TABLE users;
DROP TABLE IF EXISTS users CASCADE;

-- Indexes
CREATE INDEX idx_users_email ON users(email);
CREATE UNIQUE INDEX idx_users_email ON users(email);
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
DROP INDEX idx_users_email;
REINDEX TABLE users;

-- Constraints
ALTER TABLE orders ADD CONSTRAINT fk_customer
    FOREIGN KEY (customer_id) REFERENCES customers(id);
ALTER TABLE users ADD CONSTRAINT email_format
    CHECK (email ~ '^[^@]+@[^@]+\.[^@]+$');
ALTER TABLE orders DROP CONSTRAINT fk_customer;

-- Views
CREATE VIEW active_users AS
    SELECT * FROM users WHERE deleted_at IS NULL;
CREATE OR REPLACE VIEW active_users AS ...
CREATE MATERIALIZED VIEW user_stats AS ...
REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats;
DROP VIEW active_users;
```

### DML (Data Manipulation)

```sql
-- INSERT
INSERT INTO users (email) VALUES ('user@example.com');
INSERT INTO users VALUES (DEFAULT, 'user@example.com', NOW());
INSERT INTO users (email)
    VALUES ('user1@example.com'), ('user2@example.com');
INSERT INTO users (email)
    SELECT email FROM temp_users;
INSERT INTO users (email) VALUES ('user@example.com')
    ON CONFLICT (email) DO NOTHING;
INSERT INTO users (email) VALUES ('user@example.com')
    ON CONFLICT (email) DO UPDATE SET updated_at = NOW();

-- UPDATE
UPDATE users SET age = 30 WHERE id = 1;
UPDATE users SET age = age + 1;
UPDATE orders o SET total = (
    SELECT SUM(price * quantity)
    FROM order_items oi
    WHERE oi.order_id = o.id
);

-- DELETE
DELETE FROM users WHERE id = 1;
DELETE FROM users WHERE created_at < NOW() - INTERVAL '1 year';
TRUNCATE TABLE temp_data;
TRUNCATE TABLE users RESTART IDENTITY CASCADE;

-- SELECT
SELECT * FROM users;
SELECT DISTINCT city FROM users;
SELECT * FROM users WHERE age > 18;
SELECT * FROM users ORDER BY created_at DESC;
SELECT * FROM users LIMIT 10 OFFSET 20;
SELECT * FROM users TABLESAMPLE SYSTEM (1);  -- 1% sample

-- RETURNING
INSERT INTO users (email) VALUES ('new@example.com') RETURNING id;
UPDATE users SET age = 30 WHERE id = 1 RETURNING *;
DELETE FROM users WHERE id = 1 RETURNING email;
```

### Joins

```sql
-- INNER JOIN
SELECT * FROM orders o
INNER JOIN customers c ON o.customer_id = c.id;

-- LEFT JOIN
SELECT * FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id;

-- RIGHT JOIN
SELECT * FROM orders o
RIGHT JOIN customers c ON o.customer_id = c.id;

-- FULL OUTER JOIN
SELECT * FROM customers c
FULL OUTER JOIN orders o ON c.id = o.customer_id;

-- CROSS JOIN
SELECT * FROM products CROSS JOIN categories;

-- Self JOIN
SELECT e1.name AS employee, e2.name AS manager
FROM employees e1
JOIN employees e2 ON e1.manager_id = e2.id;

-- LATERAL JOIN
SELECT * FROM customers c,
LATERAL (
    SELECT * FROM orders o
    WHERE o.customer_id = c.id
    ORDER BY created_at DESC
    LIMIT 3
) recent_orders;
```

### Common Table Expressions (CTEs)

```sql
-- Basic CTE
WITH user_orders AS (
    SELECT user_id, COUNT(*) as order_count
    FROM orders
    GROUP BY user_id
)
SELECT * FROM user_orders WHERE order_count > 5;

-- Multiple CTEs
WITH
regional_sales AS (
    SELECT region, SUM(amount) as total
    FROM sales GROUP BY region
),
top_regions AS (
    SELECT region FROM regional_sales
    WHERE total > (SELECT AVG(total) FROM regional_sales)
)
SELECT * FROM sales WHERE region IN (SELECT region FROM top_regions);

-- Recursive CTE
WITH RECURSIVE subordinates AS (
    SELECT id, name, manager_id
    FROM employees
    WHERE name = 'CEO'

    UNION ALL

    SELECT e.id, e.name, e.manager_id
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id
)
SELECT * FROM subordinates;
```

### Window Functions

```sql
-- ROW_NUMBER
SELECT
    ROW_NUMBER() OVER (ORDER BY salary DESC) as rank,
    name, salary
FROM employees;

-- RANK and DENSE_RANK
SELECT
    RANK() OVER (ORDER BY score DESC) as rank,
    DENSE_RANK() OVER (ORDER BY score DESC) as dense_rank,
    name, score
FROM students;

-- Running totals
SELECT
    date,
    amount,
    SUM(amount) OVER (ORDER BY date) as running_total
FROM transactions;

-- Moving average
SELECT
    date,
    price,
    AVG(price) OVER (
        ORDER BY date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as moving_avg_7d
FROM stock_prices;

-- Partition
SELECT
    department,
    name,
    salary,
    AVG(salary) OVER (PARTITION BY department) as dept_avg
FROM employees;

-- LAG and LEAD
SELECT
    date,
    value,
    LAG(value) OVER (ORDER BY date) as prev_value,
    LEAD(value) OVER (ORDER BY date) as next_value
FROM metrics;
```

## Data Types

### Numeric
```sql
SMALLINT                 -- 2 bytes, -32768 to 32767
INTEGER / INT            -- 4 bytes, -2147483648 to 2147483647
BIGINT                   -- 8 bytes, -9223372036854775808 to 9223372036854775807
DECIMAL(p,s) / NUMERIC   -- Variable precision
REAL                     -- 4 bytes, 6 decimal precision
DOUBLE PRECISION         -- 8 bytes, 15 decimal precision
SMALLSERIAL              -- 2 bytes, auto-increment
SERIAL                   -- 4 bytes, auto-increment
BIGSERIAL                -- 8 bytes, auto-increment
```

### Text
```sql
CHAR(n)                  -- Fixed length, blank padded
VARCHAR(n)               -- Variable length with limit
TEXT                     -- Variable unlimited length
```

### Date/Time
```sql
DATE                     -- Date only
TIME                     -- Time only
TIMESTAMP                -- Date and time, no timezone
TIMESTAMPTZ              -- Date and time with timezone
INTERVAL                 -- Time interval
```

### Boolean
```sql
BOOLEAN / BOOL           -- true/false/null
```

### Binary
```sql
BYTEA                    -- Binary data
```

### JSON
```sql
JSON                     -- Text storage
JSONB                    -- Binary storage, indexable
```

### Arrays
```sql
INTEGER[]                -- Array of integers
TEXT[]                   -- Array of text
ANYARRAY                 -- Any array type
```

### Special
```sql
UUID                     -- Universally unique identifier
INET                     -- IPv4/IPv6 address
CIDR                     -- IPv4/IPv6 network
MACADDR                  -- MAC address
POINT                    -- Geometric point
LINE                     -- Geometric line
POLYGON                  -- Geometric polygon
CIRCLE                   -- Geometric circle
```

## Operators

### Comparison
```sql
=                        -- Equal
<> or !=                 -- Not equal
<                        -- Less than
>                        -- Greater than
<=                       -- Less than or equal
>=                       -- Greater than or equal
BETWEEN x AND y          -- Between two values
IN (...)                 -- In list
NOT IN (...)             -- Not in list
IS NULL                  -- Is null
IS NOT NULL              -- Is not null
IS DISTINCT FROM         -- Null-safe inequality
IS NOT DISTINCT FROM     -- Null-safe equality
```

### Pattern Matching
```sql
LIKE 'pattern'           -- SQL pattern matching
ILIKE 'pattern'          -- Case-insensitive LIKE
NOT LIKE                 -- Negation
~ 'regex'                -- POSIX regex match
~* 'regex'               -- Case-insensitive regex
!~ 'regex'               -- Not match regex
SIMILAR TO 'pattern'     -- SQL regex
```

### Logical
```sql
AND                      -- Logical AND
OR                       -- Logical OR
NOT                      -- Logical NOT
```

### Mathematical
```sql
+                        -- Addition
-                        -- Subtraction
*                        -- Multiplication
/                        -- Division
%                        -- Modulo
^                        -- Exponentiation
|/                       -- Square root
||/                      -- Cube root
@                        -- Absolute value
```

### String
```sql
||                       -- Concatenation
LENGTH(string)           -- String length
LOWER(string)            -- Lowercase
UPPER(string)            -- Uppercase
TRIM(string)             -- Remove spaces
SUBSTRING(string, start, length)
REPLACE(string, from, to)
POSITION(substring IN string)
```

### JSON/JSONB
```sql
->                       -- Get JSON object field
->>                      -- Get JSON object field as text
#>                       -- Get JSON object at path
#>>                      -- Get JSON object at path as text
@>                       -- Contains
<@                       -- Is contained by
?                        -- Key exists
?&                       -- All keys exist
?|                       -- Any key exists
||                       -- Concatenate
-                        -- Delete key/element
```

## Functions

### Aggregate
```sql
COUNT(*)                 -- Count rows
COUNT(column)            -- Count non-null values
COUNT(DISTINCT column)   -- Count distinct values
SUM(column)              -- Sum values
AVG(column)              -- Average value
MIN(column)              -- Minimum value
MAX(column)              -- Maximum value
STRING_AGG(column, delimiter)  -- Concatenate strings
ARRAY_AGG(column)        -- Create array
JSONB_AGG(column)        -- Create JSON array
JSONB_OBJECT_AGG(key, value)  -- Create JSON object
```

### Date/Time
```sql
NOW()                    -- Current timestamp
CURRENT_DATE             -- Current date
CURRENT_TIME             -- Current time
CURRENT_TIMESTAMP        -- Current timestamp
AGE(timestamp)           -- Age from now
AGE(timestamp1, timestamp2)  -- Age between
DATE_TRUNC('hour', timestamp)  -- Truncate to hour
DATE_PART('year', timestamp)   -- Extract year
EXTRACT(YEAR FROM timestamp)   -- Extract year
TO_CHAR(timestamp, 'YYYY-MM-DD')  -- Format date
```

### String
```sql
CONCAT(str1, str2, ...)  -- Concatenate
CONCAT_WS(sep, str1, ...) -- Concatenate with separator
FORMAT(fmt, ...)         -- Format string
SPLIT_PART(string, delimiter, position)
REGEXP_SPLIT_TO_ARRAY(string, pattern)
INITCAP(string)          -- Capitalize words
REPEAT(string, n)        -- Repeat string
REVERSE(string)          -- Reverse string
MD5(string)              -- MD5 hash
```

### Math
```sql
ABS(n)                   -- Absolute value
CEIL(n)                  -- Ceiling
FLOOR(n)                 -- Floor
ROUND(n, decimals)       -- Round
TRUNC(n, decimals)       -- Truncate
RANDOM()                 -- Random 0-1
GREATEST(a, b, ...)      -- Greatest value
LEAST(a, b, ...)         -- Least value
```

### Type Casting
```sql
CAST(value AS type)      -- SQL standard
value::type              -- PostgreSQL shorthand

-- Common casts
'123'::INTEGER
'2024-01-01'::DATE
'{"key": "value"}'::JSONB
ARRAY[1,2,3]::INTEGER[]
```

## System Administration

### User Management
```sql
CREATE USER username WITH PASSWORD 'password';
CREATE ROLE rolename;
ALTER USER username WITH SUPERUSER;
ALTER USER username WITH CREATEDB;
ALTER USER username WITH REPLICATION;
GRANT ALL PRIVILEGES ON DATABASE dbname TO username;
GRANT SELECT, INSERT ON TABLE tablename TO username;
REVOKE ALL ON TABLE tablename FROM username;
DROP USER username;
```

### Database Management
```sql
CREATE DATABASE dbname;
CREATE DATABASE dbname OWNER username;
CREATE DATABASE dbname TEMPLATE template0 ENCODING 'UTF8';
ALTER DATABASE dbname SET timezone TO 'UTC';
DROP DATABASE dbname;
```

### Backup and Restore
```bash
# Backup
pg_dump dbname > backup.sql
pg_dump -Fc dbname > backup.dump           # Custom format
pg_dump -Fd -j 4 dbname -f backup_dir      # Directory format
pg_dumpall > all_databases.sql             # All databases

# Restore
psql dbname < backup.sql
pg_restore -d dbname backup.dump
pg_restore -d dbname -j 4 backup_dir       # Parallel restore
```

### Maintenance
```sql
VACUUM;                  -- Clean up dead rows
VACUUM FULL;             -- Reclaim disk space
VACUUM ANALYZE;          -- Update statistics
ANALYZE;                 -- Update statistics only
REINDEX TABLE tablename; -- Rebuild indexes
REINDEX DATABASE dbname; -- Rebuild all indexes
CLUSTER tablename USING indexname;  -- Physically reorder
```

### Configuration
```sql
SHOW ALL;                -- Show all settings
SHOW work_mem;           -- Show specific setting
SET work_mem = '256MB';  -- Session setting
ALTER SYSTEM SET work_mem = '256MB';  -- Persistent
SELECT pg_reload_conf(); -- Reload configuration
```

### Monitoring
```sql
-- Active queries
SELECT pid, usename, application_name, state, query
FROM pg_stat_activity WHERE state != 'idle';

-- Table sizes
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Index usage
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes ORDER BY idx_scan;

-- Cache hit ratio
SELECT
    sum(blks_hit) * 100.0 / sum(blks_hit + blks_read) AS cache_hit_ratio
FROM pg_stat_database;

-- Lock monitoring
SELECT
    blocked.pid AS blocked_pid,
    blocking.pid AS blocking_pid,
    blocked.query AS blocked_query,
    blocking.query AS blocking_query
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

## Performance Tuning

### Key Parameters
```sql
-- Memory
shared_buffers = '25% of RAM'        -- Shared memory
work_mem = '4MB'                     -- Per operation
maintenance_work_mem = '256MB'       -- Maintenance operations
effective_cache_size = '75% of RAM'  -- Planner hint

-- Connections
max_connections = 100                -- Maximum connections
max_parallel_workers = 8            -- Parallel query workers

-- Checkpoint
checkpoint_completion_target = 0.9   -- Spread checkpoint I/O
min_wal_size = '1GB'
max_wal_size = '4GB'

-- Planner
random_page_cost = 1.1               -- SSD = 1.1, HDD = 4
effective_io_concurrency = 200       -- SSD = 200, HDD = 2
```

### EXPLAIN Options
```sql
EXPLAIN SELECT ...;                   -- Basic plan
EXPLAIN (ANALYZE) SELECT ...;         -- With execution
EXPLAIN (ANALYZE, BUFFERS) SELECT ...; -- With buffer info
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT ...; -- All details
```

## Extensions

### Common Extensions
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";      -- UUID generation
CREATE EXTENSION IF NOT EXISTS "pgcrypto";       -- Cryptography
CREATE EXTENSION IF NOT EXISTS "pg_stat_statements"; -- Query stats
CREATE EXTENSION IF NOT EXISTS "pg_trgm";        -- Trigram search
CREATE EXTENSION IF NOT EXISTS "btree_gist";     -- GiST indexes
CREATE EXTENSION IF NOT EXISTS "citext";         -- Case-insensitive text
CREATE EXTENSION IF NOT EXISTS "hstore";         -- Key-value store
CREATE EXTENSION IF NOT EXISTS "tablefunc";      -- Crosstab
CREATE EXTENSION IF NOT EXISTS "postgres_fdw";   -- Foreign data wrapper
CREATE EXTENSION IF NOT EXISTS "pg_cron";        -- Job scheduler
```

## Emergency Commands

```sql
-- Kill query
SELECT pg_cancel_backend(pid);         -- Graceful
SELECT pg_terminate_backend(pid);      -- Force

-- Kill all connections to database
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'dbname' AND pid <> pg_backend_pid();

-- Reset statistics
SELECT pg_stat_reset();

-- Force checkpoint
CHECKPOINT;

-- Switch WAL file
SELECT pg_switch_wal();

-- Check recovery status
SELECT pg_is_in_recovery();

-- Promote standby
SELECT pg_promote();
```

Keep this reference handy—you'll need it more often than you think!