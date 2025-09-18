# Chapter 5: Advanced PostgreSQL Features - The Good Stuff

This is where PostgreSQL flexes. While other databases were arguing about NoSQL vs SQL, PostgreSQL just quietly added both. And then added full-text search. And geographic information systems. And...

Let's explore the features that make PostgreSQL the "everything database."

## JSONB: NoSQL Inside SQL

Remember when MongoDB said relational databases couldn't handle JSON? PostgreSQL took that personally.

### Setting Up JSON Data

```sql
-- Create a products table with JSON
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    details JSONB
);

-- Insert some products
INSERT INTO products (name, details) VALUES
('iPhone 15', '{
    "brand": "Apple",
    "specs": {
        "storage": "256GB",
        "color": "Blue",
        "camera": "48MP"
    },
    "price": 999,
    "tags": ["smartphone", "premium", "5G"]
}'::jsonb),
('MacBook Pro', '{
    "brand": "Apple",
    "specs": {
        "cpu": "M3",
        "ram": "16GB",
        "storage": "512GB SSD"
    },
    "price": 2499,
    "tags": ["laptop", "professional"]
}'::jsonb),
('AirPods', '{
    "brand": "Apple",
    "specs": {
        "type": "wireless",
        "battery_life": "6 hours"
    },
    "price": 179,
    "tags": ["audio", "wireless"]
}'::jsonb);
```

### Querying JSON

```sql
-- Extract values with -> and ->>
SELECT
    name,
    details->>'brand' AS brand,
    details->'specs'->>'storage' AS storage
FROM products;

-- Query nested values
SELECT name, details->'price' AS price
FROM products
WHERE (details->>'price')::int > 200;

-- Check if JSON contains specific values
SELECT name
FROM products
WHERE details @> '{"brand": "Apple"}';

-- Query arrays in JSON
SELECT name
FROM products
WHERE details->'tags' ? 'wireless';

-- Path queries
SELECT name
FROM products
WHERE details #> '{specs,ram}' = '"16GB"';
```

### JSON Operators Cheat Sheet

```sql
-- -> : Get JSON object field (returns JSON)
-- ->> : Get JSON object field as text
-- #> : Get JSON at path (returns JSON)
-- #>> : Get JSON at path as text
-- @> : Contains
-- <@ : Is contained by
-- ? : Key exists
-- ?| : Any keys exist
-- ?& : All keys exist
```

### Modifying JSON

```sql
-- Add or update fields
UPDATE products
SET details = details || '{"warranty": "1 year"}'::jsonb
WHERE name = 'iPhone 15';

-- Update nested values
UPDATE products
SET details = jsonb_set(
    details,
    '{specs,storage}',
    '"512GB"'
)
WHERE name = 'MacBook Pro';

-- Remove fields
UPDATE products
SET details = details - 'warranty';

-- Append to arrays
UPDATE products
SET details = jsonb_set(
    details,
    '{tags}',
    details->'tags' || '["bestseller"]'::jsonb
)
WHERE name = 'iPhone 15';
```

### JSON Aggregation

```sql
-- Aggregate into JSON arrays
SELECT
    jsonb_agg(name) AS product_names
FROM products
WHERE (details->>'price')::int < 1000;

-- Build JSON objects
SELECT
    jsonb_build_object(
        'total_products', COUNT(*),
        'avg_price', AVG((details->>'price')::int),
        'brands', jsonb_agg(DISTINCT details->>'brand')
    ) AS summary
FROM products;
```

## Arrays: Lists Without Leaving SQL

PostgreSQL arrays let you store multiple values in a single column. It's like having a list in Python, but in your database.

```sql
-- Create a table with arrays
CREATE TABLE projects (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    technologies TEXT[],
    team_members VARCHAR(50)[],
    milestones DATE[]
);

-- Insert data with arrays
INSERT INTO projects (name, technologies, team_members, milestones) VALUES
('E-commerce Platform',
 ARRAY['PostgreSQL', 'React', 'Node.js', 'Redis'],
 ARRAY['Alice', 'Bob', 'Charlie'],
 ARRAY['2025-01-15'::date, '2025-03-01'::date, '2025-05-15'::date]
),
('Mobile App',
 ARRAY['PostgreSQL', 'Flutter', 'GraphQL'],
 ARRAY['Dave', 'Eve'],
 ARRAY['2025-02-01'::date, '2025-04-01'::date]
);

-- Query arrays
SELECT name
FROM projects
WHERE 'PostgreSQL' = ANY(technologies);

-- Find overlap
SELECT name
FROM projects
WHERE technologies && ARRAY['React', 'Vue.js'];  -- Has React OR Vue.js

-- Array length
SELECT name, array_length(team_members, 1) AS team_size
FROM projects;

-- Unnest arrays (flatten)
SELECT
    p.name,
    unnest(p.technologies) AS tech
FROM projects p;

-- Array aggregation
SELECT
    array_agg(DISTINCT unnest(technologies)) AS all_technologies
FROM projects;
```

## Window Functions: Analytics Without Leaving PostgreSQL

Window functions are SQL's superpowers. They calculate across rows while keeping all the detail.

```sql
-- Sample sales data
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    product VARCHAR(50),
    category VARCHAR(50),
    sale_date DATE,
    amount DECIMAL(10,2),
    quantity INTEGER
);

INSERT INTO sales (product, category, sale_date, amount, quantity) VALUES
('Laptop', 'Electronics', '2025-01-01', 1200, 1),
('Mouse', 'Electronics', '2025-01-01', 25, 2),
('Desk', 'Furniture', '2025-01-02', 350, 1),
('Laptop', 'Electronics', '2025-01-02', 1200, 1),
('Chair', 'Furniture', '2025-01-03', 200, 2),
('Monitor', 'Electronics', '2025-01-03', 400, 1);

-- Running totals
SELECT
    sale_date,
    product,
    amount,
    SUM(amount) OVER (ORDER BY sale_date) AS running_total
FROM sales
ORDER BY sale_date;

-- Ranking
SELECT
    product,
    amount,
    RANK() OVER (ORDER BY amount DESC) AS price_rank,
    DENSE_RANK() OVER (ORDER BY amount DESC) AS dense_rank,
    ROW_NUMBER() OVER (ORDER BY amount DESC) AS row_num
FROM sales;

-- Partitioned calculations
SELECT
    category,
    product,
    amount,
    AVG(amount) OVER (PARTITION BY category) AS category_avg,
    amount - AVG(amount) OVER (PARTITION BY category) AS diff_from_avg
FROM sales;

-- Lead and Lag (compare with previous/next rows)
SELECT
    sale_date,
    amount,
    LAG(amount, 1) OVER (ORDER BY sale_date) AS previous_sale,
    LEAD(amount, 1) OVER (ORDER BY sale_date) AS next_sale,
    amount - LAG(amount, 1) OVER (ORDER BY sale_date) AS change
FROM sales;

-- Percentiles
SELECT
    product,
    amount,
    PERCENT_RANK() OVER (ORDER BY amount) AS percentile,
    NTILE(4) OVER (ORDER BY amount) AS quartile
FROM sales;
```

## Full-Text Search: Google for Your Data

PostgreSQL's full-text search is powerful enough that many sites use it instead of Elasticsearch.

```sql
-- Create articles table
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    body TEXT,
    search_vector tsvector
);

-- Add some articles
INSERT INTO articles (title, body) VALUES
('PostgreSQL Full-Text Search', 'PostgreSQL provides powerful full-text search capabilities that rival dedicated search engines.'),
('Database Performance Tips', 'Improving database performance requires understanding of indexes, query planning, and caching strategies.'),
('Modern SQL Features', 'SQL has evolved beyond simple queries. Modern features include window functions, CTEs, and JSON support.');

-- Update search vectors
UPDATE articles
SET search_vector = to_tsvector('english', title || ' ' || body);

-- Create index for fast searching
CREATE INDEX idx_search ON articles USING GIN (search_vector);

-- Search queries
SELECT title
FROM articles
WHERE search_vector @@ to_tsquery('english', 'PostgreSQL & search');

-- With ranking
SELECT
    title,
    ts_rank(search_vector, query) AS rank
FROM articles,
     to_tsquery('english', 'database | performance') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- Highlight matches
SELECT
    ts_headline('english', body, query) AS highlighted
FROM articles,
     to_tsquery('english', 'PostgreSQL') AS query
WHERE search_vector @@ query;

-- Phrase search
SELECT title
FROM articles
WHERE search_vector @@ phraseto_tsquery('english', 'full text search');
```

## Common Table Expressions (CTEs): Advanced Edition

CTEs make complex queries readable and reusable.

```sql
-- Recursive CTE: Organization hierarchy
CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    manager_id INTEGER REFERENCES employees(id)
);

INSERT INTO employees VALUES
(1, 'CEO', NULL),
(2, 'CTO', 1),
(3, 'CFO', 1),
(4, 'Lead Dev', 2),
(5, 'Junior Dev', 4),
(6, 'Accountant', 3);

WITH RECURSIVE org_chart AS (
    -- Base case: top of hierarchy
    SELECT id, name, manager_id, 0 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive case
    SELECT e.id, e.name, e.manager_id, oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT
    REPEAT('  ', level) || name AS org_structure,
    level
FROM org_chart
ORDER BY level, name;

-- Multiple CTEs
WITH
category_sales AS (
    SELECT category, SUM(amount) AS total
    FROM sales
    GROUP BY category
),
category_rank AS (
    SELECT
        category,
        total,
        RANK() OVER (ORDER BY total DESC) AS rank
    FROM category_sales
)
SELECT * FROM category_rank WHERE rank <= 2;
```

## Custom Types and Domains

PostgreSQL lets you create your own types for better data modeling.

```sql
-- Create an enum type
CREATE TYPE order_status AS ENUM ('pending', 'processing', 'shipped', 'delivered', 'cancelled');

-- Create a domain (constrained type)
CREATE DOMAIN email AS VARCHAR(255)
CHECK (VALUE ~ '^[A-Za-z0-9._%-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$');

CREATE DOMAIN us_postal_code AS TEXT
CHECK (VALUE ~ '^\d{5}(-\d{4})?$');

-- Use custom types
CREATE TABLE customer_orders (
    id SERIAL PRIMARY KEY,
    customer_email email,
    status order_status DEFAULT 'pending',
    ship_to_zip us_postal_code
);

-- Composite types
CREATE TYPE address AS (
    street VARCHAR(100),
    city VARCHAR(50),
    state CHAR(2),
    zip us_postal_code
);

CREATE TABLE customers_advanced (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    billing_address address,
    shipping_address address
);

-- Insert with composite type
INSERT INTO customers_advanced (name, billing_address, shipping_address)
VALUES (
    'John Doe',
    ROW('123 Main St', 'Boston', 'MA', '02101'),
    ROW('456 Oak Ave', 'Cambridge', 'MA', '02139')
);
```

## Table Inheritance

PostgreSQL supports object-oriented concepts like inheritance.

```sql
-- Base table
CREATE TABLE vehicles (
    id SERIAL PRIMARY KEY,
    make VARCHAR(50),
    model VARCHAR(50),
    year INTEGER
);

-- Inherited tables
CREATE TABLE cars (
    doors INTEGER,
    fuel_type VARCHAR(20)
) INHERITS (vehicles);

CREATE TABLE motorcycles (
    engine_cc INTEGER
) INHERITS (vehicles);

-- Insert data
INSERT INTO cars (make, model, year, doors, fuel_type)
VALUES ('Toyota', 'Camry', 2024, 4, 'Hybrid');

INSERT INTO motorcycles (make, model, year, engine_cc)
VALUES ('Harley-Davidson', 'Street 750', 2023, 750);

-- Query parent table includes all children
SELECT * FROM vehicles;

-- Query only parent
SELECT * FROM ONLY vehicles;
```

## Triggers and Functions

Automate complex logic at the database level.

```sql
-- Create a function
CREATE OR REPLACE FUNCTION update_modified_time()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create a table with timestamps
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    content TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create trigger
CREATE TRIGGER update_posts_modtime
BEFORE UPDATE ON posts
FOR EACH ROW
EXECUTE FUNCTION update_modified_time();

-- Test it
INSERT INTO posts (title, content) VALUES ('Test', 'Content');
UPDATE posts SET title = 'Updated Test' WHERE id = 1;
SELECT * FROM posts;  -- updated_at changed automatically

-- Audit trigger
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name VARCHAR(50),
    operation VARCHAR(10),
    user_name VARCHAR(50),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    old_data JSONB,
    new_data JSONB
);

CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, operation, user_name, old_data, new_data)
    VALUES (
        TG_TABLE_NAME,
        TG_OP,
        current_user,
        to_jsonb(OLD),
        to_jsonb(NEW)
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## Materialized Views: Cached Queries

When a query is too slow to run repeatedly, materialize it.

```sql
-- Create a complex view
CREATE MATERIALIZED VIEW sales_summary AS
SELECT
    DATE_TRUNC('month', sale_date) AS month,
    category,
    COUNT(*) AS transaction_count,
    SUM(amount) AS total_sales,
    AVG(amount) AS avg_sale
FROM sales
GROUP BY DATE_TRUNC('month', sale_date), category
WITH DATA;

-- Query it like a table (fast!)
SELECT * FROM sales_summary;

-- Refresh when needed
REFRESH MATERIALIZED VIEW sales_summary;

-- Create index on materialized view
CREATE INDEX idx_sales_summary_month ON sales_summary(month);

-- Concurrent refresh (doesn't lock)
CREATE UNIQUE INDEX idx_sales_summary_unique ON sales_summary(month, category);
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary;
```

## Foreign Data Wrappers: Query Other Databases

PostgreSQL can query external data sources as if they were local tables.

```sql
-- Install extension
CREATE EXTENSION postgres_fdw;

-- Create server connection
CREATE SERVER remote_db
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'remote.example.com', dbname 'external_db', port '5432');

-- Create user mapping
CREATE USER MAPPING FOR current_user
SERVER remote_db
OPTIONS (user 'remote_user', password 'remote_password');

-- Import foreign schema
IMPORT FOREIGN SCHEMA public
FROM SERVER remote_db
INTO local_schema;

-- Now query remote tables like local ones
SELECT * FROM local_schema.remote_table;
```

## Listen/Notify: Real-time Events

PostgreSQL can send real-time notifications.

```sql
-- In session 1: Listen for events
LISTEN new_order;

-- In session 2: Send notification
NOTIFY new_order, 'Order #1234 received';

-- With triggers
CREATE OR REPLACE FUNCTION notify_new_order()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('new_order',
        json_build_object(
            'order_id', NEW.id,
            'customer', NEW.customer_id,
            'total', NEW.total_amount
        )::text
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER order_notification
AFTER INSERT ON orders
FOR EACH ROW
EXECUTE FUNCTION notify_new_order();
```

## What You've Learned

You now understand PostgreSQL's advanced features:
- JSONB for document storage with SQL power
- Arrays for list data types
- Window functions for analytics
- Full-text search that rivals Elasticsearch
- CTEs for complex, readable queries
- Custom types and domains
- Table inheritance
- Triggers for automation
- Materialized views for performance
- Foreign data wrappers for federation
- Listen/Notify for real-time events

These features aren't just checkboxes. They're production-ready, battle-tested capabilities that let PostgreSQL handle workloads that would require multiple specialized databases elsewhere.

## What's Next

Chapter 6 dives into performance optimization. We'll learn how to make PostgreSQL fast, diagnose slow queries, and tune for your specific workload.

*"PostgreSQL: Come for the SQL, stay for the everything else."*