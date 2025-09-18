# Chapter 6: Advanced Features - Beyond Basic SQL

## Chapter Overview

In this chapter, you'll learn:
- Working with JSON/JSONB for flexible data structures
- Using arrays and composite types
- Full-text search capabilities
- Custom data types and domains
- Stored procedures and functions
- Triggers for automation
- Views and materialized views
- Table partitioning strategies
- Foreign data wrappers

By the end of this chapter, you'll understand the features that make PostgreSQL more than just a relational database—it's a complete data platform.

## JSON and JSONB

PostgreSQL offers two JSON types: `JSON` (stores exact text) and `JSONB` (binary format, supports indexing). Use JSONB unless you need to preserve exact formatting.

### Basic JSON Operations

```sql
-- Create a table with JSONB
CREATE TABLE user_profiles (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    profile JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert JSON data
INSERT INTO user_profiles (username, profile) VALUES
('alice', '{
    "name": "Alice Johnson",
    "age": 28,
    "email": "alice@example.com",
    "interests": ["reading", "hiking", "photography"],
    "address": {
        "city": "Seattle",
        "state": "WA",
        "zip": "98101"
    },
    "premium": true
}'),
('bob', '{
    "name": "Bob Smith",
    "age": 35,
    "email": "bob@example.com",
    "interests": ["cooking", "music"],
    "address": {
        "city": "Portland",
        "state": "OR",
        "zip": "97201"
    },
    "premium": false,
    "metadata": {
        "last_login": "2024-01-15",
        "account_type": "basic"
    }
}');
```

### Querying JSON Data

```sql
-- Extract values with -> (returns JSON) and ->> (returns text)
SELECT
    username,
    profile->>'name' AS full_name,
    profile->>'age' AS age,
    profile->'address'->>'city' AS city
FROM user_profiles;

-- Extract nested values
SELECT
    username,
    profile#>>'{address,city}' AS city,  -- Path notation
    profile#>>'{metadata,account_type}' AS account_type
FROM user_profiles;

-- Query JSON arrays
SELECT
    username,
    jsonb_array_elements_text(profile->'interests') AS interest
FROM user_profiles;

-- Check if key exists
SELECT username
FROM user_profiles
WHERE profile ? 'metadata';  -- Has metadata key

-- Check if any of multiple keys exist
SELECT username
FROM user_profiles
WHERE profile ?| array['metadata', 'settings'];

-- Check if all keys exist
SELECT username
FROM user_profiles
WHERE profile ?& array['name', 'email', 'age'];
```

### JSON Operators and Functions

```sql
-- Contains operator @>
SELECT username
FROM user_profiles
WHERE profile @> '{"premium": true}';

-- Contained by operator <@
SELECT username
FROM user_profiles
WHERE '{"city": "Seattle"}' <@ (profile->'address');

-- Path exists operator @?
SELECT username
FROM user_profiles
WHERE profile @? '$.address.city ? (@ == "Seattle")';

-- Build JSON
SELECT
    jsonb_build_object(
        'user', username,
        'city', profile#>>'{address,city}',
        'is_premium', profile->>'premium'
    ) AS user_summary
FROM user_profiles;

-- Aggregate into JSON
SELECT
    jsonb_agg(
        jsonb_build_object('id', id, 'name', profile->>'name')
    ) AS all_users
FROM user_profiles;

-- Pretty-print JSON
SELECT jsonb_pretty(profile) FROM user_profiles LIMIT 1;
```

### Modifying JSON Data

```sql
-- Update entire JSON column
UPDATE user_profiles
SET profile = profile || '{"verified": true}'
WHERE username = 'alice';

-- Update nested value
UPDATE user_profiles
SET profile = jsonb_set(
    profile,
    '{address,country}',
    '"USA"'
)
WHERE username = 'alice';

-- Update or insert nested value
UPDATE user_profiles
SET profile = jsonb_set(
    profile,
    '{settings,theme}',
    '"dark"',
    true  -- Create if missing
);

-- Remove a key
UPDATE user_profiles
SET profile = profile - 'metadata'
WHERE username = 'bob';

-- Remove nested key
UPDATE user_profiles
SET profile = profile #- '{address,zip}'
WHERE username = 'alice';

-- Append to array
UPDATE user_profiles
SET profile = jsonb_set(
    profile,
    '{interests}',
    profile->'interests' || '["painting"]'::jsonb
)
WHERE username = 'alice';
```

### Indexing JSON Data

```sql
-- GIN index for containment queries
CREATE INDEX idx_profile_gin ON user_profiles USING GIN (profile);

-- Now these queries are fast:
SELECT * FROM user_profiles WHERE profile @> '{"premium": true}';
SELECT * FROM user_profiles WHERE profile ? 'metadata';

-- Index specific JSON field
CREATE INDEX idx_profile_email ON user_profiles ((profile->>'email'));

-- Expression index for nested field
CREATE INDEX idx_profile_city ON user_profiles ((profile#>>'{address,city}'));

-- Partial index for filtered data
CREATE INDEX idx_premium_users ON user_profiles (username)
WHERE (profile->>'premium')::boolean = true;
```

## Arrays

Arrays allow storing multiple values of the same type in a single column.

### Array Basics

```sql
-- Create table with arrays
CREATE TABLE projects (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    tags TEXT[],
    team_members VARCHAR(50)[],
    budget_by_quarter DECIMAL(10,2)[],
    milestone_dates DATE[]
);

-- Insert arrays
INSERT INTO projects (name, tags, team_members, budget_by_quarter, milestone_dates) VALUES
('Website Redesign',
 ARRAY['frontend', 'design', 'urgent'],
 ARRAY['Alice', 'Bob', 'Charlie'],
 ARRAY[50000, 45000, 40000, 35000],
 ARRAY['2024-03-31', '2024-06-30', '2024-09-30']::DATE[]
),
('Mobile App',
 '{"backend", "mobile", "ios", "android"}',  -- Alternative syntax
 '{"Dave", "Eve"}',
 '{60000, 60000, 55000, 55000}',
 NULL
);

-- Array constructors
INSERT INTO projects (name, tags) VALUES
('Data Pipeline', ARRAY['data', 'etl', 'python']);
```

### Querying Arrays

```sql
-- Access array elements (1-indexed!)
SELECT
    name,
    tags[1] AS first_tag,
    tags[2] AS second_tag,
    team_members[1] AS lead_developer
FROM projects;

-- Array slicing
SELECT
    name,
    tags[1:2] AS first_two_tags,
    budget_by_quarter[2:4] AS q2_to_q4_budget
FROM projects;

-- Array length
SELECT
    name,
    array_length(tags, 1) AS tag_count,
    cardinality(team_members) AS team_size  -- Alternative
FROM projects;

-- Check if value exists in array
SELECT name
FROM projects
WHERE 'urgent' = ANY(tags);

-- Check if all values match
SELECT name
FROM projects
WHERE ARRAY['frontend', 'design'] <@ tags;  -- Is contained by

-- Array overlap
SELECT name
FROM projects
WHERE tags && ARRAY['mobile', 'web'];  -- Has any of these

-- Unnest arrays
SELECT
    p.name,
    unnest(p.tags) AS tag
FROM projects p;

-- Unnest with ordinality (position)
SELECT
    p.name,
    tag,
    position
FROM projects p,
    unnest(p.tags) WITH ORDINALITY AS t(tag, position);
```

### Array Aggregation

```sql
-- Aggregate into array
SELECT
    array_agg(DISTINCT tag) AS all_tags
FROM projects, unnest(tags) AS tag;

-- Aggregate with ordering
SELECT
    array_agg(name ORDER BY name) AS project_names
FROM projects;

-- Array concatenation
SELECT
    name,
    tags || ARRAY['reviewed'] AS updated_tags
FROM projects;

-- Array functions
SELECT
    name,
    array_remove(tags, 'urgent') AS non_urgent_tags,
    array_replace(tags, 'frontend', 'ui') AS updated_tags,
    array_cat(tags, ARRAY['2024']) AS all_tags
FROM projects;
```

## Full-Text Search

PostgreSQL includes powerful full-text search capabilities that rival dedicated search engines for many use cases.

### Setting Up Full-Text Search

```sql
-- Create articles table
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    body TEXT NOT NULL,
    author VARCHAR(100),
    published_date DATE,
    tags TEXT[],
    search_vector tsvector
);

-- Insert sample articles
INSERT INTO articles (title, body, author, published_date, tags) VALUES
('Introduction to PostgreSQL',
 'PostgreSQL is a powerful, open source object-relational database system. It has a strong reputation for reliability, feature robustness, and performance.',
 'John Doe',
 '2024-01-15',
 ARRAY['database', 'postgresql', 'tutorial']),
('Advanced PostgreSQL Features',
 'PostgreSQL offers advanced features like full-text search, JSON support, and window functions. These features make PostgreSQL suitable for modern applications.',
 'Jane Smith',
 '2024-01-20',
 ARRAY['database', 'postgresql', 'advanced']);

-- Update search vectors
UPDATE articles
SET search_vector =
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(body, '')), 'B') ||
    setweight(to_tsvector('english', coalesce(author, '')), 'C');

-- Create GIN index
CREATE INDEX idx_articles_search ON articles USING GIN (search_vector);

-- Trigger to automatically update search vector
CREATE OR REPLACE FUNCTION articles_search_vector_update()
RETURNS trigger AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(NEW.body, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(NEW.author, '')), 'C');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER articles_search_vector_trigger
BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION articles_search_vector_update();
```

### Full-Text Search Queries

```sql
-- Basic text search
SELECT title, author
FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgresql & features');

-- Phrase search
SELECT title
FROM articles
WHERE search_vector @@ phraseto_tsquery('english', 'open source database');

-- Ranking results
SELECT
    title,
    ts_rank(search_vector, query) AS rank
FROM articles,
    to_tsquery('english', 'postgresql | database') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- Highlighting matches
SELECT
    title,
    ts_headline('english', body, query,
        'StartSel=<mark>, StopSel=</mark>, MaxWords=20, MinWords=10'
    ) AS excerpt
FROM articles,
    to_tsquery('english', 'postgresql') AS query
WHERE search_vector @@ query;

-- Search with stemming
SELECT title
FROM articles
WHERE search_vector @@ to_tsquery('english', 'databases');  -- Matches 'database'

-- Websearch syntax (PostgreSQL 11+)
SELECT title
FROM articles
WHERE search_vector @@ websearch_to_tsquery('english', 'postgresql advanced -basic');
```

## Custom Types and Domains

### Domains

Domains are custom data types based on existing types with additional constraints.

```sql
-- Create domains
CREATE DOMAIN email AS VARCHAR(255)
CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$');

CREATE DOMAIN us_zip_code AS VARCHAR(10)
CHECK (VALUE ~ '^\d{5}(-\d{4})?$');

CREATE DOMAIN positive_decimal AS DECIMAL(10,2)
CHECK (VALUE > 0);

CREATE DOMAIN phone_number AS VARCHAR(20)
CHECK (VALUE ~ '^\+?[1-9]\d{1,14}$');  -- E.164 format

-- Use domains in tables
CREATE TABLE contacts (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email_address email,
    phone phone_number,
    zip_code us_zip_code,
    account_balance positive_decimal
);

-- Domains enforce constraints
INSERT INTO contacts (name, email_address) VALUES
('John', 'invalid-email');  -- ERROR: violates check constraint

INSERT INTO contacts (name, email_address, zip_code) VALUES
('John', 'john@example.com', '12345'),     -- Valid
('Jane', 'jane@example.com', '12345-6789'); -- Valid
```

### Composite Types

```sql
-- Create composite type
CREATE TYPE address AS (
    street VARCHAR(100),
    city VARCHAR(50),
    state CHAR(2),
    zip_code us_zip_code,
    country VARCHAR(50)
);

CREATE TYPE price_range AS (
    min_price DECIMAL(10,2),
    max_price DECIMAL(10,2)
);

-- Use composite types
CREATE TABLE businesses (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    primary_address address,
    billing_address address,
    price_range price_range
);

-- Insert composite type data
INSERT INTO businesses (name, primary_address, billing_address, price_range) VALUES
('Tech Corp',
 ROW('123 Main St', 'Seattle', 'WA', '98101', 'USA'),
 ROW('456 Bill Ave', 'Seattle', 'WA', '98102', 'USA'),
 ROW(100, 500)
);

-- Access composite type fields
SELECT
    name,
    (primary_address).city,
    (primary_address).state,
    (price_range).min_price
FROM businesses;

-- Update composite type field
UPDATE businesses
SET primary_address.street = '789 New St'
WHERE id = 1;
```

### Enum Types

```sql
-- Create enum type
CREATE TYPE order_status AS ENUM (
    'pending',
    'confirmed',
    'processing',
    'shipped',
    'delivered',
    'cancelled',
    'refunded'
);

CREATE TYPE priority_level AS ENUM ('low', 'medium', 'high', 'urgent');

-- Use enum in table
CREATE TABLE support_tickets (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    priority priority_level DEFAULT 'medium',
    status order_status DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Enums provide type safety
INSERT INTO support_tickets (title, priority) VALUES
('System down', 'urgent'),
('Feature request', 'low');

-- This fails:
INSERT INTO support_tickets (title, priority) VALUES
('Test', 'very-high');  -- ERROR: invalid input value

-- Query enum values
SELECT enum_range(NULL::order_status);
```

## Stored Procedures and Functions

### Functions

```sql
-- Simple function
CREATE OR REPLACE FUNCTION calculate_tax(amount DECIMAL)
RETURNS DECIMAL AS $$
BEGIN
    RETURN amount * 0.08;  -- 8% tax
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Use function
SELECT price, calculate_tax(price) AS tax
FROM products;

-- Function with multiple parameters
CREATE OR REPLACE FUNCTION calculate_discount(
    price DECIMAL,
    discount_percent DECIMAL DEFAULT 0
)
RETURNS DECIMAL AS $$
BEGIN
    RETURN price * (1 - discount_percent / 100);
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Table-returning function
CREATE OR REPLACE FUNCTION get_products_by_category(category_name VARCHAR)
RETURNS TABLE(
    product_id INT,
    product_name VARCHAR,
    price DECIMAL
) AS $$
BEGIN
    RETURN QUERY
    SELECT p.id, p.name, p.unit_price
    FROM products p
    JOIN categories c ON p.category_id = c.id
    WHERE c.name = category_name;
END;
$$ LANGUAGE plpgsql STABLE;

-- Use table function
SELECT * FROM get_products_by_category('Electronics');

-- Function with OUT parameters
CREATE OR REPLACE FUNCTION get_order_summary(
    order_id_param INT,
    OUT total_items INT,
    OUT total_amount DECIMAL
) AS $$
BEGIN
    SELECT
        COUNT(*),
        SUM(unit_price * quantity * (1 - discount))
    INTO total_items, total_amount
    FROM order_items
    WHERE order_id = order_id_param;
END;
$$ LANGUAGE plpgsql;

-- Use function with OUT parameters
SELECT * FROM get_order_summary(1);
```

### Stored Procedures (PostgreSQL 11+)

```sql
-- Procedure with transaction control
CREATE OR REPLACE PROCEDURE transfer_funds(
    from_account INT,
    to_account INT,
    amount DECIMAL
)
LANGUAGE plpgsql
AS $$
DECLARE
    from_balance DECIMAL;
BEGIN
    -- Lock accounts in order to prevent deadlock
    IF from_account < to_account THEN
        PERFORM * FROM accounts WHERE id = from_account FOR UPDATE;
        PERFORM * FROM accounts WHERE id = to_account FOR UPDATE;
    ELSE
        PERFORM * FROM accounts WHERE id = to_account FOR UPDATE;
        PERFORM * FROM accounts WHERE id = from_account FOR UPDATE;
    END IF;

    -- Check balance
    SELECT balance INTO from_balance
    FROM accounts WHERE id = from_account;

    IF from_balance < amount THEN
        RAISE EXCEPTION 'Insufficient funds';
    END IF;

    -- Transfer
    UPDATE accounts SET balance = balance - amount WHERE id = from_account;
    UPDATE accounts SET balance = balance + amount WHERE id = to_account;

    -- Log transaction
    INSERT INTO transaction_log (from_account, to_account, amount)
    VALUES (from_account, to_account, amount);

    COMMIT;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
$$;

-- Call procedure
CALL transfer_funds(1, 2, 100.00);

-- Procedure with INOUT parameters
CREATE OR REPLACE PROCEDURE process_order(
    INOUT order_id INT,
    IN customer_id INT
)
LANGUAGE plpgsql
AS $$
BEGIN
    -- Create order
    INSERT INTO orders (customer_id, order_status)
    VALUES (customer_id, 'pending')
    RETURNING id INTO order_id;

    -- Additional processing...
END;
$$;
```

## Triggers

Triggers automatically execute functions in response to database events.

### Basic Triggers

```sql
-- Audit trigger
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name VARCHAR(50),
    operation VARCHAR(10),
    user_name VARCHAR(50),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    row_data JSONB
);

CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, operation, user_name, row_data)
    VALUES (
        TG_TABLE_NAME,
        TG_OP,
        current_user,
        CASE
            WHEN TG_OP = 'DELETE' THEN to_jsonb(OLD)
            ELSE to_jsonb(NEW)
        END
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_audit
AFTER INSERT OR UPDATE OR DELETE ON products
FOR EACH ROW EXECUTE FUNCTION audit_trigger();

-- Update timestamp trigger
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_products_timestamp
BEFORE UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION update_timestamp();

-- Validation trigger
CREATE OR REPLACE FUNCTION validate_stock()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.stock_quantity < 0 THEN
        RAISE EXCEPTION 'Stock cannot be negative';
    END IF;

    IF NEW.stock_quantity < NEW.reorder_level THEN
        -- Send notification (pseudo-code)
        INSERT INTO notifications (message)
        VALUES ('Low stock alert for product: ' || NEW.name);
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER validate_product_stock
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION validate_stock();
```

### Event Triggers

```sql
-- DDL event trigger
CREATE OR REPLACE FUNCTION log_ddl_changes()
RETURNS event_trigger AS $$
DECLARE
    obj record;
BEGIN
    FOR obj IN SELECT * FROM pg_event_trigger_ddl_commands()
    LOOP
        INSERT INTO ddl_log (command_tag, object_type, object_identity)
        VALUES (obj.command_tag, obj.object_type, obj.object_identity);
    END LOOP;
END;
$$ LANGUAGE plpgsql;

CREATE EVENT TRIGGER log_ddl_trigger
ON ddl_command_end
EXECUTE FUNCTION log_ddl_changes();
```

## Views and Materialized Views

### Regular Views

```sql
-- Simple view
CREATE VIEW active_products AS
SELECT * FROM products
WHERE discontinued = FALSE
  AND stock_quantity > 0;

-- Complex view with joins
CREATE VIEW order_details AS
SELECT
    o.id AS order_id,
    o.order_number,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.email AS customer_email,
    o.order_date,
    o.order_status,
    SUM(oi.unit_price * oi.quantity * (1 - oi.discount)) AS total_amount,
    COUNT(oi.id) AS item_count
FROM orders o
JOIN customers c ON o.customer_id = c.id
LEFT JOIN order_items oi ON o.id = oi.order_id
GROUP BY o.id, o.order_number, c.first_name, c.last_name, c.email;

-- Updatable view
CREATE VIEW customer_contacts AS
SELECT id, first_name, last_name, email, phone
FROM customers;

-- Can update through view
UPDATE customer_contacts
SET phone = '555-1234'
WHERE id = 1;

-- View with security
CREATE VIEW my_orders AS
SELECT * FROM orders
WHERE customer_id = current_setting('app.current_user_id')::INT
WITH CHECK OPTION;  -- Prevents inserting rows that wouldn't be visible
```

### Materialized Views

```sql
-- Create materialized view
CREATE MATERIALIZED VIEW product_sales_summary AS
SELECT
    p.id AS product_id,
    p.name AS product_name,
    c.name AS category,
    COUNT(DISTINCT oi.order_id) AS times_ordered,
    SUM(oi.quantity) AS total_quantity_sold,
    SUM(oi.unit_price * oi.quantity * (1 - oi.discount)) AS total_revenue,
    AVG(r.rating) AS avg_rating,
    COUNT(DISTINCT r.customer_id) AS review_count
FROM products p
LEFT JOIN categories c ON p.category_id = c.id
LEFT JOIN order_items oi ON p.id = oi.product_id
LEFT JOIN reviews r ON p.id = r.product_id
GROUP BY p.id, p.name, c.name;

-- Create indexes on materialized view
CREATE INDEX idx_product_sales_revenue
ON product_sales_summary(total_revenue DESC);

-- Refresh materialized view
REFRESH MATERIALIZED VIEW product_sales_summary;

-- Concurrent refresh (doesn't lock)
CREATE UNIQUE INDEX idx_product_sales_product
ON product_sales_summary(product_id);

REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_summary;

-- Automatic refresh using pg_cron (extension required)
SELECT cron.schedule(
    'refresh-product-sales',
    '0 2 * * *',  -- 2 AM daily
    'REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_summary;'
);
```

## Table Partitioning

Partitioning divides large tables into smaller, more manageable pieces.

### Range Partitioning

```sql
-- Create partitioned table
CREATE TABLE sales_data (
    id SERIAL,
    sale_date DATE NOT NULL,
    product_id INT,
    quantity INT,
    amount DECIMAL(10,2),
    PRIMARY KEY (id, sale_date)
) PARTITION BY RANGE (sale_date);

-- Create partitions
CREATE TABLE sales_data_2024_q1 PARTITION OF sales_data
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE sales_data_2024_q2 PARTITION OF sales_data
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

CREATE TABLE sales_data_2024_q3 PARTITION OF sales_data
    FOR VALUES FROM ('2024-07-01') TO ('2024-10-01');

CREATE TABLE sales_data_2024_q4 PARTITION OF sales_data
    FOR VALUES FROM ('2024-10-01') TO ('2025-01-01');

-- Create indexes on partitions
CREATE INDEX idx_sales_2024_q1_date ON sales_data_2024_q1(sale_date);
CREATE INDEX idx_sales_2024_q2_date ON sales_data_2024_q2(sale_date);

-- Automatic routing
INSERT INTO sales_data (sale_date, product_id, quantity, amount)
VALUES ('2024-03-15', 1, 5, 99.99);  -- Goes to Q1 partition

-- Query across partitions
SELECT * FROM sales_data
WHERE sale_date BETWEEN '2024-02-01' AND '2024-05-31';
```

### List Partitioning

```sql
-- Partition by region
CREATE TABLE customer_data (
    id SERIAL,
    name VARCHAR(100),
    email VARCHAR(255),
    region VARCHAR(20) NOT NULL,
    PRIMARY KEY (id, region)
) PARTITION BY LIST (region);

CREATE TABLE customer_data_us PARTITION OF customer_data
    FOR VALUES IN ('US-East', 'US-West', 'US-Central');

CREATE TABLE customer_data_europe PARTITION OF customer_data
    FOR VALUES IN ('EU-West', 'EU-East', 'EU-North');

CREATE TABLE customer_data_asia PARTITION OF customer_data
    FOR VALUES IN ('Asia-Pacific', 'Asia-East', 'Asia-South');
```

### Hash Partitioning

```sql
-- Partition by hash for even distribution
CREATE TABLE user_sessions (
    id UUID,
    user_id INT NOT NULL,
    session_data JSONB,
    created_at TIMESTAMP,
    PRIMARY KEY (id, user_id)
) PARTITION BY HASH (user_id);

-- Create 4 hash partitions
CREATE TABLE user_sessions_0 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE user_sessions_1 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE user_sessions_2 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE user_sessions_3 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

## Extensions

PostgreSQL's extension system allows adding powerful capabilities.

### Common Extensions

```sql
-- UUID generation
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
SELECT uuid_generate_v4();

-- Cryptographic functions
CREATE EXTENSION IF NOT EXISTS pgcrypto;
SELECT encode(digest('password', 'sha256'), 'hex');

-- Trigram similarity
CREATE EXTENSION IF NOT EXISTS pg_trgm;
SELECT similarity('PostgreSQL', 'PostGRESQL');

-- Foreign data wrapper
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

-- Full list of available extensions
SELECT * FROM pg_available_extensions ORDER BY name;

-- Installed extensions
SELECT * FROM pg_extension;
```

### PostGIS for Geographic Data

```sql
-- Install PostGIS
CREATE EXTENSION IF NOT EXISTS postgis;

-- Create table with geographic data
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    position GEOGRAPHY(POINT, 4326)
);

-- Insert geographic data
INSERT INTO locations (name, position) VALUES
('Seattle', ST_GeogFromText('POINT(-122.3321 47.6062)')),
('Portland', ST_GeogFromText('POINT(-122.6765 45.5152)'));

-- Distance calculations
SELECT
    l1.name AS from_city,
    l2.name AS to_city,
    ST_Distance(l1.position, l2.position) / 1000 AS distance_km
FROM locations l1, locations l2
WHERE l1.id < l2.id;

-- Find nearby locations
SELECT name
FROM locations
WHERE ST_DWithin(
    position,
    ST_GeogFromText('POINT(-122.3 47.6)'),
    50000  -- Within 50km
);
```

## Table Inheritance

PostgreSQL supports object-oriented concepts through table inheritance.

```sql
-- Base table
CREATE TABLE vehicles (
    id SERIAL PRIMARY KEY,
    make TEXT NOT NULL,
    model TEXT NOT NULL,
    year INTEGER NOT NULL,
    color TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Child tables inherit all columns
CREATE TABLE cars (
    doors INTEGER NOT NULL,
    fuel_type TEXT
) INHERITS (vehicles);

CREATE TABLE motorcycles (
    engine_cc INTEGER,
    has_sidecar BOOLEAN DEFAULT FALSE
) INHERITS (vehicles);

-- Insert into child tables
INSERT INTO cars (make, model, year, color, doors, fuel_type)
VALUES ('Toyota', 'Camry', 2024, 'Blue', 4, 'Hybrid');

INSERT INTO motorcycles (make, model, year, color, engine_cc, has_sidecar)
VALUES ('Harley-Davidson', 'Street 750', 2023, 'Black', 750, FALSE);

-- Query parent table shows all records
SELECT * FROM vehicles;  -- Shows both cars and motorcycles

-- Query only specific type
SELECT * FROM ONLY vehicles;  -- Only direct inserts to vehicles
SELECT * FROM cars;          -- Only cars

-- Add constraint to parent affects children
ALTER TABLE vehicles ADD CHECK (year >= 1900);

-- Caveats: Primary keys and unique constraints don't inherit
-- Need to create separately on each child table
```

## Foreign Data Wrappers

Access external data sources as if they were PostgreSQL tables.

### PostgreSQL to PostgreSQL

```sql
-- Install extension
CREATE EXTENSION postgres_fdw;

-- Create server connection
CREATE SERVER remote_db
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'remote.example.com', port '5432', dbname 'remote_db');

-- Create user mapping
CREATE USER MAPPING FOR current_user
SERVER remote_db
OPTIONS (user 'remote_user', password 'secret_password');

-- Import foreign schema
IMPORT FOREIGN SCHEMA public
LIMIT TO (users, orders, products)
FROM SERVER remote_db
INTO local_schema;

-- Query remote tables
SELECT * FROM local_schema.users;

-- Join local and remote tables
SELECT l.*, r.order_count
FROM local_customers l
JOIN local_schema.users r ON l.email = r.email;
```

### CSV Files as Tables

```sql
-- Install file FDW
CREATE EXTENSION file_fdw;

-- Create server
CREATE SERVER csv_files FOREIGN DATA WRAPPER file_fdw;

-- Create foreign table for CSV
CREATE FOREIGN TABLE sales_import (
    date DATE,
    product_id INTEGER,
    quantity INTEGER,
    amount DECIMAL(10,2)
) SERVER csv_files
OPTIONS (
    filename '/data/imports/sales_2024.csv',
    format 'csv',
    header 'true'
);

-- Query CSV file like a table
SELECT * FROM sales_import WHERE date >= '2024-01-01';

-- Load into permanent table
INSERT INTO sales (date, product_id, quantity, amount)
SELECT * FROM sales_import
WHERE date NOT IN (SELECT date FROM sales);
```

### MongoDB Integration

```sql
-- Install MongoDB FDW
CREATE EXTENSION mongo_fdw;

-- Create server
CREATE SERVER mongodb_server
FOREIGN DATA WRAPPER mongo_fdw
OPTIONS (address 'mongodb.example.com', port '27017');

-- Create user mapping
CREATE USER MAPPING FOR current_user
SERVER mongodb_server
OPTIONS (username 'mongo_user', password 'mongo_pass');

-- Create foreign table
CREATE FOREIGN TABLE mongo_products (
    _id NAME,
    name TEXT,
    price NUMERIC,
    categories TEXT[],
    specs JSONB
) SERVER mongodb_server
OPTIONS (database 'store', collection 'products');

-- Query MongoDB from PostgreSQL
SELECT name, price, specs->>'color' AS color
FROM mongo_products
WHERE price > 100;
```

## LISTEN/NOTIFY - Real-time Events

PostgreSQL's built-in pub/sub system for real-time notifications.

```sql
-- Create notification trigger
CREATE OR REPLACE FUNCTION notify_order_status_change()
RETURNS TRIGGER AS $$
BEGIN
    -- Send notification with payload
    PERFORM pg_notify(
        'order_status_changed',
        json_build_object(
            'order_id', NEW.id,
            'old_status', OLD.status,
            'new_status', NEW.status,
            'customer_id', NEW.customer_id,
            'timestamp', NOW()
        )::text
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Attach trigger to orders table
CREATE TRIGGER order_status_change_trigger
AFTER UPDATE OF status ON orders
FOR EACH ROW
WHEN (OLD.status IS DISTINCT FROM NEW.status)
EXECUTE FUNCTION notify_order_status_change();

-- In application code (Python example):
-- import psycopg2
-- conn = psycopg2.connect(...)
-- conn.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_AUTOCOMMIT)
-- cur = conn.cursor()
-- cur.execute("LISTEN order_status_changed;")
--
-- while True:
--     conn.poll()
--     while conn.notifies:
--         notify = conn.notifies.pop(0)
--         print(f"Got notification: {notify.payload}")

-- SQL client can listen too
LISTEN order_status_changed;

-- In another session, update an order
UPDATE orders SET status = 'shipped' WHERE id = 123;

-- First session receives:
-- Asynchronous notification "order_status_changed" with payload
-- "{"order_id": 123, "old_status": "processing", "new_status": "shipped", ...}"
```

### Real-time Dashboard Example

```sql
-- Create comprehensive notification system
CREATE OR REPLACE FUNCTION notify_dashboard_event()
RETURNS TRIGGER AS $$
DECLARE
    channel TEXT;
    payload JSONB;
BEGIN
    -- Determine channel based on table
    channel := CASE TG_TABLE_NAME
        WHEN 'orders' THEN 'dashboard_orders'
        WHEN 'users' THEN 'dashboard_users'
        WHEN 'products' THEN 'dashboard_products'
        ELSE 'dashboard_general'
    END;

    -- Build payload
    payload := jsonb_build_object(
        'operation', TG_OP,
        'table', TG_TABLE_NAME,
        'timestamp', NOW(),
        'user', current_user,
        'data', CASE
            WHEN TG_OP = 'DELETE' THEN to_jsonb(OLD)
            WHEN TG_OP = 'UPDATE' THEN jsonb_build_object(
                'old', to_jsonb(OLD),
                'new', to_jsonb(NEW)
            )
            ELSE to_jsonb(NEW)
        END
    );

    -- Send notification
    PERFORM pg_notify(channel, payload::text);

    -- Return appropriate value
    RETURN CASE
        WHEN TG_OP = 'DELETE' THEN OLD
        ELSE NEW
    END;
END;
$$ LANGUAGE plpgsql;

-- Apply to multiple tables
CREATE TRIGGER dashboard_trigger
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION notify_dashboard_event();

CREATE TRIGGER dashboard_trigger
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION notify_dashboard_event();

-- Rate limiting notifications
CREATE OR REPLACE FUNCTION notify_with_throttle()
RETURNS TRIGGER AS $$
DECLARE
    last_notification TIMESTAMP;
BEGIN
    -- Check last notification time
    SELECT last_notified INTO last_notification
    FROM notification_throttle
    WHERE channel = 'high_volume_channel';

    IF last_notification IS NULL OR
       last_notification < NOW() - INTERVAL '1 second' THEN
        -- Send notification
        PERFORM pg_notify('high_volume_channel', to_jsonb(NEW)::text);

        -- Update throttle
        INSERT INTO notification_throttle (channel, last_notified)
        VALUES ('high_volume_channel', NOW())
        ON CONFLICT (channel)
        DO UPDATE SET last_notified = NOW();
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## Exercises

### Exercise 6.1: JSON Operations
1. Create a product catalog using JSONB for flexible attributes
2. Implement search across JSON fields
3. Update nested JSON values based on conditions

### Exercise 6.2: Full-Text Search
1. Build a blog system with full-text search
2. Implement search with ranking and highlighting
3. Create a search suggestion feature

### Exercise 6.3: Custom Types
1. Design custom types for a financial application
2. Create validation rules using domains
3. Implement an audit system using triggers

### Exercise 6.4: Performance with Partitioning
1. Partition a large log table by date
2. Measure query performance before and after
3. Implement automatic partition creation

## Summary

You've learned PostgreSQL's advanced features:

✅ **JSON/JSONB**: Document storage with SQL power
✅ **Arrays**: Multiple values in single columns
✅ **Full-Text Search**: Built-in search engine capabilities
✅ **Custom Types**: Domain-specific data types
✅ **Functions & Procedures**: Database-side logic
✅ **Triggers**: Automated reactions to events
✅ **Views**: Virtual tables and materialized summaries
✅ **Partitioning**: Managing large tables efficiently
✅ **Extensions**: Expanding PostgreSQL's capabilities

These features transform PostgreSQL from a simple relational database into a comprehensive data platform capable of handling diverse workloads.

## What's Next

Chapter 7 focuses on performance optimization. You'll learn how to make PostgreSQL fast, diagnose performance problems, and tune for specific workloads.

## Additional Resources

- **JSONB Documentation**: [postgresql.org/docs/current/datatype-json.html](https://www.postgresql.org/docs/current/datatype-json.html)
- **Full-Text Search**: [postgresql.org/docs/current/textsearch.html](https://www.postgresql.org/docs/current/textsearch.html)
- **Table Partitioning**: [postgresql.org/docs/current/ddl-partitioning.html](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- **PostGIS**: [postgis.net](https://postgis.net/)

---

*"PostgreSQL's advanced features aren't just checkboxes—they're production-ready tools that solve real problems."*