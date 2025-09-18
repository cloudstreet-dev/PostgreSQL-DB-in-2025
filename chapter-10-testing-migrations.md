# Chapter 10: Testing and Migrations

*By Claude Code Opus 4.1*

"It works on my machine" - Famous last words before production deployment. This chapter ensures your database changes work everywhere, every time, and can be rolled back when (not if) something goes wrong.

## The Database Change Challenge

Databases are stateful beasts. Unlike application code that you can simply redeploy, database changes accumulate. One bad migration can ruin your entire day (or career). Let's make sure that never happens.

## Schema Migration Fundamentals

### Why Migrations Matter

```sql
-- Without migrations: Chaos
-- Developer 1 on Monday
ALTER TABLE users ADD COLUMN age integer;

-- Developer 2 on Tuesday (doesn't know about age)
ALTER TABLE users ADD COLUMN birthdate date;

-- Production on Wednesday
-- ERROR: column "age" already exists (or doesn't exist)
-- Which version is correct? Nobody knows!
```

### Enter Migration Tools

Migrations bring order to chaos:
- **Version Control**: Every schema change is tracked
- **Repeatability**: Same changes applied everywhere
- **Rollback Capability**: Undo when things go wrong
- **Team Collaboration**: Everyone stays in sync

## Choosing a Migration Tool

### Popular Options

1. **Flyway**: Java-based, simple, widely supported
2. **Liquibase**: XML/YAML/JSON based, database agnostic
3. **Sqitch**: Git-like, dependency management
4. **golang-migrate**: Simple, CLI focused
5. **Alembic**: Python/SQLAlchemy integrated

Let's explore each approach.

## Flyway Migrations

### Setup

```bash
# Download Flyway
wget -qO- https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/9.22.0/flyway-commandline-9.22.0-linux-x64.tar.gz | tar xvz

# Configure flyway.conf
flyway.url=jdbc:postgresql://localhost:5432/mydb
flyway.user=dbuser
flyway.password=secret
flyway.schemas=public
flyway.locations=filesystem:./migrations
```

### Creating Migrations

Flyway uses naming conventions:

```sql
-- V1__Create_users_table.sql
CREATE TABLE users (
    id bigserial PRIMARY KEY,
    email text UNIQUE NOT NULL,
    created_at timestamptz DEFAULT now()
);

-- V2__Add_user_profiles.sql
CREATE TABLE user_profiles (
    user_id bigint PRIMARY KEY REFERENCES users(id),
    full_name text,
    bio text,
    avatar_url text
);

-- V3__Add_age_to_users.sql
ALTER TABLE users ADD COLUMN age integer CHECK (age >= 0);
```

### Running Migrations

```bash
# Check status
flyway info

# Run migrations
flyway migrate

# Validate migrations
flyway validate

# Clean (DANGER: Drops everything!)
flyway clean
```

## Sqitch: Git for Databases

Sqitch treats migrations like Git commits:

### Initialize Project

```bash
sqitch init myapp --engine pg
```

### Create Changes

```bash
# Add a change
sqitch add users -n "Add users table"
```

Edit `deploy/users.sql`:

```sql
-- Deploy myapp:users to pg

BEGIN;

CREATE TABLE users (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    email text UNIQUE NOT NULL,
    password_hash text NOT NULL,
    created_at timestamptz DEFAULT now()
);

COMMIT;
```

Edit `revert/users.sql`:

```sql
-- Revert myapp:users from pg

BEGIN;

DROP TABLE users;

COMMIT;
```

Edit `verify/users.sql`:

```sql
-- Verify myapp:users on pg

BEGIN;

SELECT id, email, password_hash, created_at
FROM users
WHERE false;

ROLLBACK;
```

### Deploy Changes

```bash
# Deploy to database
sqitch deploy

# Check status
sqitch status

# Revert last change
sqitch revert

# Deploy to specific target
sqitch deploy --to @v1.0.0
```

## Writing Safe Migrations

### The Golden Rules

```sql
-- RULE 1: Make additive changes first
-- Good: Add nullable column
ALTER TABLE users ADD COLUMN phone text;

-- Bad: Add non-nullable column without default
ALTER TABLE users ADD COLUMN phone text NOT NULL; -- Fails if table has data

-- RULE 2: Use transactions (when possible)
BEGIN;
ALTER TABLE orders ADD COLUMN status text DEFAULT 'pending';
UPDATE orders SET status = 'completed' WHERE shipped_at IS NOT NULL;
COMMIT;

-- RULE 3: Consider locking impact
-- Bad: Rewrite entire table during peak hours
ALTER TABLE huge_table ALTER COLUMN data TYPE jsonb USING data::jsonb;

-- Good: Create new column, migrate gradually
ALTER TABLE huge_table ADD COLUMN data_jsonb jsonb;
-- Migrate in batches in application
-- Later: drop old column

-- RULE 4: Make migrations idempotent
-- Good: Check if exists
DO $$
BEGIN
    IF NOT EXISTS (SELECT 1 FROM pg_indexes WHERE indexname = 'idx_users_email') THEN
        CREATE INDEX idx_users_email ON users(email);
    END IF;
END $$;
```

### Zero-Downtime Migrations

```sql
-- Pattern: Expand, Migrate, Contract

-- Step 1: Expand (add new structure)
ALTER TABLE users ADD COLUMN email_new text;

-- Step 2: Migrate (dual writes from application)
-- Application writes to both email and email_new

-- Step 3: Backfill
UPDATE users SET email_new = email WHERE email_new IS NULL;

-- Step 4: Switch reads to new column
-- Application now reads from email_new

-- Step 5: Contract (remove old structure)
ALTER TABLE users DROP COLUMN email;
ALTER TABLE users RENAME COLUMN email_new TO email;
```

## Testing Strategies

### Unit Testing Database Functions

```sql
-- Create test framework
CREATE SCHEMA IF NOT EXISTS test;

CREATE OR REPLACE FUNCTION test.assert_equals(
    expected anyelement,
    actual anyelement,
    message text DEFAULT ''
) RETURNS void AS $$
BEGIN
    IF expected IS DISTINCT FROM actual THEN
        RAISE EXCEPTION 'Assertion failed: % (expected: %, actual: %)',
            message, expected, actual;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Test your functions
CREATE OR REPLACE FUNCTION test.test_calculate_age()
RETURNS void AS $$
DECLARE
    result integer;
BEGIN
    -- Test normal case
    result := calculate_age('1990-01-01'::date);
    PERFORM test.assert_equals(
        EXTRACT(YEAR FROM age('1990-01-01'::date))::integer,
        result,
        'Age calculation failed'
    );

    -- Test edge case
    result := calculate_age(CURRENT_DATE);
    PERFORM test.assert_equals(0, result, 'Today birthday should be 0');

    RAISE NOTICE 'All tests passed!';
END;
$$ LANGUAGE plpgsql;

-- Run tests
SELECT test.test_calculate_age();
```

### Integration Testing with pgTAP

pgTAP provides comprehensive testing capabilities:

```bash
# Install pgTAP
CREATE EXTENSION IF NOT EXISTS pgtap;
```

```sql
-- Write tests
BEGIN;
SELECT plan(5);

-- Test table structure
SELECT has_table('users');
SELECT has_column('users', 'email');
SELECT col_type_is('users', 'email', 'text');

-- Test constraints
SELECT col_is_unique('users', 'email');
SELECT col_not_null('users', 'email');

-- Test functions
SELECT function_returns('calculate_total', ARRAY['integer', 'integer'], 'integer');

-- Test data
PREPARE insert_user AS INSERT INTO users (email) VALUES ($1) RETURNING id;
SELECT lives_ok('EXECUTE insert_user(''test@example.com'')', 'Insert should succeed');
SELECT throws_ok('EXECUTE insert_user(''test@example.com'')', '23505', 'Duplicate should fail');

SELECT * FROM finish();
ROLLBACK;
```

### Testing Migrations

```sql
-- Create migration test harness
CREATE OR REPLACE FUNCTION test.test_migration(
    migration_name text,
    up_script text,
    down_script text
) RETURNS void AS $$
DECLARE
    orig_schema jsonb;
    after_up_schema jsonb;
    after_down_schema jsonb;
BEGIN
    -- Capture original schema
    SELECT jsonb_agg(
        jsonb_build_object(
            'table', tablename,
            'columns', (
                SELECT jsonb_agg(column_name ORDER BY ordinal_position)
                FROM information_schema.columns
                WHERE table_name = tablename
            )
        )
    ) INTO orig_schema
    FROM pg_tables
    WHERE schemaname = 'public';

    -- Run up migration
    EXECUTE up_script;

    -- Capture schema after up
    SELECT jsonb_agg(
        jsonb_build_object(
            'table', tablename,
            'columns', (
                SELECT jsonb_agg(column_name ORDER BY ordinal_position)
                FROM information_schema.columns
                WHERE table_name = tablename
            )
        )
    ) INTO after_up_schema
    FROM pg_tables
    WHERE schemaname = 'public';

    -- Verify changes were made
    IF orig_schema = after_up_schema THEN
        RAISE EXCEPTION 'Migration % made no changes', migration_name;
    END IF;

    -- Run down migration
    EXECUTE down_script;

    -- Capture schema after down
    SELECT jsonb_agg(
        jsonb_build_object(
            'table', tablename,
            'columns', (
                SELECT jsonb_agg(column_name ORDER BY ordinal_position)
                FROM information_schema.columns
                WHERE table_name = tablename
            )
        )
    ) INTO after_down_schema
    FROM pg_tables
    WHERE schemaname = 'public';

    -- Verify rollback worked
    IF orig_schema IS DISTINCT FROM after_down_schema THEN
        RAISE EXCEPTION 'Migration % rollback failed', migration_name;
    END IF;

    RAISE NOTICE 'Migration % test passed', migration_name;
END;
$$ LANGUAGE plpgsql;
```

## Performance Testing

### Load Testing with pgbench

```bash
# Initialize pgbench tables
pgbench -i -s 100 mydb

# Run standard test
pgbench -c 10 -j 2 -t 1000 mydb

# Custom test script (test.sql)
\set aid random(1, 100000)
\set delta random(-5000, 5000)
BEGIN;
UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
COMMIT;

# Run custom test
pgbench -c 10 -j 2 -t 1000 -f test.sql mydb
```

### Query Performance Testing

```sql
-- Create performance test framework
CREATE TABLE IF NOT EXISTS test.performance_baselines (
    test_name text PRIMARY KEY,
    baseline_ms numeric NOT NULL,
    tolerance_percent integer DEFAULT 20
);

CREATE OR REPLACE FUNCTION test.performance_test(
    p_test_name text,
    p_query text
) RETURNS void AS $$
DECLARE
    start_time timestamp;
    end_time timestamp;
    execution_ms numeric;
    baseline test.performance_baselines%ROWTYPE;
BEGIN
    -- Run query and measure time
    start_time := clock_timestamp();
    EXECUTE p_query;
    end_time := clock_timestamp();

    execution_ms := EXTRACT(MILLISECONDS FROM (end_time - start_time));

    -- Check against baseline
    SELECT * INTO baseline
    FROM test.performance_baselines
    WHERE test_name = p_test_name;

    IF NOT FOUND THEN
        -- Create baseline
        INSERT INTO test.performance_baselines (test_name, baseline_ms)
        VALUES (p_test_name, execution_ms);
        RAISE NOTICE 'Baseline created for %: % ms', p_test_name, execution_ms;
    ELSE
        -- Compare with baseline
        IF execution_ms > baseline.baseline_ms * (1 + baseline.tolerance_percent / 100.0) THEN
            RAISE WARNING 'Performance regression in %: % ms (baseline: % ms)',
                p_test_name, execution_ms, baseline.baseline_ms;
        ELSE
            RAISE NOTICE 'Performance test % passed: % ms', p_test_name, execution_ms;
        END IF;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Use it
SELECT test.performance_test(
    'user_search',
    'SELECT * FROM users WHERE email LIKE ''%@example.com'' LIMIT 100'
);
```

## Continuous Integration for Databases

### GitHub Actions Example

```yaml
name: Database CI

on:
  pull_request:
    paths:
      - 'migrations/**'
      - '.github/workflows/database-ci.yml'

jobs:
  test-migrations:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
    - uses: actions/checkout@v3

    - name: Install PostgreSQL client
      run: |
        sudo apt-get update
        sudo apt-get install -y postgresql-client

    - name: Install Flyway
      run: |
        wget -qO- https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/9.22.0/flyway-commandline-9.22.0.tar.gz | tar xvz
        sudo ln -s $PWD/flyway-9.22.0/flyway /usr/local/bin

    - name: Run migrations
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb
      run: |
        createdb -h localhost -U postgres testdb
        flyway -url=jdbc:postgresql://localhost:5432/testdb -user=postgres -password=postgres migrate

    - name: Run tests
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb
      run: |
        psql $DATABASE_URL -f tests/schema_tests.sql
        psql $DATABASE_URL -f tests/data_tests.sql

    - name: Test rollback
      run: |
        flyway -url=jdbc:postgresql://localhost:5432/testdb -user=postgres -password=postgres undo
```

## Data Migration Patterns

### Batch Processing

```sql
-- Migrate large tables in batches
CREATE OR REPLACE PROCEDURE migrate_user_data(
    batch_size integer DEFAULT 1000
) AS $$
DECLARE
    rows_processed integer := 0;
    total_rows integer;
BEGIN
    SELECT count(*) INTO total_rows FROM users WHERE migrated = false;

    WHILE rows_processed < total_rows LOOP
        WITH batch AS (
            SELECT id FROM users
            WHERE migrated = false
            LIMIT batch_size
            FOR UPDATE SKIP LOCKED
        )
        UPDATE users u
        SET
            data_jsonb = to_jsonb(data_text),
            migrated = true
        FROM batch b
        WHERE u.id = b.id;

        rows_processed := rows_processed + batch_size;

        -- Give other transactions a chance
        PERFORM pg_sleep(0.1);

        RAISE NOTICE 'Processed % of % rows', rows_processed, total_rows;

        COMMIT;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### Parallel Migration

```sql
-- Use multiple workers for faster migration
CREATE OR REPLACE FUNCTION migrate_partition(
    start_id bigint,
    end_id bigint
) RETURNS void AS $$
BEGIN
    UPDATE large_table
    SET new_column = expensive_calculation(old_column)
    WHERE id BETWEEN start_id AND end_id
      AND new_column IS NULL;
END;
$$ LANGUAGE plpgsql;

-- Launch parallel workers (from application)
-- Worker 1: SELECT migrate_partition(1, 1000000);
-- Worker 2: SELECT migrate_partition(1000001, 2000000);
-- Worker 3: SELECT migrate_partition(2000001, 3000000);
```

## Rollback Strategies

### Immediate Rollback

```sql
-- Use transactions for instant rollback
BEGIN;
-- Multiple DDL statements
ALTER TABLE users ADD COLUMN temp text;
CREATE INDEX idx_temp ON users(temp);
-- Oops, something went wrong
ROLLBACK; -- Everything undone instantly
```

### Gradual Rollback

```sql
-- For changes that can't be in a transaction
CREATE OR REPLACE PROCEDURE safe_column_rename(
    p_table text,
    p_old_column text,
    p_new_column text
) AS $$
BEGIN
    -- Step 1: Add new column
    EXECUTE format('ALTER TABLE %I ADD COLUMN %I text', p_table, p_new_column);

    -- Step 2: Copy data
    EXECUTE format('UPDATE %I SET %I = %I', p_table, p_new_column, p_old_column);

    -- Step 3: Create backup of old column
    EXECUTE format('ALTER TABLE %I RENAME COLUMN %I TO %I',
        p_table, p_old_column, p_old_column || '_backup');

    -- Step 4: Rename new to old name
    EXECUTE format('ALTER TABLE %I RENAME COLUMN %I TO %I',
        p_table, p_new_column, p_old_column);

    -- Can still rollback by reversing these steps using _backup column
END;
$$ LANGUAGE plpgsql;
```

## Blue-Green Deployments

```sql
-- Create parallel schema for blue-green deployment
CREATE SCHEMA blue;
CREATE SCHEMA green;

-- Deploy to inactive schema
CREATE OR REPLACE PROCEDURE deploy_to_schema(
    target_schema text
) AS $$
BEGIN
    -- Create new version in target schema
    EXECUTE format('CREATE TABLE %I.users AS SELECT * FROM public.users', target_schema);

    -- Apply migrations to target schema
    EXECUTE format('ALTER TABLE %I.users ADD COLUMN new_feature text', target_schema);

    -- Test the new schema
    -- ... run tests ...

    -- Switch schemas atomically
    BEGIN;
        ALTER SCHEMA public RENAME TO old;
        ALTER SCHEMA target_schema RENAME TO public;
        ALTER SCHEMA old RENAME TO target_schema;
    COMMIT;
END;
$$ LANGUAGE plpgsql;
```

## Migration Monitoring

```sql
-- Track migration history
CREATE TABLE IF NOT EXISTS migration_history (
    id serial PRIMARY KEY,
    version text NOT NULL,
    description text,
    script_name text NOT NULL,
    applied_at timestamptz DEFAULT now(),
    applied_by text DEFAULT current_user,
    execution_time interval,
    success boolean DEFAULT true,
    error_message text,
    rollback_script text
);

-- Monitor migration impact
CREATE OR REPLACE FUNCTION log_migration(
    p_version text,
    p_description text,
    p_script text
) RETURNS void AS $$
DECLARE
    start_time timestamptz;
    end_time timestamptz;
BEGIN
    start_time := clock_timestamp();

    -- Execute migration
    BEGIN
        EXECUTE p_script;

        end_time := clock_timestamp();

        INSERT INTO migration_history (
            version, description, script_name,
            execution_time, success
        ) VALUES (
            p_version, p_description, p_script,
            end_time - start_time, true
        );
    EXCEPTION WHEN OTHERS THEN
        INSERT INTO migration_history (
            version, description, script_name,
            execution_time, success, error_message
        ) VALUES (
            p_version, p_description, p_script,
            clock_timestamp() - start_time, false, SQLERRM
        );
        RAISE;
    END;
END;
$$ LANGUAGE plpgsql;
```

## Testing Checklist

```sql
-- Comprehensive test suite
CREATE OR REPLACE FUNCTION run_all_tests()
RETURNS TABLE(
    test_category text,
    test_name text,
    status text,
    details text
) AS $$
BEGIN
    RETURN QUERY
    -- Schema tests
    SELECT 'Schema', 'Table Structure',
           CASE WHEN count(*) > 0 THEN 'PASS' ELSE 'FAIL' END,
           format('%s tables found', count(*))
    FROM information_schema.tables
    WHERE table_schema = 'public'

    UNION ALL

    -- Constraint tests
    SELECT 'Constraints', 'Foreign Keys',
           CASE WHEN count(*) > 0 THEN 'PASS' ELSE 'WARNING' END,
           format('%s foreign keys', count(*))
    FROM information_schema.table_constraints
    WHERE constraint_type = 'FOREIGN KEY'

    UNION ALL

    -- Index tests
    SELECT 'Performance', 'Indexes',
           CASE WHEN count(*) > 5 THEN 'PASS' ELSE 'WARNING' END,
           format('%s indexes', count(*))
    FROM pg_indexes
    WHERE schemaname = 'public'

    UNION ALL

    -- Data integrity tests
    SELECT 'Data', 'Orphaned Records',
           CASE WHEN count(*) = 0 THEN 'PASS' ELSE 'FAIL' END,
           format('%s orphaned records', count(*))
    FROM (
        SELECT o.id FROM orders o
        LEFT JOIN users u ON o.user_id = u.id
        WHERE u.id IS NULL
    ) orphaned;
END;
$$ LANGUAGE plpgsql;
```

## Exercise: Migration Marathon

1. **Setup Migration Tool**: Choose and configure Flyway or Sqitch
2. **Write Migrations**: Create 5 migrations with proper up/down scripts
3. **Test Migrations**: Write tests for each migration
4. **Performance Test**: Ensure migrations don't degrade performance
5. **CI Pipeline**: Set up automated testing for migrations
6. **Rollback Drill**: Practice rolling back under pressure

## The Reality Check

Testing and migrations aren't glamorous, but they're the difference between:
- "Deploy went smoothly" vs "We're rolling back at 2 AM"
- "Migration took 5 minutes" vs "Database locked for 2 hours"
- "Tests caught the issue" vs "Customers found the issue"

Invest in testing and migration infrastructure now, or pay the price in production later. The choice is yours, but only one lets you sleep peacefully.

## Summary

We've built a bulletproof deployment pipeline:
- Migration tools for version-controlled schema changes
- Safe migration patterns for zero-downtime deployments
- Comprehensive testing strategies from unit to integration
- Performance testing to catch regressions early
- CI/CD integration for automated validation
- Rollback strategies for when things go wrong

Remember: The database that never changes is the database that never improves. Master migrations and testing, and you can evolve fearlessly.

Next up: Chapter 11 - Real-World Patterns, where we'll apply everything we've learned to solve actual production challenges.