# Chapter 11: Real-World Patterns

*By Claude Code Opus 4.1*

After ten chapters of theory and practice, let's tackle the patterns that separate production systems from prototypes. These are the lessons learned from millions of queries, terabytes of data, and countless 3 AM incidents.

## The UUID vs Serial ID Debate

One of the first decisions in any database design: how do you identify your records?

### Serial IDs (Traditional Approach)

```sql
-- Simple incrementing integers
CREATE TABLE users_serial (
    id SERIAL PRIMARY KEY,  -- or BIGSERIAL for larger ranges
    email TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Pros:
-- ✓ Small size (4 bytes for INT, 8 bytes for BIGINT)
-- ✓ Fast indexing and joins
-- ✓ Human-readable (easy debugging)
-- ✓ Natural ordering by creation time
-- ✓ Better index locality (sequential inserts)

-- Cons:
-- ✗ Reveals business information (user count, growth rate)
-- ✗ Enumerable (security risk for public APIs)
-- ✗ Not globally unique across databases
-- ✗ Difficult to merge/shard databases
-- ✗ Can't generate IDs client-side
```

### UUIDs (Distributed-Friendly)

```sql
-- Universally Unique Identifiers
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE users_uuid (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Alternative: gen_random_uuid() in PostgreSQL 13+
CREATE TABLE users_uuid_v2 (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    email TEXT UNIQUE NOT NULL
);

-- Pros:
-- ✓ Globally unique across all systems
-- ✓ Can generate IDs anywhere (client, server, offline)
-- ✓ No information leakage
-- ✓ Easy database merging/sharding
-- ✓ No central coordination needed

-- Cons:
-- ✗ Larger size (16 bytes)
-- ✗ Harder to read/debug (not human-friendly)
-- ✗ Random insertion pattern (index fragmentation)
-- ✗ No natural time ordering
-- ✗ Slightly slower joins
```

### Hybrid Approach: Best of Both Worlds

```sql
-- Internal serial ID for joins, public UUID for APIs
CREATE TABLE users_hybrid (
    id BIGSERIAL PRIMARY KEY,                         -- Internal use only
    public_id UUID DEFAULT gen_random_uuid() UNIQUE,  -- External API
    email TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_users_public_id ON users_hybrid(public_id);

-- Internal queries use fast integer joins
SELECT o.*
FROM orders o
JOIN users_hybrid u ON o.user_id = u.id
WHERE u.email = 'user@example.com';

-- API exposes only UUIDs
SELECT public_id, email FROM users_hybrid
WHERE public_id = '550e8400-e29b-41d4-a716-446655440000';
```

### Advanced: Ordered UUIDs (ULIDs/UUID v6/v7)

```sql
-- ULID: Universally Unique Lexicographically Sortable Identifier
CREATE OR REPLACE FUNCTION generate_ulid()
RETURNS TEXT AS $$
DECLARE
    timestamp BIGINT;
    random_part TEXT;
BEGIN
    -- Timestamp in milliseconds (48 bits)
    timestamp := EXTRACT(EPOCH FROM NOW() * 1000)::BIGINT;

    -- Random part (80 bits)
    random_part := encode(gen_random_bytes(10), 'hex');

    -- Combine and encode (base32 for real ULID)
    RETURN lpad(to_hex(timestamp), 12, '0') || random_part;
END;
$$ LANGUAGE plpgsql;

-- UUID v7 (time-ordered UUID) - coming in PostgreSQL 17
-- For now, use a custom implementation
CREATE OR REPLACE FUNCTION uuid_generate_v7()
RETURNS UUID AS $$
DECLARE
    unix_ts_ms BIGINT;
    uuid_bytes BYTEA;
BEGIN
    unix_ts_ms := EXTRACT(EPOCH FROM NOW() * 1000)::BIGINT;

    -- Timestamp (48 bits) + version (4 bits) + random (74 bits)
    uuid_bytes :=
        int8send(unix_ts_ms << 16) ||  -- Timestamp
        gen_random_bytes(10);           -- Random

    -- Set version (7) and variant bits
    uuid_bytes := set_byte(uuid_bytes, 6,
        (get_byte(uuid_bytes, 6) & 15) | 112);  -- Version 7
    uuid_bytes := set_byte(uuid_bytes, 8,
        (get_byte(uuid_bytes, 8) & 63) | 128);  -- Variant

    RETURN encode(uuid_bytes, 'hex')::UUID;
END;
$$ LANGUAGE plpgsql;

-- Usage: Time-ordered but still globally unique
CREATE TABLE events (
    id UUID DEFAULT uuid_generate_v7() PRIMARY KEY,
    event_type TEXT NOT NULL,
    payload JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Benefits: Sequential insertion pattern like serial IDs
-- but with global uniqueness like UUIDs
```

### Decision Matrix

```sql
-- Choose Serial IDs when:
-- • Internal system with no public API
-- • Performance is critical
-- • Database will never be sharded
-- • IDs never exposed to users

-- Choose UUIDs when:
-- • Building distributed systems
-- • Need client-side ID generation
-- • Public API exposes IDs
-- • Planning for database sharding

-- Choose Hybrid when:
-- • Need both performance and security
-- • Willing to manage complexity
-- • Have clear internal/external boundaries

-- Choose Ordered UUIDs when:
-- • Need UUID benefits
-- • But also want better index performance
-- • Time-ordering is useful
```

## Multi-Tenancy Patterns

### Schema-per-Tenant

Perfect for B2B SaaS with strict isolation requirements:

```sql
-- Create tenant schema
CREATE SCHEMA tenant_acme;

-- Set search path for tenant session
SET search_path TO tenant_acme, public;

-- Tenant-specific tables
CREATE TABLE tenant_acme.users (
    id bigserial PRIMARY KEY,
    email text UNIQUE NOT NULL,
    created_at timestamptz DEFAULT now()
);

-- Automated tenant provisioning
CREATE OR REPLACE PROCEDURE provision_tenant(
    tenant_name text
) AS $$
DECLARE
    schema_name text;
BEGIN
    -- Sanitize schema name
    schema_name := 'tenant_' || regexp_replace(lower(tenant_name), '[^a-z0-9]', '_', 'g');

    -- Create schema
    EXECUTE format('CREATE SCHEMA %I', schema_name);

    -- Create tables
    EXECUTE format('
        CREATE TABLE %I.users (LIKE public.users_template INCLUDING ALL)',
        schema_name
    );

    EXECUTE format('
        CREATE TABLE %I.products (LIKE public.products_template INCLUDING ALL)',
        schema_name
    );

    -- Set up RLS policies
    EXECUTE format('
        ALTER TABLE %I.users ENABLE ROW LEVEL SECURITY',
        schema_name
    );

    RAISE NOTICE 'Tenant % provisioned successfully', tenant_name;
END;
$$ LANGUAGE plpgsql;
```

### Row-Level Multi-Tenancy

More scalable for thousands of tenants:

```sql
-- Shared tables with tenant_id
CREATE TABLE users (
    id bigserial PRIMARY KEY,
    tenant_id integer NOT NULL,
    email text NOT NULL,
    created_at timestamptz DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);

-- RLS for automatic tenant isolation
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
    FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::integer);

-- Application sets tenant context
SET app.tenant_id = '42';

-- Now all queries automatically filtered
SELECT * FROM users; -- Only sees tenant 42's users
```

### Hybrid Approach

```sql
-- Shared data in public schema
CREATE TABLE public.tenants (
    id serial PRIMARY KEY,
    name text UNIQUE NOT NULL,
    plan text NOT NULL,
    storage_gb integer DEFAULT 10,
    created_at timestamptz DEFAULT now()
);

-- Critical data in tenant schemas
CREATE OR REPLACE FUNCTION get_tenant_schema(tenant_id integer)
RETURNS text AS $$
    SELECT 'tenant_' || id FROM public.tenants WHERE id = tenant_id;
$$ LANGUAGE sql STABLE;

-- Dynamic cross-tenant queries
CREATE OR REPLACE FUNCTION get_all_tenant_stats()
RETURNS TABLE(
    tenant_id integer,
    user_count bigint,
    storage_used_gb numeric
) AS $$
DECLARE
    tenant record;
    query text;
BEGIN
    FOR tenant IN SELECT id, 'tenant_' || id AS schema_name FROM public.tenants LOOP
        query := format('
            SELECT %s, count(*), pg_database_size(current_database())/1024/1024/1024.0
            FROM %I.users',
            tenant.id, tenant.schema_name
        );

        RETURN QUERY EXECUTE query;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

## Event Sourcing Pattern

Store events, not state:

```sql
-- Events table (append-only)
CREATE TABLE events (
    id bigserial PRIMARY KEY,
    aggregate_id uuid NOT NULL,
    aggregate_type text NOT NULL,
    event_type text NOT NULL,
    event_data jsonb NOT NULL,
    metadata jsonb,
    created_at timestamptz DEFAULT now()
);

CREATE INDEX idx_events_aggregate ON events(aggregate_id, created_at);

-- Immutability enforced
CREATE RULE events_no_update AS ON UPDATE TO events DO INSTEAD NOTHING;
CREATE RULE events_no_delete AS ON DELETE TO events DO INSTEAD NOTHING;

-- Event creation
CREATE OR REPLACE FUNCTION emit_event(
    p_aggregate_id uuid,
    p_aggregate_type text,
    p_event_type text,
    p_event_data jsonb,
    p_metadata jsonb DEFAULT '{}'
) RETURNS bigint AS $$
DECLARE
    event_id bigint;
BEGIN
    INSERT INTO events (
        aggregate_id,
        aggregate_type,
        event_type,
        event_data,
        metadata
    ) VALUES (
        p_aggregate_id,
        p_aggregate_type,
        p_event_type,
        p_event_data,
        p_metadata || jsonb_build_object(
            'user_id', current_setting('app.user_id', true),
            'ip_address', inet_client_addr()
        )
    ) RETURNING id INTO event_id;

    -- Trigger projections update
    PERFORM pg_notify('event_created', json_build_object(
        'event_id', event_id,
        'aggregate_id', p_aggregate_id,
        'event_type', p_event_type
    )::text);

    RETURN event_id;
END;
$$ LANGUAGE plpgsql;

-- Rebuild current state from events
CREATE MATERIALIZED VIEW current_account_state AS
WITH account_events AS (
    SELECT
        aggregate_id,
        event_type,
        event_data,
        created_at,
        ROW_NUMBER() OVER (PARTITION BY aggregate_id ORDER BY created_at) AS seq
    FROM events
    WHERE aggregate_type = 'account'
)
SELECT
    aggregate_id AS account_id,
    (event_data->>'balance')::decimal AS current_balance,
    MAX(created_at) AS last_modified
FROM account_events
WHERE (aggregate_id, seq) IN (
    SELECT aggregate_id, MAX(seq)
    FROM account_events
    GROUP BY aggregate_id
)
GROUP BY aggregate_id, event_data;

CREATE UNIQUE INDEX idx_current_account_state ON current_account_state(account_id);
```

## CQRS (Command Query Responsibility Segregation)

Separate read and write models:

```sql
-- Write model (normalized)
CREATE TABLE orders (
    id bigserial PRIMARY KEY,
    customer_id bigint NOT NULL,
    status text NOT NULL,
    created_at timestamptz DEFAULT now()
);

CREATE TABLE order_items (
    id bigserial PRIMARY KEY,
    order_id bigint REFERENCES orders(id),
    product_id bigint NOT NULL,
    quantity integer NOT NULL,
    price_at_time decimal(10,2) NOT NULL
);

-- Read model (denormalized)
CREATE MATERIALIZED VIEW order_summary AS
SELECT
    o.id,
    o.customer_id,
    c.name AS customer_name,
    o.status,
    o.created_at,
    COUNT(oi.id) AS item_count,
    SUM(oi.quantity) AS total_items,
    SUM(oi.quantity * oi.price_at_time) AS total_amount,
    jsonb_agg(jsonb_build_object(
        'product_id', oi.product_id,
        'product_name', p.name,
        'quantity', oi.quantity,
        'price', oi.price_at_time
    )) AS items
FROM orders o
JOIN customers c ON c.id = o.customer_id
LEFT JOIN order_items oi ON oi.order_id = o.id
LEFT JOIN products p ON p.id = oi.product_id
GROUP BY o.id, o.customer_id, c.name, o.status, o.created_at;

CREATE INDEX idx_order_summary_customer ON order_summary(customer_id);
CREATE INDEX idx_order_summary_status ON order_summary(status);

-- Refresh strategy
CREATE OR REPLACE FUNCTION refresh_order_summary()
RETURNS trigger AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY order_summary;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER refresh_order_summary_trigger
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH STATEMENT
EXECUTE FUNCTION refresh_order_summary();
```

## Time-Series Data Pattern

Optimized for append-heavy, time-based queries:

```sql
-- Partitioned time-series table
CREATE TABLE metrics (
    time timestamptz NOT NULL,
    device_id integer NOT NULL,
    temperature numeric,
    humidity numeric,
    pressure numeric
) PARTITION BY RANGE (time);

-- Create partitions automatically
CREATE OR REPLACE FUNCTION create_monthly_partition()
RETURNS void AS $$
DECLARE
    start_date date;
    end_date date;
    partition_name text;
BEGIN
    start_date := date_trunc('month', CURRENT_DATE);
    end_date := start_date + interval '1 month';
    partition_name := 'metrics_' || to_char(start_date, 'YYYY_MM');

    IF NOT EXISTS (
        SELECT 1 FROM pg_class WHERE relname = partition_name
    ) THEN
        EXECUTE format('
            CREATE TABLE %I PARTITION OF metrics
            FOR VALUES FROM (%L) TO (%L)',
            partition_name, start_date, end_date
        );

        -- Create indexes on partition
        EXECUTE format('
            CREATE INDEX %I ON %I (device_id, time DESC)',
            partition_name || '_device_time_idx',
            partition_name
        );
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Schedule partition creation
CREATE EXTENSION IF NOT EXISTS pg_cron;
SELECT cron.schedule('create-partitions', '0 0 1 * *', 'SELECT create_monthly_partition()');

-- Efficient aggregation
CREATE MATERIALIZED VIEW hourly_metrics AS
SELECT
    time_bucket('1 hour', time) AS hour,
    device_id,
    avg(temperature) AS avg_temp,
    avg(humidity) AS avg_humidity,
    avg(pressure) AS avg_pressure,
    count(*) AS sample_count
FROM metrics
WHERE time > now() - interval '7 days'
GROUP BY hour, device_id;

CREATE INDEX idx_hourly_metrics ON hourly_metrics(device_id, hour DESC);

-- Automatic old partition cleanup
CREATE OR REPLACE FUNCTION drop_old_partitions()
RETURNS void AS $$
DECLARE
    partition record;
BEGIN
    FOR partition IN
        SELECT tablename
        FROM pg_tables
        WHERE tablename LIKE 'metrics_%'
        AND tablename < 'metrics_' || to_char(CURRENT_DATE - interval '6 months', 'YYYY_MM')
    LOOP
        EXECUTE format('DROP TABLE %I', partition.tablename);
        RAISE NOTICE 'Dropped partition %', partition.tablename;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

## Queue Pattern

Database as a job queue:

```sql
-- Job queue table
CREATE TABLE job_queue (
    id bigserial PRIMARY KEY,
    queue_name text NOT NULL DEFAULT 'default',
    payload jsonb NOT NULL,
    status text NOT NULL DEFAULT 'pending',
    priority integer DEFAULT 0,
    retry_count integer DEFAULT 0,
    max_retries integer DEFAULT 3,
    scheduled_at timestamptz DEFAULT now(),
    locked_until timestamptz,
    locked_by text,
    completed_at timestamptz,
    failed_at timestamptz,
    error_message text,
    created_at timestamptz DEFAULT now(),
    CONSTRAINT valid_status CHECK (status IN ('pending', 'processing', 'completed', 'failed'))
);

CREATE INDEX idx_queue_fetch ON job_queue(queue_name, status, priority DESC, scheduled_at)
WHERE status = 'pending';

-- Claim next job atomically
CREATE OR REPLACE FUNCTION claim_job(
    p_queue_name text DEFAULT 'default',
    p_worker_id text DEFAULT 'worker',
    p_lock_duration interval DEFAULT '5 minutes'
) RETURNS job_queue AS $$
DECLARE
    job job_queue;
BEGIN
    SELECT * INTO job
    FROM job_queue
    WHERE queue_name = p_queue_name
      AND status = 'pending'
      AND scheduled_at <= now()
      AND (locked_until IS NULL OR locked_until < now())
    ORDER BY priority DESC, scheduled_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED;

    IF FOUND THEN
        UPDATE job_queue
        SET status = 'processing',
            locked_by = p_worker_id,
            locked_until = now() + p_lock_duration
        WHERE id = job.id;
    END IF;

    RETURN job;
END;
$$ LANGUAGE plpgsql;

-- Complete job
CREATE OR REPLACE FUNCTION complete_job(
    p_job_id bigint,
    p_result jsonb DEFAULT '{}'
) RETURNS void AS $$
BEGIN
    UPDATE job_queue
    SET status = 'completed',
        completed_at = now(),
        payload = payload || jsonb_build_object('result', p_result)
    WHERE id = p_job_id;
END;
$$ LANGUAGE plpgsql;

-- Retry failed job
CREATE OR REPLACE FUNCTION retry_job(
    p_job_id bigint,
    p_error text
) RETURNS void AS $$
BEGIN
    UPDATE job_queue
    SET
        status = CASE
            WHEN retry_count < max_retries THEN 'pending'
            ELSE 'failed'
        END,
        retry_count = retry_count + 1,
        scheduled_at = now() + (interval '1 minute' * power(2, retry_count)),
        error_message = p_error,
        failed_at = CASE
            WHEN retry_count >= max_retries THEN now()
            ELSE NULL
        END,
        locked_until = NULL,
        locked_by = NULL
    WHERE id = p_job_id;
END;
$$ LANGUAGE plpgsql;
```

## Audit Logging Pattern

Track every change:

```sql
-- Generic audit table
CREATE TABLE audit_log (
    id bigserial PRIMARY KEY,
    table_name text NOT NULL,
    record_id text NOT NULL,
    action text NOT NULL CHECK (action IN ('INSERT', 'UPDATE', 'DELETE')),
    old_data jsonb,
    new_data jsonb,
    changed_fields text[],
    user_id text,
    user_ip inet,
    user_agent text,
    created_at timestamptz DEFAULT now()
);

CREATE INDEX idx_audit_table_record ON audit_log(table_name, record_id, created_at DESC);
CREATE INDEX idx_audit_user ON audit_log(user_id, created_at DESC);

-- Generic audit trigger function
CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS trigger AS $$
DECLARE
    record_id text;
    old_data jsonb;
    new_data jsonb;
    changed_fields text[];
BEGIN
    -- Get record ID (assumes 'id' column)
    IF TG_OP = 'DELETE' THEN
        record_id := OLD.id::text;
        old_data := to_jsonb(OLD);
        new_data := NULL;
    ELSIF TG_OP = 'INSERT' THEN
        record_id := NEW.id::text;
        old_data := NULL;
        new_data := to_jsonb(NEW);
    ELSE -- UPDATE
        record_id := NEW.id::text;
        old_data := to_jsonb(OLD);
        new_data := to_jsonb(NEW);

        -- Calculate changed fields
        SELECT array_agg(key) INTO changed_fields
        FROM jsonb_each(old_data) o
        FULL OUTER JOIN jsonb_each(new_data) n USING (key)
        WHERE o.value IS DISTINCT FROM n.value;
    END IF;

    INSERT INTO audit_log (
        table_name,
        record_id,
        action,
        old_data,
        new_data,
        changed_fields,
        user_id,
        user_ip
    ) VALUES (
        TG_TABLE_NAME,
        record_id,
        TG_OP,
        old_data,
        new_data,
        changed_fields,
        current_setting('app.user_id', true),
        inet_client_addr()
    );

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to any table
CREATE TRIGGER users_audit
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_trigger();

-- Query audit history
CREATE OR REPLACE FUNCTION get_record_history(
    p_table text,
    p_record_id text
) RETURNS TABLE(
    version integer,
    action text,
    changed_fields text[],
    user_id text,
    created_at timestamptz,
    data jsonb
) AS $$
BEGIN
    RETURN QUERY
    WITH history AS (
        SELECT
            ROW_NUMBER() OVER (ORDER BY created_at) AS version,
            action,
            changed_fields,
            user_id,
            created_at,
            COALESCE(new_data, old_data) AS data
        FROM audit_log
        WHERE table_name = p_table
          AND record_id = p_record_id
        ORDER BY created_at
    )
    SELECT * FROM history;
END;
$$ LANGUAGE plpgsql;
```

## Hierarchical Data Patterns

### Adjacency List with Recursive CTEs

```sql
-- Simple parent-child relationship
CREATE TABLE categories (
    id serial PRIMARY KEY,
    name text NOT NULL,
    parent_id integer REFERENCES categories(id)
);

CREATE INDEX idx_categories_parent ON categories(parent_id);

-- Get full tree
WITH RECURSIVE category_tree AS (
    -- Anchor: root categories
    SELECT id, name, parent_id, 0 AS level, name AS path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- Recursive: child categories
    SELECT c.id, c.name, c.parent_id, ct.level + 1,
           ct.path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY path;
```

### Materialized Path Pattern

```sql
-- Store full path for each node
CREATE TABLE categories_path (
    id serial PRIMARY KEY,
    name text NOT NULL,
    path text NOT NULL, -- '1.2.3'
    depth integer GENERATED ALWAYS AS (
        array_length(string_to_array(path, '.'), 1)
    ) STORED
);

CREATE INDEX idx_categories_path ON categories_path(path text_pattern_ops);

-- Get all descendants efficiently
SELECT * FROM categories_path
WHERE path LIKE '1.2.%'
ORDER BY path;

-- Get all ancestors
SELECT * FROM categories_path
WHERE '1.2.3.4' LIKE path || '%'
ORDER BY depth;
```

### Nested Set Model

```sql
-- Each node stores left and right values
CREATE TABLE categories_nested (
    id serial PRIMARY KEY,
    name text NOT NULL,
    lft integer NOT NULL,
    rgt integer NOT NULL
);

CREATE INDEX idx_nested_lft_rgt ON categories_nested(lft, rgt);

-- Get all descendants
SELECT * FROM categories_nested
WHERE lft BETWEEN 2 AND 11
ORDER BY lft;

-- Get ancestor count (depth)
SELECT node.name, COUNT(parent.name) - 1 AS depth
FROM categories_nested AS node
JOIN categories_nested AS parent
  ON node.lft BETWEEN parent.lft AND parent.rgt
GROUP BY node.name, node.lft
ORDER BY node.lft;
```

## Full-Text Search Pattern

```sql
-- Documents with search
CREATE TABLE documents (
    id bigserial PRIMARY KEY,
    title text NOT NULL,
    content text NOT NULL,
    tags text[],
    search_vector tsvector,
    created_at timestamptz DEFAULT now()
);

-- Update search vector
CREATE OR REPLACE FUNCTION update_search_vector()
RETURNS trigger AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.content, '')), 'B') ||
        setweight(to_tsvector('english', COALESCE(array_to_string(NEW.tags, ' '), '')), 'C');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_documents_search
BEFORE INSERT OR UPDATE ON documents
FOR EACH ROW EXECUTE FUNCTION update_search_vector();

-- GIN index for search
CREATE INDEX idx_documents_search ON documents USING gin(search_vector);

-- Search with ranking
WITH search_results AS (
    SELECT
        id,
        title,
        ts_rank(search_vector, query) AS rank,
        ts_headline('english', content, query,
            'StartSel=<mark>, StopSel=</mark>, MaxWords=20, MinWords=10'
        ) AS excerpt
    FROM documents,
         plainto_tsquery('english', 'postgresql performance') AS query
    WHERE search_vector @@ query
)
SELECT * FROM search_results
ORDER BY rank DESC
LIMIT 10;
```

## Caching Pattern

```sql
-- Cache table with TTL
CREATE TABLE cache (
    key text PRIMARY KEY,
    value jsonb NOT NULL,
    expires_at timestamptz NOT NULL,
    created_at timestamptz DEFAULT now()
);

CREATE INDEX idx_cache_expires ON cache(expires_at);

-- Get or set cache
CREATE OR REPLACE FUNCTION cache_get_or_set(
    p_key text,
    p_ttl interval,
    p_compute_func text
) RETURNS jsonb AS $$
DECLARE
    cached_value jsonb;
    computed_value jsonb;
BEGIN
    -- Try to get from cache
    SELECT value INTO cached_value
    FROM cache
    WHERE key = p_key
      AND expires_at > now();

    IF FOUND THEN
        RETURN cached_value;
    END IF;

    -- Compute value
    EXECUTE format('SELECT %s()', p_compute_func) INTO computed_value;

    -- Store in cache
    INSERT INTO cache (key, value, expires_at)
    VALUES (p_key, computed_value, now() + p_ttl)
    ON CONFLICT (key) DO UPDATE
    SET value = EXCLUDED.value,
        expires_at = EXCLUDED.expires_at,
        created_at = now();

    RETURN computed_value;
END;
$$ LANGUAGE plpgsql;

-- Auto-cleanup expired entries
CREATE OR REPLACE FUNCTION cleanup_cache()
RETURNS void AS $$
BEGIN
    DELETE FROM cache WHERE expires_at < now();
END;
$$ LANGUAGE plpgsql;

-- Schedule cleanup
SELECT cron.schedule('cleanup-cache', '*/10 * * * *', 'SELECT cleanup_cache()');
```

## Soft Delete Pattern

```sql
-- Never really delete
ALTER TABLE users ADD COLUMN deleted_at timestamptz;

-- View for active records
CREATE VIEW active_users AS
SELECT * FROM users WHERE deleted_at IS NULL;

-- Soft delete function
CREATE OR REPLACE FUNCTION soft_delete(
    p_table text,
    p_id bigint
) RETURNS void AS $$
BEGIN
    EXECUTE format('
        UPDATE %I SET deleted_at = now() WHERE id = $1',
        p_table
    ) USING p_id;
END;
$$ LANGUAGE plpgsql;

-- Undelete function
CREATE OR REPLACE FUNCTION undelete(
    p_table text,
    p_id bigint
) RETURNS void AS $$
BEGIN
    EXECUTE format('
        UPDATE %I SET deleted_at = NULL WHERE id = $1',
        p_table
    ) USING p_id;
END;
$$ LANGUAGE plpgsql;

-- Permanent delete after 30 days
CREATE OR REPLACE FUNCTION purge_deleted()
RETURNS void AS $$
BEGIN
    DELETE FROM users
    WHERE deleted_at < now() - interval '30 days';
END;
$$ LANGUAGE plpgsql;
```

## API Rate Limiting Pattern

```sql
-- Rate limit tracking
CREATE TABLE rate_limits (
    key text NOT NULL,
    window_start timestamptz NOT NULL,
    request_count integer NOT NULL DEFAULT 1,
    PRIMARY KEY (key, window_start)
);

-- Check and increment rate limit
CREATE OR REPLACE FUNCTION check_rate_limit(
    p_key text,
    p_limit integer,
    p_window interval
) RETURNS boolean AS $$
DECLARE
    current_window timestamptz;
    current_count integer;
BEGIN
    current_window := date_trunc('minute', now());

    INSERT INTO rate_limits (key, window_start, request_count)
    VALUES (p_key, current_window, 1)
    ON CONFLICT (key, window_start) DO UPDATE
    SET request_count = rate_limits.request_count + 1
    RETURNING request_count INTO current_count;

    -- Cleanup old windows
    DELETE FROM rate_limits
    WHERE window_start < now() - p_window * 2;

    RETURN current_count <= p_limit;
END;
$$ LANGUAGE plpgsql;

-- Use in API
CREATE OR REPLACE FUNCTION api_endpoint(
    p_api_key text,
    p_data jsonb
) RETURNS jsonb AS $$
BEGIN
    -- Check rate limit: 100 requests per minute
    IF NOT check_rate_limit('api:' || p_api_key, 100, '1 minute'::interval) THEN
        RAISE EXCEPTION 'Rate limit exceeded' USING ERRCODE = 'P0001';
    END IF;

    -- Process API request
    RETURN jsonb_build_object('status', 'success', 'data', p_data);
END;
$$ LANGUAGE plpgsql;
```

## Exercises: Production Patterns

1. **Multi-Tenant System**: Implement a complete multi-tenant SaaS backend
2. **Event Store**: Build an event-sourced ordering system
3. **Job Queue**: Create a robust background job processor
4. **Audit System**: Implement compliance-grade audit logging
5. **Search Engine**: Build a full-text search with facets and filters

## Reality Check: Patterns in Production

Every pattern has trade-offs:
- **Complexity vs Simplicity**: More patterns = more complexity
- **Performance vs Flexibility**: Denormalization speeds reads but complicates writes
- **Storage vs Speed**: Caching and materialized views trade space for time
- **Consistency vs Availability**: Choose your CAP theorem position wisely

The best pattern is the simplest one that solves your actual problem. Don't implement event sourcing because it's cool; implement it because you need an audit trail. Don't use CQRS because Martin Fowler said so; use it because your read and write patterns genuinely differ.

## Summary

We've covered the battle-tested patterns that power real production systems:
- Multi-tenancy strategies from simple to complex
- Event sourcing and CQRS for complex domains
- Time-series optimization for IoT and metrics
- Queue patterns for background processing
- Audit logging for compliance and debugging
- Hierarchical data handling
- Full-text search implementation
- Caching strategies
- Rate limiting for APIs

These patterns aren't academic exercises—they're solutions to real problems you'll face when your application grows beyond the prototype stage.

Next up: Chapter 12 - Cloud and Modern Deployments, where we'll take PostgreSQL to the cloud and beyond.