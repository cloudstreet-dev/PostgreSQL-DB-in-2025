# Appendix A: Solutions to Exercises

*By Claude Code Opus 4.1*

Here are comprehensive solutions to the exercises from each chapter. Remember: there's often more than one correct approach in PostgreSQL.

## Chapter 1: First Principles Solutions

### Exercise 1: Design a Library Database

```sql
-- Complete library database schema
CREATE TABLE authors (
    id serial PRIMARY KEY,
    name text NOT NULL,
    birth_date date,
    biography text,
    created_at timestamptz DEFAULT now()
);

CREATE TABLE publishers (
    id serial PRIMARY KEY,
    name text NOT NULL,
    founded_year integer,
    country text,
    created_at timestamptz DEFAULT now()
);

CREATE TABLE books (
    id serial PRIMARY KEY,
    isbn text UNIQUE NOT NULL,
    title text NOT NULL,
    publisher_id integer REFERENCES publishers(id),
    publication_date date,
    pages integer CHECK (pages > 0),
    language text DEFAULT 'English',
    created_at timestamptz DEFAULT now()
);

CREATE TABLE book_authors (
    book_id integer REFERENCES books(id) ON DELETE CASCADE,
    author_id integer REFERENCES authors(id) ON DELETE CASCADE,
    author_order integer DEFAULT 1,
    PRIMARY KEY (book_id, author_id)
);

CREATE TABLE members (
    id serial PRIMARY KEY,
    member_number text UNIQUE NOT NULL,
    name text NOT NULL,
    email text UNIQUE NOT NULL,
    phone text,
    address text,
    joined_date date DEFAULT CURRENT_DATE,
    membership_expires date,
    created_at timestamptz DEFAULT now()
);

CREATE TABLE loans (
    id serial PRIMARY KEY,
    book_id integer REFERENCES books(id),
    member_id integer REFERENCES members(id),
    loan_date date DEFAULT CURRENT_DATE,
    due_date date NOT NULL,
    return_date date,
    fine_amount decimal(10,2) DEFAULT 0,
    created_at timestamptz DEFAULT now(),
    CONSTRAINT valid_dates CHECK (due_date > loan_date),
    CONSTRAINT valid_return CHECK (return_date IS NULL OR return_date >= loan_date)
);

-- Indexes for performance
CREATE INDEX idx_books_title ON books(title);
CREATE INDEX idx_loans_member ON loans(member_id) WHERE return_date IS NULL;
CREATE INDEX idx_loans_due ON loans(due_date) WHERE return_date IS NULL;
```

### Exercise 2: ACID Violations

```sql
-- Scenario 1: Non-atomic transfer (ACID violation)
-- WRONG - Can fail halfway through
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- System crashes here!
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- CORRECT - Atomic transaction
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- Scenario 2: Consistency violation
-- WRONG - Can create orphaned records
DELETE FROM authors WHERE id = 1;
-- Books still reference deleted author!

-- CORRECT - Use CASCADE or handle in transaction
BEGIN;
DELETE FROM book_authors WHERE author_id = 1;
DELETE FROM authors WHERE id = 1;
COMMIT;

-- Scenario 3: Isolation violation
-- WRONG - Dirty read in READ UNCOMMITTED (if it existed in PostgreSQL)
-- Session 1
BEGIN;
UPDATE products SET price = 99.99 WHERE id = 1;
-- Session 2 could read uncommitted price
ROLLBACK; -- Price change never happened!

-- CORRECT - PostgreSQL's default READ COMMITTED prevents this
```

### Exercise 3: When NOT to Use a Database

Text file solution for appropriate use cases:
```python
# Use case: Application logs
# Good with files - sequential writes, rarely queried
with open('app.log', 'a') as f:
    f.write(f"{datetime.now()}: User logged in\n")

# Use case: Configuration
# Good with files - rarely changes, version controlled
config = yaml.load(open('config.yml'))

# Use case: Large binary data
# Good with filesystem + metadata in DB
video_path = f"/storage/videos/{video_id}.mp4"
# Store path in database, file on disk

# When TO use a database:
# - Concurrent updates
# - Complex queries
# - Relationships between entities
# - ACID requirements
```

## Chapter 2: Why PostgreSQL Solutions

### Exercise: Database Selection Matrix

```python
def select_database(requirements):
    """
    Select appropriate database based on requirements
    """
    score = {
        'postgresql': 0,
        'mysql': 0,
        'mongodb': 0,
        'redis': 0,
        'sqlite': 0
    }

    # Complex queries with JOINs
    if requirements.get('complex_queries'):
        score['postgresql'] += 3
        score['mysql'] += 2
        score['sqlite'] += 2

    # ACID compliance
    if requirements.get('acid'):
        score['postgresql'] += 3
        score['mysql'] += 2
        score['sqlite'] += 3
        score['mongodb'] -= 1

    # Horizontal scaling
    if requirements.get('horizontal_scaling'):
        score['mongodb'] += 3
        score['redis'] += 2
        score['postgresql'] -= 1
        score['sqlite'] -= 3

    # JSON/Document storage
    if requirements.get('json_data'):
        score['mongodb'] += 3
        score['postgresql'] += 2  # JSONB support

    # Caching layer
    if requirements.get('caching'):
        score['redis'] += 5

    # Embedded/Mobile
    if requirements.get('embedded'):
        score['sqlite'] += 5
        score['postgresql'] -= 3

    # High write throughput
    if requirements.get('high_writes'):
        score['mongodb'] += 2
        score['redis'] += 3
        score['postgresql'] += 1

    # Advanced features (CTEs, Window Functions)
    if requirements.get('advanced_sql'):
        score['postgresql'] += 3
        score['sqlite'] += 1

    return max(score, key=score.get)

# Test scenarios
ecommerce = {
    'complex_queries': True,
    'acid': True,
    'json_data': True,
    'advanced_sql': True
}
print(f"E-commerce: {select_database(ecommerce)}")  # postgresql

iot_platform = {
    'high_writes': True,
    'horizontal_scaling': True,
    'json_data': True
}
print(f"IoT Platform: {select_database(iot_platform)}")  # mongodb

mobile_app = {
    'embedded': True,
    'acid': True
}
print(f"Mobile App: {select_database(mobile_app)}")  # sqlite
```

## Chapter 4: SQL Fundamentals Solutions

### Exercise: E-commerce Queries

```sql
-- 1. Find top 5 customers by total spending
SELECT
    c.id,
    c.name,
    c.email,
    SUM(o.total_amount) as total_spent,
    COUNT(DISTINCT o.id) as order_count
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE o.status = 'completed'
GROUP BY c.id, c.name, c.email
ORDER BY total_spent DESC
LIMIT 5;

-- 2. Products never ordered
SELECT p.*
FROM products p
LEFT JOIN order_items oi ON p.id = oi.product_id
WHERE oi.id IS NULL
ORDER BY p.created_at;

-- 3. Monthly revenue trend
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', created_at) as month,
        SUM(total_amount) as revenue,
        COUNT(*) as order_count
    FROM orders
    WHERE status = 'completed'
    GROUP BY DATE_TRUNC('month', created_at)
)
SELECT
    month,
    revenue,
    order_count,
    revenue / order_count as avg_order_value,
    LAG(revenue) OVER (ORDER BY month) as prev_month_revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) as revenue_change,
    ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month)) /
          NULLIF(LAG(revenue) OVER (ORDER BY month), 0), 2) as percent_change
FROM monthly_revenue
ORDER BY month;

-- 4. Customer cohort analysis
WITH customer_cohorts AS (
    SELECT
        c.id,
        DATE_TRUNC('month', c.created_at) as cohort_month,
        DATE_TRUNC('month', o.created_at) as order_month
    FROM customers c
    JOIN orders o ON c.id = o.customer_id
    WHERE o.status = 'completed'
)
SELECT
    cohort_month,
    COUNT(DISTINCT CASE WHEN order_month = cohort_month THEN id END) as month_0,
    COUNT(DISTINCT CASE WHEN order_month = cohort_month + interval '1 month' THEN id END) as month_1,
    COUNT(DISTINCT CASE WHEN order_month = cohort_month + interval '2 months' THEN id END) as month_2,
    COUNT(DISTINCT CASE WHEN order_month = cohort_month + interval '3 months' THEN id END) as month_3
FROM customer_cohorts
GROUP BY cohort_month
ORDER BY cohort_month;

-- 5. Product recommendation engine
WITH product_pairs AS (
    SELECT
        oi1.product_id as product1,
        oi2.product_id as product2,
        COUNT(*) as times_bought_together
    FROM order_items oi1
    JOIN order_items oi2 ON oi1.order_id = oi2.order_id
        AND oi1.product_id < oi2.product_id
    GROUP BY oi1.product_id, oi2.product_id
    HAVING COUNT(*) > 5
)
SELECT
    p1.name as product1_name,
    p2.name as product2_name,
    pp.times_bought_together
FROM product_pairs pp
JOIN products p1 ON pp.product1 = p1.id
JOIN products p2 ON pp.product2 = p2.id
ORDER BY pp.times_bought_together DESC
LIMIT 10;
```

## Chapter 5: Transactions Solutions

### Exercise: Bank Transfer System

```sql
-- Complete bank transfer implementation
CREATE OR REPLACE FUNCTION transfer_money(
    p_from_account text,
    p_to_account text,
    p_amount decimal,
    p_description text DEFAULT NULL
) RETURNS jsonb AS $$
DECLARE
    v_from_balance decimal;
    v_to_balance decimal;
    v_transaction_id bigint;
    v_result jsonb;
BEGIN
    -- Set lock timeout to prevent long waits
    SET LOCAL lock_timeout = '5s';

    -- Lock accounts in consistent order to prevent deadlock
    IF p_from_account < p_to_account THEN
        PERFORM * FROM accounts WHERE account_number = p_from_account FOR UPDATE;
        PERFORM * FROM accounts WHERE account_number = p_to_account FOR UPDATE;
    ELSE
        PERFORM * FROM accounts WHERE account_number = p_to_account FOR UPDATE;
        PERFORM * FROM accounts WHERE account_number = p_from_account FOR UPDATE;
    END IF;

    -- Check from account exists and has sufficient funds
    SELECT balance INTO v_from_balance
    FROM accounts
    WHERE account_number = p_from_account;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Account % not found', p_from_account;
    END IF;

    IF v_from_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient funds. Balance: %, Requested: %',
            v_from_balance, p_amount;
    END IF;

    -- Check to account exists
    SELECT balance INTO v_to_balance
    FROM accounts
    WHERE account_number = p_to_account;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Account % not found', p_to_account;
    END IF;

    -- Perform transfer
    UPDATE accounts
    SET balance = balance - p_amount,
        updated_at = now()
    WHERE account_number = p_from_account;

    UPDATE accounts
    SET balance = balance + p_amount,
        updated_at = now()
    WHERE account_number = p_to_account;

    -- Record transaction
    INSERT INTO transactions (
        from_account,
        to_account,
        amount,
        description,
        status
    ) VALUES (
        p_from_account,
        p_to_account,
        p_amount,
        p_description,
        'completed'
    ) RETURNING id INTO v_transaction_id;

    -- Prepare result
    v_result := jsonb_build_object(
        'success', true,
        'transaction_id', v_transaction_id,
        'from_account', p_from_account,
        'to_account', p_to_account,
        'amount', p_amount,
        'timestamp', now()
    );

    RETURN v_result;

EXCEPTION
    WHEN lock_timeout THEN
        RAISE EXCEPTION 'Transaction timeout - please try again';
    WHEN OTHERS THEN
        -- Log error
        INSERT INTO error_log (error_message, error_detail)
        VALUES (SQLERRM, SQLSTATE);

        -- Re-raise
        RAISE;
END;
$$ LANGUAGE plpgsql;

-- Test concurrent transfers
-- Session 1
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT transfer_money('ACC001', 'ACC002', 100.00);
-- Session 2 (concurrent)
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT transfer_money('ACC002', 'ACC001', 50.00);
-- Both should succeed without deadlock
```

## Chapter 6: Advanced Features Solutions

### Exercise: JSONB Document Store

```sql
-- Complete document store implementation
CREATE TABLE documents (
    id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
    type text NOT NULL,
    data jsonb NOT NULL,
    tags text[],
    created_at timestamptz DEFAULT now(),
    updated_at timestamptz DEFAULT now()
);

-- Indexes for different access patterns
CREATE INDEX idx_documents_type ON documents(type);
CREATE INDEX idx_documents_data_gin ON documents USING gin(data);
CREATE INDEX idx_documents_tags ON documents USING gin(tags);
CREATE INDEX idx_documents_created ON documents(created_at DESC);

-- Extract specific fields for frequent queries
CREATE INDEX idx_documents_email ON documents((data->>'email'))
    WHERE type = 'user';

-- Insert documents
INSERT INTO documents (type, data, tags) VALUES
('user', '{
    "name": "John Doe",
    "email": "john@example.com",
    "age": 30,
    "interests": ["postgresql", "hiking"],
    "address": {
        "city": "Seattle",
        "state": "WA"
    }
}'::jsonb, ARRAY['active', 'premium']),
('product', '{
    "name": "PostgreSQL Guide",
    "price": 49.99,
    "categories": ["database", "education"],
    "metadata": {
        "author": "Claude",
        "pages": 500
    }
}'::jsonb, ARRAY['book', 'technical']);

-- Complex queries
-- Find users in Seattle interested in PostgreSQL
SELECT
    id,
    data->>'name' as name,
    data->>'email' as email
FROM documents
WHERE type = 'user'
    AND data->'address'->>'city' = 'Seattle'
    AND data->'interests' ? 'postgresql';

-- Aggregate product prices by category
SELECT
    jsonb_array_elements_text(data->'categories') as category,
    COUNT(*) as product_count,
    AVG((data->>'price')::numeric) as avg_price
FROM documents
WHERE type = 'product'
GROUP BY category
ORDER BY product_count DESC;

-- Update nested field
UPDATE documents
SET data = jsonb_set(
    data,
    '{address,zip}',
    '"98101"'
)
WHERE type = 'user'
    AND data->'address'->>'city' = 'Seattle';

-- Full-text search on JSONB
ALTER TABLE documents ADD COLUMN search_vector tsvector;

UPDATE documents
SET search_vector = to_tsvector('english', data::text);

CREATE INDEX idx_documents_fts ON documents USING gin(search_vector);

SELECT
    id,
    data->>'name' as name,
    ts_rank(search_vector, query) as rank
FROM documents,
    plainto_tsquery('english', 'postgresql database') as query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

## Chapter 7: Performance Solutions

### Exercise: Slow Query Optimization

```sql
-- Original slow query
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    c.name,
    COUNT(o.id) as order_count,
    SUM(o.total_amount) as total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.created_at > NOW() - INTERVAL '1 year'
GROUP BY c.id
ORDER BY total_spent DESC;

-- Problem: LEFT JOIN with WHERE on right table acts like INNER JOIN
-- Also missing useful indexes

-- Solution 1: Fix the JOIN logic
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    c.name,
    COUNT(o.id) as order_count,
    COALESCE(SUM(o.total_amount), 0) as total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
    AND o.created_at > NOW() - INTERVAL '1 year'  -- Move to JOIN condition
GROUP BY c.id, c.name
ORDER BY total_spent DESC;

-- Solution 2: Add appropriate indexes
CREATE INDEX idx_orders_customer_created
    ON orders(customer_id, created_at DESC)
    INCLUDE (total_amount)  -- Covering index
    WHERE created_at > NOW() - INTERVAL '2 years';  -- Partial index

-- Solution 3: Use materialized view for frequent queries
CREATE MATERIALIZED VIEW customer_order_summary AS
SELECT
    c.id as customer_id,
    c.name,
    COUNT(o.id) as order_count_year,
    COALESCE(SUM(o.total_amount), 0) as total_spent_year,
    MAX(o.created_at) as last_order_date
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
    AND o.created_at > NOW() - INTERVAL '1 year'
GROUP BY c.id, c.name;

CREATE UNIQUE INDEX ON customer_order_summary(customer_id);
CREATE INDEX ON customer_order_summary(total_spent_year DESC);

-- Refresh periodically
CREATE OR REPLACE FUNCTION refresh_customer_summary()
RETURNS void AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY customer_order_summary;
END;
$$ LANGUAGE plpgsql;

-- Schedule refresh
SELECT cron.schedule('refresh-customer-summary', '0 * * * *',
    'SELECT refresh_customer_summary()');
```

## Chapter 8: Security Solutions

### Exercise: Multi-tenant Security

```sql
-- Complete multi-tenant security implementation
-- 1. Schema setup
CREATE SCHEMA IF NOT EXISTS tenant_shared;

-- 2. Tenant table
CREATE TABLE tenant_shared.tenants (
    id serial PRIMARY KEY,
    name text UNIQUE NOT NULL,
    api_key uuid DEFAULT gen_random_uuid(),
    active boolean DEFAULT true,
    created_at timestamptz DEFAULT now()
);

-- 3. User table with tenant association
CREATE TABLE tenant_shared.users (
    id serial PRIMARY KEY,
    tenant_id integer NOT NULL REFERENCES tenant_shared.tenants(id),
    email text NOT NULL,
    role text NOT NULL CHECK (role IN ('admin', 'user', 'viewer')),
    created_at timestamptz DEFAULT now(),
    UNIQUE(tenant_id, email)
);

-- 4. Enable RLS
ALTER TABLE tenant_shared.users ENABLE ROW LEVEL SECURITY;
ALTER TABLE tenant_shared.tenants ENABLE ROW LEVEL SECURITY;

-- 5. Create roles
CREATE ROLE tenant_admin;
CREATE ROLE tenant_user;
CREATE ROLE tenant_viewer;

-- 6. RLS Policies
-- Tenant isolation policy
CREATE POLICY tenant_isolation ON tenant_shared.users
    FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::integer);

-- Admin can see all in their tenant
CREATE POLICY admin_all ON tenant_shared.users
    FOR ALL
    TO tenant_admin
    USING (
        tenant_id = current_setting('app.tenant_id')::integer
    );

-- Users can see and update themselves
CREATE POLICY user_self ON tenant_shared.users
    FOR ALL
    TO tenant_user
    USING (
        tenant_id = current_setting('app.tenant_id')::integer
        AND email = current_setting('app.user_email')
    );

-- Viewers can only SELECT
CREATE POLICY viewer_read ON tenant_shared.users
    FOR SELECT
    TO tenant_viewer
    USING (tenant_id = current_setting('app.tenant_id')::integer);

-- 7. Secure session function
CREATE OR REPLACE FUNCTION establish_tenant_session(
    p_api_key uuid,
    p_user_email text
) RETURNS jsonb
SECURITY DEFINER
AS $$
DECLARE
    v_tenant_id integer;
    v_user_role text;
    v_session_token uuid;
BEGIN
    -- Validate API key
    SELECT id INTO v_tenant_id
    FROM tenant_shared.tenants
    WHERE api_key = p_api_key AND active = true;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Invalid API key';
    END IF;

    -- Get user role
    SELECT role INTO v_user_role
    FROM tenant_shared.users
    WHERE tenant_id = v_tenant_id
        AND email = p_user_email;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'User not found';
    END IF;

    -- Generate session token
    v_session_token := gen_random_uuid();

    -- Store session (implement session table)
    INSERT INTO tenant_shared.sessions (
        token,
        tenant_id,
        user_email,
        user_role,
        expires_at
    ) VALUES (
        v_session_token,
        v_tenant_id,
        p_user_email,
        v_user_role,
        now() + interval '1 hour'
    );

    -- Set session variables
    PERFORM set_config('app.tenant_id', v_tenant_id::text, false);
    PERFORM set_config('app.user_email', p_user_email, false);
    PERFORM set_config('app.user_role', v_user_role, false);

    -- Grant appropriate role
    EXECUTE format('SET ROLE tenant_%s', v_user_role);

    RETURN jsonb_build_object(
        'session_token', v_session_token,
        'tenant_id', v_tenant_id,
        'user_role', v_user_role,
        'expires_at', now() + interval '1 hour'
    );
END;
$$ LANGUAGE plpgsql;

-- 8. Test the implementation
-- Create test data
INSERT INTO tenant_shared.tenants (name) VALUES ('ACME Corp'), ('Tech Inc');
INSERT INTO tenant_shared.users (tenant_id, email, role) VALUES
    (1, 'admin@acme.com', 'admin'),
    (1, 'user@acme.com', 'user'),
    (2, 'admin@tech.com', 'admin');

-- Test session
SELECT establish_tenant_session(
    (SELECT api_key FROM tenant_shared.tenants WHERE name = 'ACME Corp'),
    'admin@acme.com'
);
```

## Chapter 9: High Availability Solutions

### Exercise: HA Cluster Setup

```bash
#!/bin/bash
# Complete HA cluster setup script

# 1. Primary setup
setup_primary() {
    # postgresql.conf
    cat >> /etc/postgresql/16/main/postgresql.conf <<EOF
# Replication Settings
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB
hot_standby = on
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
synchronous_commit = on
synchronous_standby_names = 'FIRST 2 (standby1, standby2)'
EOF

    # pg_hba.conf
    echo "host replication replicator 192.168.1.0/24 scram-sha-256" >> /etc/postgresql/16/main/pg_hba.conf

    # Create replication user
    sudo -u postgres psql <<EOF
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'StrongPassword123!';
SELECT pg_reload_conf();
EOF
}

# 2. Standby setup
setup_standby() {
    local primary_host=$1
    local standby_name=$2

    # Stop PostgreSQL
    systemctl stop postgresql

    # Clear data directory
    rm -rf /var/lib/postgresql/16/main/*

    # Base backup
    sudo -u postgres pg_basebackup \
        -h $primary_host \
        -D /var/lib/postgresql/16/main \
        -U replicator \
        -P -v -R -X stream \
        -C -S $standby_name \
        -W

    # Start PostgreSQL
    systemctl start postgresql
}

# 3. Monitoring setup
setup_monitoring() {
    sudo -u postgres psql <<'EOF'
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

CREATE OR REPLACE FUNCTION check_replication_status()
RETURNS TABLE(
    replica text,
    state text,
    lag_bytes bigint,
    lag_seconds numeric
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        application_name,
        state,
        pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn),
        EXTRACT(EPOCH FROM (now() - reply_time))
    FROM pg_stat_replication
    ORDER BY application_name;
END;
$$ LANGUAGE plpgsql;

-- Alert if lag exceeds threshold
CREATE OR REPLACE FUNCTION alert_high_lag()
RETURNS void AS $$
DECLARE
    rec record;
BEGIN
    FOR rec IN SELECT * FROM check_replication_status() WHERE lag_seconds > 10 LOOP
        RAISE WARNING 'High replication lag on %: % seconds',
            rec.replica, rec.lag_seconds;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
EOF

    # Cron job for monitoring
    echo "*/1 * * * * postgres psql -c 'SELECT alert_high_lag()'" | crontab -
}

# 4. Failover procedure
perform_failover() {
    local new_primary=$1

    echo "Promoting $new_primary to primary..."

    # On the standby to be promoted
    ssh $new_primary "sudo -u postgres pg_ctl promote -D /var/lib/postgresql/16/main"

    # Update application connection string
    update_app_config $new_primary

    # Reconfigure old primary as standby (after fixing issues)
    # ...
}

# 5. Test the setup
test_ha_setup() {
    echo "Testing replication..."

    # Create test table on primary
    sudo -u postgres psql <<EOF
CREATE TABLE ha_test (id serial PRIMARY KEY, data text, created_at timestamptz DEFAULT now());
INSERT INTO ha_test (data) VALUES ('Test replication');
EOF

    sleep 2

    # Check on standby
    ssh standby1 "sudo -u postgres psql -c 'SELECT * FROM ha_test'"

    # Test failover
    echo "Testing failover..."
    perform_failover standby1
}

# Main execution
case "$1" in
    primary)
        setup_primary
        ;;
    standby)
        setup_standby $2 $3
        ;;
    monitor)
        setup_monitoring
        ;;
    failover)
        perform_failover $2
        ;;
    test)
        test_ha_setup
        ;;
    *)
        echo "Usage: $0 {primary|standby|monitor|failover|test}"
        exit 1
esac
```

## Chapter 10: Testing and Migrations Solutions

### Exercise: Migration Test Suite

```sql
-- Complete migration testing framework
CREATE SCHEMA IF NOT EXISTS migration_test;

-- Test data generator
CREATE OR REPLACE FUNCTION migration_test.generate_test_data(
    p_rows integer DEFAULT 1000
) RETURNS void AS $$
BEGIN
    -- Generate customers
    INSERT INTO customers (name, email)
    SELECT
        'Customer ' || i,
        'customer' || i || '@test.com'
    FROM generate_series(1, p_rows) i
    ON CONFLICT (email) DO NOTHING;

    -- Generate products
    INSERT INTO products (name, price)
    SELECT
        'Product ' || i,
        (random() * 1000)::decimal(10,2)
    FROM generate_series(1, p_rows / 10) i;

    -- Generate orders
    INSERT INTO orders (customer_id, total_amount, status)
    SELECT
        (random() * p_rows + 1)::integer,
        (random() * 1000)::decimal(10,2),
        CASE
            WHEN random() < 0.8 THEN 'completed'
            WHEN random() < 0.9 THEN 'pending'
            ELSE 'cancelled'
        END
    FROM generate_series(1, p_rows * 2) i;
END;
$$ LANGUAGE plpgsql;

-- Migration test harness
CREATE OR REPLACE FUNCTION migration_test.test_migration(
    p_migration_file text
) RETURNS jsonb AS $$
DECLARE
    v_start_time timestamp;
    v_end_time timestamp;
    v_test_results jsonb = '{}';
    v_row_counts jsonb;
    v_performance jsonb;
BEGIN
    -- Capture initial state
    SELECT jsonb_object_agg(tablename, row_count)
    INTO v_row_counts
    FROM (
        SELECT
            tablename,
            (xpath('/row/cnt/text()',
                xml_query))[1]::text::bigint as row_count
        FROM (
            SELECT
                tablename,
                query_to_xml(format('SELECT COUNT(*) AS cnt FROM %I', tablename), false, true, '') AS xml_query
            FROM pg_tables
            WHERE schemaname = 'public'
        ) t
    ) counts;

    v_test_results = v_test_results || jsonb_build_object('initial_counts', v_row_counts);

    -- Run migration
    v_start_time = clock_timestamp();

    BEGIN
        EXECUTE pg_read_file(p_migration_file);
        v_test_results = v_test_results || jsonb_build_object('migration_status', 'success');
    EXCEPTION WHEN OTHERS THEN
        v_test_results = v_test_results || jsonb_build_object(
            'migration_status', 'failed',
            'error', SQLERRM
        );
        RETURN v_test_results;
    END;

    v_end_time = clock_timestamp();

    -- Capture performance metrics
    v_performance = jsonb_build_object(
        'execution_time_ms', EXTRACT(MILLISECONDS FROM (v_end_time - v_start_time)),
        'locks_acquired', (SELECT COUNT(*) FROM pg_locks WHERE pid = pg_backend_pid())
    );

    v_test_results = v_test_results || jsonb_build_object('performance', v_performance);

    -- Verify data integrity
    PERFORM migration_test.verify_constraints();
    PERFORM migration_test.verify_indexes();

    RETURN v_test_results;
END;
$$ LANGUAGE plpgsql;

-- Constraint verification
CREATE OR REPLACE FUNCTION migration_test.verify_constraints()
RETURNS void AS $$
DECLARE
    v_constraint record;
    v_violations integer;
BEGIN
    FOR v_constraint IN
        SELECT
            conname,
            conrelid::regclass AS table_name,
            pg_get_constraintdef(oid) AS definition
        FROM pg_constraint
        WHERE connamespace = 'public'::regnamespace
    LOOP
        -- Test each constraint
        EXECUTE format('
            SELECT COUNT(*) FROM %s WHERE NOT (%s)',
            v_constraint.table_name,
            v_constraint.definition
        ) INTO v_violations;

        IF v_violations > 0 THEN
            RAISE WARNING 'Constraint % violated: % rows',
                v_constraint.conname, v_violations;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Performance regression test
CREATE OR REPLACE FUNCTION migration_test.test_query_performance(
    p_query text,
    p_expected_ms numeric
) RETURNS boolean AS $$
DECLARE
    v_start timestamp;
    v_end timestamp;
    v_actual_ms numeric;
BEGIN
    v_start = clock_timestamp();
    EXECUTE p_query;
    v_end = clock_timestamp();

    v_actual_ms = EXTRACT(MILLISECONDS FROM (v_end - v_start));

    IF v_actual_ms > p_expected_ms * 1.2 THEN  -- 20% tolerance
        RAISE WARNING 'Performance regression: Query took % ms (expected < % ms)',
            v_actual_ms, p_expected_ms;
        RETURN false;
    END IF;

    RETURN true;
END;
$$ LANGUAGE plpgsql;
```

## Chapter 11: Real-World Patterns Solutions

### Exercise: Event Sourcing System

```sql
-- Complete event sourcing implementation
CREATE SCHEMA IF NOT EXISTS event_store;

-- Event storage
CREATE TABLE event_store.events (
    id bigserial PRIMARY KEY,
    stream_id uuid NOT NULL,
    stream_type text NOT NULL,
    event_type text NOT NULL,
    event_version integer NOT NULL,
    event_data jsonb NOT NULL,
    metadata jsonb DEFAULT '{}',
    created_at timestamptz DEFAULT now(),
    created_by text DEFAULT current_user,
    UNIQUE(stream_id, event_version)
);

CREATE INDEX idx_events_stream ON event_store.events(stream_id, event_version);
CREATE INDEX idx_events_type ON event_store.events(stream_type, created_at);

-- Event bus for subscriptions
CREATE TABLE event_store.subscriptions (
    id serial PRIMARY KEY,
    name text UNIQUE NOT NULL,
    event_types text[],
    last_processed_id bigint DEFAULT 0,
    active boolean DEFAULT true
);

-- Append event with optimistic locking
CREATE OR REPLACE FUNCTION event_store.append_event(
    p_stream_id uuid,
    p_stream_type text,
    p_event_type text,
    p_event_data jsonb,
    p_expected_version integer DEFAULT NULL
) RETURNS bigint AS $$
DECLARE
    v_current_version integer;
    v_event_id bigint;
BEGIN
    -- Get current version with lock
    SELECT COALESCE(MAX(event_version), 0)
    INTO v_current_version
    FROM event_store.events
    WHERE stream_id = p_stream_id
    FOR UPDATE;

    -- Check expected version (optimistic concurrency)
    IF p_expected_version IS NOT NULL AND
       p_expected_version != v_current_version THEN
        RAISE EXCEPTION 'Concurrency conflict: expected version %, current version %',
            p_expected_version, v_current_version;
    END IF;

    -- Insert event
    INSERT INTO event_store.events (
        stream_id,
        stream_type,
        event_type,
        event_version,
        event_data
    ) VALUES (
        p_stream_id,
        p_stream_type,
        p_event_type,
        v_current_version + 1,
        p_event_data
    ) RETURNING id INTO v_event_id;

    -- Notify subscribers
    PERFORM pg_notify(
        'event_created',
        jsonb_build_object(
            'event_id', v_event_id,
            'stream_id', p_stream_id,
            'event_type', p_event_type
        )::text
    );

    RETURN v_event_id;
END;
$$ LANGUAGE plpgsql;

-- Get stream events
CREATE OR REPLACE FUNCTION event_store.get_stream(
    p_stream_id uuid,
    p_from_version integer DEFAULT 0
) RETURNS TABLE(
    event_version integer,
    event_type text,
    event_data jsonb,
    created_at timestamptz
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        e.event_version,
        e.event_type,
        e.event_data,
        e.created_at
    FROM event_store.events e
    WHERE e.stream_id = p_stream_id
        AND e.event_version > p_from_version
    ORDER BY e.event_version;
END;
$$ LANGUAGE plpgsql;

-- Projection builder
CREATE OR REPLACE FUNCTION event_store.build_projection(
    p_stream_id uuid
) RETURNS jsonb AS $$
DECLARE
    v_state jsonb = '{}';
    v_event record;
BEGIN
    FOR v_event IN
        SELECT * FROM event_store.get_stream(p_stream_id)
    LOOP
        -- Apply event to state
        v_state = event_store.apply_event(
            v_state,
            v_event.event_type,
            v_event.event_data
        );
    END LOOP;

    RETURN v_state;
END;
$$ LANGUAGE plpgsql;

-- Event applier (reducer)
CREATE OR REPLACE FUNCTION event_store.apply_event(
    p_state jsonb,
    p_event_type text,
    p_event_data jsonb
) RETURNS jsonb AS $$
BEGIN
    CASE p_event_type
        WHEN 'AccountCreated' THEN
            RETURN p_state || jsonb_build_object(
                'account_id', p_event_data->>'account_id',
                'balance', 0,
                'status', 'active'
            );

        WHEN 'MoneyDeposited' THEN
            RETURN p_state || jsonb_build_object(
                'balance', COALESCE((p_state->>'balance')::numeric, 0) +
                          (p_event_data->>'amount')::numeric
            );

        WHEN 'MoneyWithdrawn' THEN
            RETURN p_state || jsonb_build_object(
                'balance', COALESCE((p_state->>'balance')::numeric, 0) -
                          (p_event_data->>'amount')::numeric
            );

        WHEN 'AccountClosed' THEN
            RETURN p_state || jsonb_build_object('status', 'closed');

        ELSE
            RETURN p_state;
    END CASE;
END;
$$ LANGUAGE plpgsql;

-- Example usage: Banking
DO $$
DECLARE
    v_account_id uuid = gen_random_uuid();
    v_event_id bigint;
BEGIN
    -- Create account
    v_event_id = event_store.append_event(
        v_account_id,
        'Account',
        'AccountCreated',
        jsonb_build_object(
            'account_id', v_account_id,
            'owner', 'John Doe'
        )
    );

    -- Deposit money
    v_event_id = event_store.append_event(
        v_account_id,
        'Account',
        'MoneyDeposited',
        jsonb_build_object('amount', 1000)
    );

    -- Withdraw money
    v_event_id = event_store.append_event(
        v_account_id,
        'Account',
        'MoneyWithdrawn',
        jsonb_build_object('amount', 250)
    );

    -- Get current state
    RAISE NOTICE 'Current state: %',
        event_store.build_projection(v_account_id);
END $$;
```

## Chapter 12: Cloud Solutions

### Exercise: Multi-Cloud Deployment

```yaml
# Terraform configuration for multi-cloud PostgreSQL
# main.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

module "aws_postgres" {
  source = "./modules/aws-postgres"

  environment     = var.environment
  instance_class  = "db.r6g.xlarge"
  allocated_storage = 100
  multi_az        = true
  backup_retention_period = 30

  enable_cross_region_replica = true
  replica_region = "us-west-2"
}

module "gcp_postgres" {
  source = "./modules/gcp-postgres"

  environment = var.environment
  tier        = "db-n1-standard-4"
  disk_size   = 100
  availability_type = "REGIONAL"

  enable_cross_region_replica = true
  replica_location = "us-west1"
}

module "azure_postgres" {
  source = "./modules/azure-postgres"

  environment = var.environment
  sku_name    = "GP_Standard_D4s_v3"
  storage_mb  = 102400

  high_availability_mode = "ZoneRedundant"
  geo_redundant_backup = true
}

# Cross-cloud data sync job
resource "kubernetes_cron_job" "cross_cloud_sync" {
  metadata {
    name = "postgres-cross-cloud-sync"
  }

  spec {
    schedule = "0 * * * *"

    job_template {
      metadata {}

      spec {
        template {
          metadata {}

          spec {
            container {
              name  = "sync"
              image = "postgres-sync:latest"

              env {
                name  = "SOURCE_DB"
                value = module.aws_postgres.connection_string
              }

              env {
                name  = "TARGETS"
                value = jsonencode([
                  module.gcp_postgres.connection_string,
                  module.azure_postgres.connection_string
                ])
              }
            }
          }
        }
      }
    }
  }
}

# Global load balancer
resource "cloudflare_load_balancer" "global_postgres" {
  zone_id = var.cloudflare_zone_id
  name    = "postgres.example.com"

  default_pool_ids = [
    cloudflare_load_balancer_pool.aws.id,
    cloudflare_load_balancer_pool.gcp.id,
    cloudflare_load_balancer_pool.azure.id
  ]

  fallback_pool_id = cloudflare_load_balancer_pool.aws.id

  steering_policy = "geo"

  pop_pools {
    pop      = "LAX"
    pool_ids = [cloudflare_load_balancer_pool.gcp.id]
  }

  pop_pools {
    pop      = "IAD"
    pool_ids = [cloudflare_load_balancer_pool.aws.id]
  }

  pop_pools {
    pop      = "LHR"
    pool_ids = [cloudflare_load_balancer_pool.azure.id]
  }

  session_affinity = "cookie"
  session_affinity_ttl = 5400
}
```

These solutions demonstrate production-ready implementations of the concepts from each chapter. Remember to adapt them to your specific requirements and always test thoroughly in a non-production environment first!