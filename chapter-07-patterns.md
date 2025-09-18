# Chapter 7: Real-World Patterns and Best Practices

Seven chapters in, you know PostgreSQL. Now let's talk about using it in production without losing sleep, hair, or data.

This chapter covers the patterns that separate hobby projects from production systems.

## Database Design Patterns

### The UUID vs Serial ID Debate

```sql
-- Option 1: Serial IDs (default)
CREATE TABLE users_serial (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE
);

-- Pros: Small, fast, human-readable
-- Cons: Reveals count, enumerable, not globally unique

-- Option 2: UUIDs
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users_uuid (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    email VARCHAR(255) UNIQUE
);

-- Pros: Globally unique, non-enumerable, distributed-friendly
-- Cons: Larger (16 bytes vs 4), harder to debug, index fragmentation

-- Option 3: Best of both worlds
CREATE TABLE users_hybrid (
    id SERIAL PRIMARY KEY,  -- Internal use
    public_id UUID DEFAULT uuid_generate_v4() UNIQUE,  -- External API
    email VARCHAR(255) UNIQUE
);
```

### Soft Deletes Pattern

Never actually delete data (until you have to):

```sql
-- Add deleted_at column
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP;

-- "Delete" by setting timestamp
UPDATE users SET deleted_at = CURRENT_TIMESTAMP WHERE id = 123;

-- Create view for active records
CREATE VIEW active_users AS
SELECT * FROM users WHERE deleted_at IS NULL;

-- Create index for performance
CREATE INDEX idx_users_active ON users(id) WHERE deleted_at IS NULL;

-- Periodic cleanup (actually delete old soft-deleted records)
DELETE FROM users
WHERE deleted_at < CURRENT_DATE - INTERVAL '90 days';
```

### Audit Trail Pattern

Track every change for compliance and debugging:

```sql
-- Audit table
CREATE TABLE audit_logs (
    id SERIAL PRIMARY KEY,
    table_name VARCHAR(50) NOT NULL,
    record_id INTEGER NOT NULL,
    action VARCHAR(10) NOT NULL,
    old_data JSONB,
    new_data JSONB,
    changed_by VARCHAR(100),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Generic audit function
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        INSERT INTO audit_logs(table_name, record_id, action, old_data, changed_by)
        VALUES (TG_TABLE_NAME, OLD.id, TG_OP, to_jsonb(OLD), current_user);
        RETURN OLD;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_logs(table_name, record_id, action, old_data, new_data, changed_by)
        VALUES (TG_TABLE_NAME, NEW.id, TG_OP, to_jsonb(OLD), to_jsonb(NEW), current_user);
        RETURN NEW;
    ELSIF TG_OP = 'INSERT' THEN
        INSERT INTO audit_logs(table_name, record_id, action, new_data, changed_by)
        VALUES (TG_TABLE_NAME, NEW.id, TG_OP, to_jsonb(NEW), current_user);
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Apply to any table
CREATE TRIGGER users_audit_trigger
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();
```

### Multi-Tenant Patterns

#### Option 1: Shared Schema with Tenant ID

```sql
-- Every table has tenant_id
CREATE TABLE tenants (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE,
    plan VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER REFERENCES tenants(id),
    name VARCHAR(200),
    price DECIMAL(10,2)
);

-- Row-level security
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON products
FOR ALL
USING (tenant_id = current_setting('app.tenant_id')::INT);

-- Set tenant for session
SET app.tenant_id = 123;
```

#### Option 2: Schema per Tenant

```sql
-- Create tenant schema
CREATE SCHEMA tenant_123;

-- Create tables in tenant schema
CREATE TABLE tenant_123.products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200),
    price DECIMAL(10,2)
);

-- Set search path for tenant
SET search_path TO tenant_123, public;
```

### Event Sourcing Pattern

Store events, not state:

```sql
-- Events table
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    aggregate_type VARCHAR(50) NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    event_data JSONB NOT NULL,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Index for fast aggregate queries
CREATE INDEX idx_events_aggregate ON events(aggregate_id, created_at);

-- Insert events
INSERT INTO events (aggregate_id, aggregate_type, event_type, event_data)
VALUES
    ('123e4567-e89b-12d3-a456-426614174000', 'Order', 'OrderCreated',
     '{"customer_id": 1, "total": 99.99}'::jsonb),
    ('123e4567-e89b-12d3-a456-426614174000', 'Order', 'OrderShipped',
     '{"tracking_number": "ABC123"}'::jsonb);

-- Rebuild state from events
WITH order_events AS (
    SELECT
        event_type,
        event_data,
        created_at
    FROM events
    WHERE aggregate_id = '123e4567-e89b-12d3-a456-426614174000'
    ORDER BY created_at
)
SELECT
    jsonb_agg(event_data ORDER BY created_at) AS state_history
FROM order_events;
```

## Connection Management

### Connection Pooling with PgBouncer

Never connect directly from your app in production:

```ini
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
```

### Application Connection Best Practices

```python
# Python example with psycopg2
import psycopg2
from psycopg2 import pool
from contextlib import contextmanager

# Create connection pool
connection_pool = psycopg2.pool.ThreadedConnectionPool(
    5,  # Min connections
    20,  # Max connections
    host='localhost',
    port=6432,  # PgBouncer port
    database='mydb',
    user='appuser',
    password='secret'
)

@contextmanager
def get_db_connection():
    connection = connection_pool.getconn()
    try:
        yield connection
    finally:
        connection_pool.putconn(connection)

# Usage
with get_db_connection() as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT * FROM users")
        users = cur.fetchall()
```

## Backup and Recovery Strategies

### Backup Types

```bash
# 1. Logical backup with pg_dump (portable, slower)
pg_dump -Fc -f backup.dump mydb

# 2. Physical backup with pg_basebackup (fast, exact copy)
pg_basebackup -D /backup/location -Ft -Xs -P

# 3. Continuous archiving (point-in-time recovery)
# In postgresql.conf:
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'

# 4. Incremental backup with pgBackRest
pgbackrest --stanza=main backup --type=incr
```

### Automated Backup Script

```bash
#!/bin/bash
# backup.sh - Daily backup script

BACKUP_DIR="/backups/postgres"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DB_NAME="production"

# Create backup
pg_dump -Fc -f "$BACKUP_DIR/backup_$TIMESTAMP.dump" $DB_NAME

# Keep only last 7 days
find $BACKUP_DIR -name "backup_*.dump" -mtime +7 -delete

# Upload to S3 (optional)
aws s3 cp "$BACKUP_DIR/backup_$TIMESTAMP.dump" \
    s3://my-backup-bucket/postgres/

# Test the backup
pg_restore --list "$BACKUP_DIR/backup_$TIMESTAMP.dump" > /dev/null
if [ $? -eq 0 ]; then
    echo "Backup successful: backup_$TIMESTAMP.dump"
else
    echo "Backup failed!" | mail -s "Backup Failure" admin@example.com
fi
```

### Restore Procedures

```bash
# Restore from pg_dump
pg_restore -d newdb backup.dump

# Selective restore
pg_restore -t specific_table -d mydb backup.dump

# Point-in-time recovery
# 1. Restore base backup
# 2. Configure recovery.conf
# 3. PostgreSQL replays WAL files to specified time
```

## Security Best Practices

### User and Permission Management

```sql
-- Create application user with minimal privileges
CREATE USER app_user WITH PASSWORD 'strong_password';

-- Create read-only user
CREATE USER readonly_user WITH PASSWORD 'another_strong_password';

-- Grant permissions
GRANT CONNECT ON DATABASE mydb TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;

-- Read-only permissions
GRANT CONNECT ON DATABASE mydb TO readonly_user;
GRANT USAGE ON SCHEMA public TO readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;

-- Revoke dangerous permissions
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE mydb FROM PUBLIC;
```

### SSL/TLS Configuration

```conf
# postgresql.conf
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'root.crt'

# pg_hba.conf - Require SSL
hostssl all all 0.0.0.0/0 md5
```

### Data Encryption

```sql
-- Encrypt sensitive data
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Store encrypted passwords
INSERT INTO users (email, password_hash)
VALUES ('user@example.com', crypt('userpassword', gen_salt('bf')));

-- Verify password
SELECT id, email
FROM users
WHERE email = 'user@example.com'
AND password_hash = crypt('userpassword', password_hash);

-- Encrypt columns
CREATE TABLE sensitive_data (
    id SERIAL PRIMARY KEY,
    encrypted_ssn BYTEA
);

INSERT INTO sensitive_data (encrypted_ssn)
VALUES (pgp_sym_encrypt('123-45-6789', 'encryption_key'));

-- Decrypt
SELECT pgp_sym_decrypt(encrypted_ssn, 'encryption_key') AS ssn
FROM sensitive_data;
```

## Monitoring and Alerting

### Essential Monitoring Queries

```sql
-- Create monitoring schema
CREATE SCHEMA monitoring;

-- Database health check
CREATE VIEW monitoring.health_check AS
SELECT
    current_database() AS database,
    pg_database_size(current_database()) AS size_bytes,
    pg_size_pretty(pg_database_size(current_database())) AS size_pretty,
    (SELECT count(*) FROM pg_stat_activity) AS connections,
    (SELECT count(*) FROM pg_stat_activity WHERE state = 'active') AS active_queries,
    (SELECT max(age(clock_timestamp(), query_start))
     FROM pg_stat_activity
     WHERE state = 'active') AS longest_query,
    (SELECT count(*) FROM pg_stat_activity WHERE waiting) AS waiting_queries;

-- Slow query monitor
CREATE VIEW monitoring.slow_queries AS
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    stddev_exec_time,
    max_exec_time
FROM pg_stat_statements
WHERE mean_exec_time > 1000  -- Queries averaging > 1 second
ORDER BY mean_exec_time DESC;

-- Table bloat monitor
CREATE VIEW monitoring.table_bloat AS
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_percentage
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

### Alert Conditions

```sql
-- Alert if connections near limit
SELECT
    current_setting('max_connections')::int AS max,
    count(*) AS current,
    CASE
        WHEN count(*) > current_setting('max_connections')::int * 0.8
        THEN 'WARNING: Connection pool near limit'
        ELSE 'OK'
    END AS status
FROM pg_stat_activity;

-- Alert on replication lag
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- Alert on long-running transactions
SELECT
    pid,
    usename,
    application_name,
    state,
    age(clock_timestamp(), xact_start) AS transaction_age,
    query
FROM pg_stat_activity
WHERE xact_start < clock_timestamp() - INTERVAL '1 hour';
```

## Migration Strategies

### Zero-Downtime Migrations

```sql
-- 1. Add column (instant)
ALTER TABLE users ADD COLUMN new_field VARCHAR(100);

-- 2. Backfill in batches (to avoid locks)
DO $$
DECLARE
    batch_size INT := 1000;
    offset_val INT := 0;
    total_rows INT;
BEGIN
    SELECT COUNT(*) INTO total_rows FROM users;

    WHILE offset_val < total_rows LOOP
        UPDATE users
        SET new_field = 'default_value'
        WHERE id IN (
            SELECT id FROM users
            WHERE new_field IS NULL
            LIMIT batch_size
        );

        offset_val := offset_val + batch_size;
        PERFORM pg_sleep(0.1);  -- Brief pause
    END LOOP;
END $$;

-- 3. Add NOT NULL constraint after backfill
ALTER TABLE users ALTER COLUMN new_field SET NOT NULL;

-- 4. Create index concurrently
CREATE INDEX CONCURRENTLY idx_users_new_field ON users(new_field);
```

### Blue-Green Deployment Pattern

```sql
-- Create new version of table
CREATE TABLE users_v2 AS SELECT * FROM users;

-- Add new columns/changes
ALTER TABLE users_v2 ADD COLUMN feature_flag BOOLEAN DEFAULT FALSE;

-- Sync data using triggers
CREATE OR REPLACE FUNCTION sync_users()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO users_v2 SELECT NEW.*;
    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE users_v2 SET * = NEW.* WHERE id = NEW.id;
    ELSIF TG_OP = 'DELETE' THEN
        DELETE FROM users_v2 WHERE id = OLD.id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER sync_users_trigger
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION sync_users();

-- Switch when ready
BEGIN;
ALTER TABLE users RENAME TO users_old;
ALTER TABLE users_v2 RENAME TO users;
COMMIT;
```

## Development vs Production

### Environment-Specific Configurations

```sql
-- Development settings (postgresql.conf)
log_statement = 'all'  -- Log everything for debugging
log_duration = on
shared_buffers = 256MB  -- Smaller for dev machines
work_mem = 4MB

-- Production settings
log_statement = 'ddl'  -- Only log schema changes
log_min_duration_statement = 1000  -- Log slow queries > 1s
shared_buffers = 8GB  -- 25% of RAM
work_mem = 32MB
maintenance_work_mem = 2GB
effective_cache_size = 24GB  -- 75% of RAM
```

### Data Masking for Development

```sql
-- Create sanitized development database
CREATE OR REPLACE FUNCTION mask_sensitive_data()
RETURNS void AS $$
BEGIN
    -- Mask emails
    UPDATE users
    SET email = 'user' || id || '@example.com';

    -- Mask names
    UPDATE users
    SET name = 'User ' || id;

    -- Mask phone numbers
    UPDATE users
    SET phone = '+1555' || LPAD(id::text, 7, '0');

    -- Randomize sensitive data
    UPDATE orders
    SET total_amount = random() * 1000;
END;
$$ LANGUAGE plpgsql;

-- Run after restoring production backup to dev
SELECT mask_sensitive_data();
```

## Documentation and Knowledge Management

### Self-Documenting Database

```sql
-- Add comments to everything
COMMENT ON TABLE users IS 'Core user accounts table';
COMMENT ON COLUMN users.email IS 'Primary email, used for login';
COMMENT ON COLUMN users.status IS 'active, suspended, or deleted';

-- Document constraints
ALTER TABLE orders
ADD CONSTRAINT orders_total_positive CHECK (total_amount >= 0);
COMMENT ON CONSTRAINT orders_total_positive ON orders
IS 'Ensures order totals cannot be negative';

-- Create data dictionary view
CREATE VIEW data_dictionary AS
SELECT
    c.table_schema,
    c.table_name,
    c.column_name,
    c.data_type,
    c.is_nullable,
    c.column_default,
    obj.description AS column_comment
FROM information_schema.columns c
LEFT JOIN pg_catalog.pg_class cls ON cls.relname = c.table_name
LEFT JOIN pg_catalog.pg_namespace ns ON ns.oid = cls.relnamespace
LEFT JOIN pg_catalog.pg_description obj ON obj.objoid = cls.oid
    AND obj.objsubid = c.ordinal_position
WHERE c.table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY c.table_schema, c.table_name, c.ordinal_position;
```

## The Production Readiness Checklist

Before going live:

- [ ] Backups automated and tested
- [ ] Monitoring and alerting configured
- [ ] Connection pooling implemented
- [ ] SSL/TLS enabled
- [ ] Users and permissions properly configured
- [ ] Indexes created and analyzed
- [ ] VACUUM and maintenance scheduled
- [ ] Replication configured (if needed)
- [ ] Disaster recovery plan documented
- [ ] Performance baseline established
- [ ] Security audit completed
- [ ] Documentation up to date

## What You've Learned

You now have the knowledge to:
- Design robust database schemas
- Implement production-grade patterns
- Secure your database properly
- Monitor and maintain database health
- Perform zero-downtime migrations
- Handle backups and disaster recovery
- Scale from development to production

## Conclusion: The Journey Continues

You've traveled from "what's a database?" to production-ready PostgreSQL knowledge. But this isn't the end—it's the beginning of your journey as someone who truly understands data management.

PostgreSQL will continue evolving, but its core principles remain: correctness, reliability, and extensibility. Master these, and you'll be ready for whatever comes next.

Remember: every query you write, every index you create, every backup you configure is building toward systems that serve real users solving real problems. Use your power wisely.

*"May your queries be fast, your data consistent, and your backups always restorable."*

## Final Resources

- **Official PostgreSQL Documentation**: postgresql.org/docs
- **PostgreSQL Weekly**: postgresweekly.com
- **Planet PostgreSQL**: planet.postgresql.org
- **PostgreSQL Slack**: postgresteam.slack.com
- **Performance Tuning**: pgtune.leopard.in.ua

Now go forth and build something amazing with PostgreSQL. The elephant never forgets, and neither will you forget what you've learned here.

*End of Guide*