# Chapter 4: SQL Fundamentals - The Language of Databases

## Chapter Overview

In this chapter, you'll learn:
- How to create and modify database structures
- Core SQL query patterns and techniques
- Working with multiple tables through joins
- Aggregation and grouping for analytics
- Data modification with proper transaction handling
- Common patterns and anti-patterns
- Query optimization basics

By the end of this chapter, you'll be fluent in SQL and able to answer complex questions about your data.

## Setting Up Our Learning Environment

Let's create a realistic e-commerce database that we'll use throughout this chapter:

```sql
-- Start fresh
DROP DATABASE IF EXISTS ecommerce;
CREATE DATABASE ecommerce;
\c ecommerce

-- Enable useful extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

BEGIN;

-- Categories table
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    parent_category_id INTEGER REFERENCES categories(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Customers table
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    date_of_birth DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    CONSTRAINT valid_email CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$')
);

-- Suppliers table
CREATE TABLE suppliers (
    id SERIAL PRIMARY KEY,
    company_name VARCHAR(200) NOT NULL,
    contact_name VARCHAR(100),
    contact_email VARCHAR(255),
    phone VARCHAR(20),
    address TEXT,
    country VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Products table
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    sku VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    category_id INTEGER REFERENCES categories(id),
    supplier_id INTEGER REFERENCES suppliers(id),
    unit_price DECIMAL(10,2) NOT NULL CHECK (unit_price >= 0),
    units_in_stock INTEGER NOT NULL DEFAULT 0 CHECK (units_in_stock >= 0),
    units_on_order INTEGER NOT NULL DEFAULT 0 CHECK (units_on_order >= 0),
    reorder_level INTEGER DEFAULT 10,
    discontinued BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Orders table
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    order_number VARCHAR(20) NOT NULL UNIQUE DEFAULT 'ORD-' || LPAD(nextval('orders_id_seq')::text, 8, '0'),
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    order_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    shipped_date TIMESTAMP,
    delivery_date TIMESTAMP,
    shipping_address TEXT,
    order_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    payment_method VARCHAR(50),
    total_amount DECIMAL(10,2),
    notes TEXT,
    CONSTRAINT valid_dates CHECK (
        order_date <= shipped_date AND
        shipped_date <= delivery_date
    ),
    CONSTRAINT valid_status CHECK (
        order_status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled')
    )
);

-- Order items table (many-to-many relationship)
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id INTEGER NOT NULL REFERENCES products(id),
    unit_price DECIMAL(10,2) NOT NULL CHECK (unit_price >= 0),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    discount DECIMAL(3,2) DEFAULT 0 CHECK (discount >= 0 AND discount <= 1),
    UNIQUE(order_id, product_id)
);

-- Reviews table
CREATE TABLE reviews (
    id SERIAL PRIMARY KEY,
    product_id INTEGER NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    rating INTEGER NOT NULL CHECK (rating >= 1 AND rating <= 5),
    title VARCHAR(200),
    comment TEXT,
    helpful_count INTEGER DEFAULT 0,
    verified_purchase BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(product_id, customer_id)
);

-- Indexes for performance
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_supplier ON products(supplier_id);
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_date ON orders(order_date);
CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);
CREATE INDEX idx_reviews_product ON reviews(product_id);
CREATE INDEX idx_reviews_customer ON reviews(customer_id);

COMMIT;
```

Now let's populate it with sample data:

```sql
BEGIN;

-- Insert categories
INSERT INTO categories (name, description, parent_category_id) VALUES
('Electronics', 'Electronic devices and accessories', NULL),
('Computers', 'Desktop and laptop computers', 1),
('Mobile Devices', 'Phones and tablets', 1),
('Books', 'Physical and digital books', NULL),
('Fiction', 'Fiction books', 4),
('Non-Fiction', 'Non-fiction books', 4),
('Clothing', 'Apparel and accessories', NULL),
('Home & Garden', 'Home improvement and gardening', NULL);

-- Insert suppliers
INSERT INTO suppliers (company_name, contact_name, contact_email, phone, country) VALUES
('TechCorp Inc.', 'John Smith', 'john@techcorp.com', '555-0101', 'USA'),
('Global Books Ltd.', 'Sarah Johnson', 'sarah@globalbooks.com', '555-0102', 'UK'),
('Fashion Forward', 'Maria Garcia', 'maria@fashionforward.com', '555-0103', 'Italy'),
('Home Essentials', 'David Chen', 'david@homeessentials.com', '555-0104', 'Canada');

-- Insert customers
INSERT INTO customers (email, first_name, last_name, phone, date_of_birth) VALUES
('alice.brown@email.com', 'Alice', 'Brown', '555-1001', '1985-03-15'),
('bob.wilson@email.com', 'Bob', 'Wilson', '555-1002', '1990-07-22'),
('carol.davis@email.com', 'Carol', 'Davis', '555-1003', '1988-11-30'),
('david.miller@email.com', 'David', 'Miller', '555-1004', '1992-05-18'),
('emma.jones@email.com', 'Emma', 'Jones', '555-1005', '1995-09-08');

-- Insert products
INSERT INTO products (sku, name, description, category_id, supplier_id, unit_price, units_in_stock, reorder_level) VALUES
('LAPTOP001', 'ProBook 15', '15-inch professional laptop', 2, 1, 899.99, 25, 5),
('LAPTOP002', 'UltraBook 13', '13-inch ultralight laptop', 2, 1, 1299.99, 15, 3),
('PHONE001', 'SmartPhone X', 'Latest flagship smartphone', 3, 1, 799.99, 50, 10),
('PHONE002', 'Budget Phone A', 'Affordable smartphone', 3, 1, 299.99, 75, 15),
('BOOK001', 'The Great Novel', 'Bestselling fiction', 5, 2, 24.99, 100, 20),
('BOOK002', 'Learn SQL', 'Database programming guide', 6, 2, 39.99, 45, 10),
('BOOK003', 'Data Science 101', 'Introduction to data science', 6, 2, 44.99, 30, 8),
('SHIRT001', 'Cotton T-Shirt', 'Comfortable cotton t-shirt', 7, 3, 19.99, 200, 50),
('SHIRT002', 'Formal Shirt', 'Business formal shirt', 7, 3, 49.99, 80, 20),
('TOOL001', 'Power Drill', 'Cordless power drill', 8, 4, 129.99, 35, 10);

-- Insert orders
INSERT INTO orders (customer_id, order_date, shipped_date, order_status, payment_method) VALUES
(1, '2024-01-15 10:00:00', '2024-01-16 14:00:00', 'shipped', 'credit_card'),
(2, '2024-01-16 11:30:00', '2024-01-17 09:00:00', 'shipped', 'paypal'),
(1, '2024-01-18 14:20:00', NULL, 'confirmed', 'credit_card'),
(3, '2024-01-19 09:15:00', '2024-01-20 10:00:00', 'shipped', 'debit_card'),
(4, '2024-01-20 16:45:00', NULL, 'pending', 'credit_card');

-- Insert order items
INSERT INTO order_items (order_id, product_id, unit_price, quantity, discount) VALUES
(1, 1, 899.99, 1, 0.10),  -- Alice bought ProBook with 10% discount
(1, 5, 24.99, 2, 0),       -- Alice bought 2 novels
(2, 3, 799.99, 1, 0),      -- Bob bought SmartPhone
(2, 8, 19.99, 3, 0.15),    -- Bob bought 3 t-shirts with 15% discount
(3, 2, 1299.99, 1, 0.05),  -- Alice bought UltraBook with 5% discount
(4, 6, 39.99, 1, 0),       -- Carol bought SQL book
(4, 7, 44.99, 1, 0),       -- Carol bought Data Science book
(5, 10, 129.99, 2, 0);     -- David bought 2 power drills

-- Update order totals
UPDATE orders o
SET total_amount = (
    SELECT SUM(oi.unit_price * oi.quantity * (1 - oi.discount))
    FROM order_items oi
    WHERE oi.order_id = o.id
);

-- Insert reviews
INSERT INTO reviews (product_id, customer_id, rating, title, comment, verified_purchase) VALUES
(1, 1, 5, 'Excellent laptop!', 'Great performance and build quality', TRUE),
(3, 2, 4, 'Good phone', 'Nice features but battery could be better', TRUE),
(5, 1, 5, 'Amazing story', 'Could not put it down!', TRUE),
(6, 3, 5, 'Perfect for learning', 'Clear explanations and good examples', TRUE),
(8, 2, 3, 'Okay quality', 'Decent for the price', TRUE);

COMMIT;
```

## DDL: Defining Database Structure

### Creating Tables

```sql
-- Basic table creation
CREATE TABLE simple_table (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

-- Table with various constraints
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    employee_number VARCHAR(10) NOT NULL UNIQUE,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    hire_date DATE NOT NULL DEFAULT CURRENT_DATE,
    salary DECIMAL(10,2) CHECK (salary > 0),
    department_id INTEGER,
    manager_id INTEGER REFERENCES employees(id),
    is_active BOOLEAN DEFAULT TRUE,

    -- Table-level constraints
    CONSTRAINT valid_email CHECK (email LIKE '%@%.%'),
    CONSTRAINT hire_date_not_future CHECK (hire_date <= CURRENT_DATE),
    CONSTRAINT manager_not_self CHECK (id != manager_id)
);

-- Create table from query result
CREATE TABLE high_value_customers AS
SELECT c.*, SUM(o.total_amount) as lifetime_value
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.id
HAVING SUM(o.total_amount) > 1000;
```

### Modifying Tables

```sql
-- Add columns
ALTER TABLE products ADD COLUMN weight_kg DECIMAL(8,3);
ALTER TABLE products ADD COLUMN dimensions_cm VARCHAR(50);

-- Modify columns
ALTER TABLE products ALTER COLUMN description TYPE VARCHAR(1000);
ALTER TABLE products ALTER COLUMN units_in_stock SET DEFAULT 0;
ALTER TABLE products ALTER COLUMN sku SET NOT NULL;

-- Rename columns
ALTER TABLE products RENAME COLUMN units_in_stock TO stock_quantity;

-- Drop columns
ALTER TABLE products DROP COLUMN dimensions_cm;

-- Add constraints
ALTER TABLE products ADD CONSTRAINT positive_weight
    CHECK (weight_kg IS NULL OR weight_kg > 0);
ALTER TABLE orders ADD FOREIGN KEY (customer_id)
    REFERENCES customers(id) ON DELETE RESTRICT;

-- Drop constraints
ALTER TABLE products DROP CONSTRAINT positive_weight;

-- Rename table
ALTER TABLE high_value_customers RENAME TO vip_customers;
```

### Dropping Objects

```sql
-- Drop table (fails if referenced by foreign keys)
DROP TABLE simple_table;

-- Drop table and dependencies
DROP TABLE vip_customers CASCADE;

-- Drop if exists
DROP TABLE IF EXISTS temporary_data;

-- Drop multiple tables
DROP TABLE table1, table2, table3;
```

## DML: Querying Data

### Basic SELECT

```sql
-- Select all columns (avoid in production)
SELECT * FROM products;

-- Select specific columns
SELECT name, unit_price, stock_quantity
FROM products;

-- Column aliases
SELECT
    name AS product_name,
    unit_price AS price,
    stock_quantity AS "Stock Available"
FROM products;

-- Computed columns
SELECT
    name,
    unit_price,
    stock_quantity,
    unit_price * stock_quantity AS inventory_value
FROM products;

-- DISTINCT values
SELECT DISTINCT category_id FROM products;
SELECT DISTINCT category_id, supplier_id FROM products;

-- LIMIT and OFFSET
SELECT name, unit_price
FROM products
ORDER BY unit_price DESC
LIMIT 5;  -- Top 5 most expensive

SELECT name, unit_price
FROM products
ORDER BY unit_price DESC
LIMIT 5 OFFSET 5;  -- Next 5 most expensive
```

### WHERE Clause

```sql
-- Comparison operators
SELECT * FROM products WHERE unit_price > 50;
SELECT * FROM products WHERE stock_quantity <= reorder_level;
SELECT * FROM products WHERE discontinued = FALSE;

-- Multiple conditions
SELECT * FROM products
WHERE unit_price BETWEEN 20 AND 100
  AND stock_quantity > 0
  AND discontinued = FALSE;

-- IN operator
SELECT * FROM orders
WHERE order_status IN ('pending', 'confirmed');

SELECT * FROM products
WHERE category_id IN (
    SELECT id FROM categories WHERE name IN ('Electronics', 'Computers')
);

-- Pattern matching with LIKE
SELECT * FROM customers WHERE email LIKE '%@gmail.com';
SELECT * FROM products WHERE name LIKE 'Pro%';  -- Starts with 'Pro'
SELECT * FROM products WHERE sku LIKE '____001';  -- 4 chars + '001'

-- Case-insensitive pattern matching
SELECT * FROM products WHERE name ILIKE '%book%';

-- Regular expressions
SELECT * FROM customers
WHERE email ~ '^[a-z]+\.[a-z]+@[a-z]+\.com$';

-- NULL handling
SELECT * FROM orders WHERE shipped_date IS NULL;
SELECT * FROM orders WHERE shipped_date IS NOT NULL;
SELECT * FROM customers WHERE phone IS DISTINCT FROM NULL;
```

### Sorting Results

```sql
-- Single column sort
SELECT name, unit_price FROM products
ORDER BY unit_price DESC;

-- Multiple column sort
SELECT category_id, name, unit_price FROM products
ORDER BY category_id ASC, unit_price DESC;

-- Sort by expression
SELECT name, unit_price, stock_quantity FROM products
ORDER BY unit_price * stock_quantity DESC;

-- Sort by column position (avoid - less readable)
SELECT name, unit_price FROM products
ORDER BY 2 DESC;

-- NULL handling in sort
SELECT name, shipped_date FROM orders
ORDER BY shipped_date ASC NULLS LAST;
```

## Joins: Combining Tables

### Inner Join

```sql
-- Basic inner join
SELECT
    p.name AS product,
    c.name AS category
FROM products p
INNER JOIN categories c ON p.category_id = c.id;

-- Multiple joins
SELECT
    p.name AS product,
    c.name AS category,
    s.company_name AS supplier
FROM products p
JOIN categories c ON p.category_id = c.id
JOIN suppliers s ON p.supplier_id = s.id
WHERE p.discontinued = FALSE;

-- Join with aggregation
SELECT
    c.first_name || ' ' || c.last_name AS customer,
    COUNT(o.id) AS order_count,
    SUM(o.total_amount) AS total_spent
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.id
ORDER BY total_spent DESC;
```

### Left Join

```sql
-- Include all customers, even those without orders
SELECT
    c.first_name || ' ' || c.last_name AS customer,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.total_amount), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id
ORDER BY c.last_name;

-- Find products without reviews
SELECT
    p.name,
    COUNT(r.id) AS review_count
FROM products p
LEFT JOIN reviews r ON p.id = r.product_id
WHERE r.id IS NULL
GROUP BY p.id;
```

### Right Join

```sql
-- Less common, usually rewritten as LEFT JOIN
SELECT
    o.order_number,
    c.email
FROM orders o
RIGHT JOIN customers c ON o.customer_id = c.id;

-- Equivalent LEFT JOIN (preferred)
SELECT
    o.order_number,
    c.email
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id;
```

### Full Outer Join

```sql
-- Show all products and all categories
SELECT
    p.name AS product,
    c.name AS category
FROM products p
FULL OUTER JOIN categories c ON p.category_id = c.id
WHERE p.id IS NULL OR c.id IS NULL;  -- Find unmatched records
```

### Cross Join

```sql
-- Cartesian product (use carefully!)
SELECT
    c.name AS customer,
    p.name AS product
FROM customers c
CROSS JOIN products p
WHERE p.unit_price < 50
LIMIT 10;

-- Generate date series with cross join
SELECT
    date_series::date AS date,
    c.name AS category
FROM generate_series('2024-01-01', '2024-01-31', '1 day') date_series
CROSS JOIN categories c
WHERE c.parent_category_id IS NULL;
```

### Self Join

```sql
-- Find employees and their managers
SELECT
    e.first_name || ' ' || e.last_name AS employee,
    m.first_name || ' ' || m.last_name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- Find products in the same category
SELECT
    p1.name AS product1,
    p2.name AS product2,
    p1.category_id
FROM products p1
JOIN products p2 ON p1.category_id = p2.category_id
WHERE p1.id < p2.id  -- Avoid duplicates
  AND p1.discontinued = FALSE
  AND p2.discontinued = FALSE;
```

## Aggregation and Grouping

### Aggregate Functions

```sql
-- Count
SELECT COUNT(*) AS total_products FROM products;
SELECT COUNT(DISTINCT category_id) AS category_count FROM products;
SELECT COUNT(shipped_date) AS shipped_orders FROM orders;

-- Sum
SELECT SUM(total_amount) AS revenue FROM orders
WHERE order_date >= '2024-01-01';

-- Average
SELECT AVG(unit_price) AS avg_price FROM products;
SELECT AVG(rating) AS avg_rating FROM reviews
WHERE product_id = 1;

-- Min/Max
SELECT
    MIN(unit_price) AS cheapest,
    MAX(unit_price) AS most_expensive
FROM products
WHERE discontinued = FALSE;

-- Statistical aggregates
SELECT
    AVG(unit_price) AS mean,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY unit_price) AS median,
    STDDEV(unit_price) AS std_dev,
    VARIANCE(unit_price) AS variance
FROM products;

-- String aggregation
SELECT
    category_id,
    STRING_AGG(name, ', ' ORDER BY name) AS products
FROM products
GROUP BY category_id;

-- Array aggregation
SELECT
    customer_id,
    ARRAY_AGG(DISTINCT order_status) AS statuses
FROM orders
GROUP BY customer_id;
```

### GROUP BY

```sql
-- Basic grouping
SELECT
    category_id,
    COUNT(*) AS product_count,
    AVG(unit_price) AS avg_price
FROM products
GROUP BY category_id
ORDER BY product_count DESC;

-- Multiple grouping columns
SELECT
    category_id,
    supplier_id,
    COUNT(*) AS count,
    SUM(stock_quantity) AS total_stock
FROM products
GROUP BY category_id, supplier_id
ORDER BY category_id, supplier_id;

-- Grouping with joins
SELECT
    c.name AS category,
    s.company_name AS supplier,
    COUNT(p.id) AS product_count,
    AVG(p.unit_price) AS avg_price
FROM products p
JOIN categories c ON p.category_id = c.id
JOIN suppliers s ON p.supplier_id = s.id
GROUP BY c.id, c.name, s.id, s.company_name
ORDER BY category, supplier;

-- Grouping with expressions
SELECT
    DATE_TRUNC('month', order_date) AS month,
    COUNT(*) AS orders,
    SUM(total_amount) AS revenue
FROM orders
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month;

-- ROLLUP for subtotals
SELECT
    COALESCE(c.name, 'TOTAL') AS category,
    COALESCE(s.company_name, 'Subtotal') AS supplier,
    COUNT(p.id) AS product_count
FROM products p
JOIN categories c ON p.category_id = c.id
JOIN suppliers s ON p.supplier_id = s.id
GROUP BY ROLLUP(c.name, s.company_name)
ORDER BY c.name NULLS LAST, s.company_name NULLS LAST;
```

### HAVING Clause

```sql
-- Filter groups (not rows)
SELECT
    customer_id,
    COUNT(*) AS order_count,
    SUM(total_amount) AS total_spent
FROM orders
GROUP BY customer_id
HAVING COUNT(*) >= 2
   AND SUM(total_amount) > 500;

-- HAVING with joins
SELECT
    c.name AS category,
    COUNT(p.id) AS product_count,
    AVG(p.unit_price) AS avg_price
FROM products p
JOIN categories c ON p.category_id = c.id
GROUP BY c.id, c.name
HAVING AVG(p.unit_price) > 50
   AND COUNT(p.id) >= 2;

-- Combine WHERE and HAVING
SELECT
    category_id,
    COUNT(*) AS active_products,
    AVG(unit_price) AS avg_price
FROM products
WHERE discontinued = FALSE  -- Filter rows before grouping
GROUP BY category_id
HAVING COUNT(*) >= 2;  -- Filter groups after aggregation
```

## Subqueries

### Scalar Subqueries

```sql
-- Single value subquery
SELECT * FROM products
WHERE unit_price > (SELECT AVG(unit_price) FROM products);

-- Correlated subquery
SELECT
    p.*,
    (SELECT COUNT(*) FROM order_items oi WHERE oi.product_id = p.id) AS times_ordered
FROM products p;

-- Subquery in SELECT list
SELECT
    c.first_name || ' ' || c.last_name AS customer,
    (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) AS order_count,
    (SELECT SUM(total_amount) FROM orders o WHERE o.customer_id = c.id) AS total_spent
FROM customers c;
```

### Table Subqueries

```sql
-- IN subquery
SELECT * FROM products
WHERE category_id IN (
    SELECT id FROM categories
    WHERE name IN ('Electronics', 'Computers', 'Mobile Devices')
);

-- EXISTS subquery (often more efficient than IN)
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = c.id
      AND o.total_amount > 1000
);

-- NOT EXISTS
SELECT * FROM products p
WHERE NOT EXISTS (
    SELECT 1 FROM order_items oi
    WHERE oi.product_id = p.id
);

-- ANY/ALL operators
SELECT * FROM products
WHERE unit_price > ANY (
    SELECT unit_price FROM products
    WHERE category_id = 5
);

SELECT * FROM products
WHERE unit_price > ALL (
    SELECT unit_price FROM products
    WHERE category_id = 5
);
```

### Common Table Expressions (CTEs)

```sql
-- Simple CTE
WITH expensive_products AS (
    SELECT * FROM products
    WHERE unit_price > 100
)
SELECT * FROM expensive_products
WHERE stock_quantity < 10;

-- Multiple CTEs
WITH
category_stats AS (
    SELECT
        category_id,
        COUNT(*) AS product_count,
        AVG(unit_price) AS avg_price
    FROM products
    GROUP BY category_id
),
high_value_categories AS (
    SELECT category_id
    FROM category_stats
    WHERE avg_price > 50
)
SELECT
    p.*,
    cs.product_count,
    cs.avg_price
FROM products p
JOIN category_stats cs ON p.category_id = cs.category_id
WHERE p.category_id IN (SELECT category_id FROM high_value_categories);

-- Recursive CTE
WITH RECURSIVE category_tree AS (
    -- Base case: top-level categories
    SELECT
        id,
        name,
        parent_category_id,
        0 AS level,
        name AS path
    FROM categories
    WHERE parent_category_id IS NULL

    UNION ALL

    -- Recursive case
    SELECT
        c.id,
        c.name,
        c.parent_category_id,
        ct.level + 1,
        ct.path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_category_id = ct.id
)
SELECT * FROM category_tree
ORDER BY path;
```

## Data Modification

### INSERT

```sql
-- Single row insert
BEGIN;
INSERT INTO customers (email, first_name, last_name, phone)
VALUES ('john.doe@email.com', 'John', 'Doe', '555-1234');
COMMIT;

-- Multiple row insert
BEGIN;
INSERT INTO products (sku, name, category_id, supplier_id, unit_price, stock_quantity)
VALUES
    ('TABLET001', 'ProTab 10', 3, 1, 599.99, 30),
    ('TABLET002', 'MiniTab 7', 3, 1, 299.99, 45),
    ('BOOK004', 'PostgreSQL Mastery', 6, 2, 54.99, 25);
COMMIT;

-- Insert with RETURNING
BEGIN;
INSERT INTO customers (email, first_name, last_name)
VALUES ('jane.doe@email.com', 'Jane', 'Doe')
RETURNING id, email;
COMMIT;

-- Insert from SELECT
BEGIN;
INSERT INTO order_items (order_id, product_id, unit_price, quantity)
SELECT
    5,  -- order_id
    id,
    unit_price,
    1
FROM products
WHERE category_id = 5  -- All fiction books
  AND stock_quantity > 0;
COMMIT;

-- INSERT with conflict handling
BEGIN;
INSERT INTO customers (email, first_name, last_name)
VALUES ('john.doe@email.com', 'John', 'Doe')
ON CONFLICT (email)
DO UPDATE SET
    updated_at = CURRENT_TIMESTAMP,
    first_name = EXCLUDED.first_name,
    last_name = EXCLUDED.last_name;
COMMIT;

-- INSERT ... ON CONFLICT DO NOTHING
BEGIN;
INSERT INTO categories (name, description)
VALUES ('Electronics', 'Electronic devices')
ON CONFLICT (name) DO NOTHING;
COMMIT;
```

### UPDATE

```sql
-- Simple update
BEGIN;
UPDATE products
SET unit_price = unit_price * 1.1  -- 10% price increase
WHERE category_id = 1;
COMMIT;

-- Update multiple columns
BEGIN;
UPDATE orders
SET
    order_status = 'shipped',
    shipped_date = CURRENT_TIMESTAMP
WHERE id = 3
  AND order_status = 'confirmed';
COMMIT;

-- Update with subquery
BEGIN;
UPDATE products p
SET discontinued = TRUE
WHERE NOT EXISTS (
    SELECT 1 FROM order_items oi
    WHERE oi.product_id = p.id
      AND oi.order_id IN (
          SELECT id FROM orders
          WHERE order_date > CURRENT_DATE - INTERVAL '6 months'
      )
);
COMMIT;

-- Update with FROM clause (PostgreSQL specific)
BEGIN;
UPDATE order_items oi
SET unit_price = p.unit_price * 0.9  -- 10% discount
FROM products p
WHERE oi.product_id = p.id
  AND p.category_id = 5
  AND oi.order_id IN (
      SELECT id FROM orders
      WHERE order_status = 'pending'
  );
COMMIT;

-- Update with CTE
BEGIN;
WITH low_stock_products AS (
    SELECT id FROM products
    WHERE stock_quantity < reorder_level
      AND discontinued = FALSE
)
UPDATE products
SET units_on_order = reorder_level * 2
WHERE id IN (SELECT id FROM low_stock_products);
COMMIT;

-- Update with RETURNING
BEGIN;
UPDATE products
SET stock_quantity = stock_quantity - 1
WHERE id = 1
  AND stock_quantity > 0
RETURNING id, name, stock_quantity;
COMMIT;
```

### DELETE

```sql
-- Simple delete
BEGIN;
DELETE FROM reviews
WHERE rating < 2
  AND created_at < CURRENT_DATE - INTERVAL '1 year';
COMMIT;

-- Delete with subquery
BEGIN;
DELETE FROM customers
WHERE id NOT IN (
    SELECT DISTINCT customer_id FROM orders
)
  AND created_at < CURRENT_DATE - INTERVAL '2 years';
COMMIT;

-- Delete with USING (PostgreSQL specific)
BEGIN;
DELETE FROM order_items oi
USING orders o
WHERE oi.order_id = o.id
  AND o.order_status = 'cancelled'
  AND o.order_date < CURRENT_DATE - INTERVAL '30 days';
COMMIT;

-- Delete with RETURNING
BEGIN;
DELETE FROM products
WHERE discontinued = TRUE
  AND stock_quantity = 0
RETURNING *;
COMMIT;

-- Delete all rows (use TRUNCATE instead)
-- DELETE FROM table_name;  -- Slow, logs each row
TRUNCATE TABLE table_name;  -- Fast, minimal logging
TRUNCATE TABLE table_name CASCADE;  -- Also truncates dependent tables
```

### MERGE (UPSERT)

```sql
-- Using INSERT ... ON CONFLICT
BEGIN;
INSERT INTO products (sku, name, unit_price, stock_quantity)
VALUES ('LAPTOP001', 'ProBook 15 v2', 999.99, 30)
ON CONFLICT (sku)
DO UPDATE SET
    name = EXCLUDED.name,
    unit_price = EXCLUDED.unit_price,
    stock_quantity = products.stock_quantity + EXCLUDED.stock_quantity,
    updated_at = CURRENT_TIMESTAMP;
COMMIT;

-- Complex UPSERT with conditions
BEGIN;
INSERT INTO customer_preferences (customer_id, preference_key, preference_value)
VALUES (1, 'newsletter', 'true')
ON CONFLICT (customer_id, preference_key)
DO UPDATE SET
    preference_value = EXCLUDED.preference_value,
    updated_at = CURRENT_TIMESTAMP
WHERE customer_preferences.preference_value != EXCLUDED.preference_value;
COMMIT;
```

## Window Functions

```sql
-- Row numbering
SELECT
    ROW_NUMBER() OVER (ORDER BY unit_price DESC) AS price_rank,
    name,
    unit_price
FROM products;

-- Ranking functions
SELECT
    name,
    category_id,
    unit_price,
    RANK() OVER (PARTITION BY category_id ORDER BY unit_price DESC) AS rank_in_category,
    DENSE_RANK() OVER (ORDER BY unit_price DESC) AS overall_dense_rank,
    PERCENT_RANK() OVER (ORDER BY unit_price) AS price_percentile
FROM products;

-- Running totals and moving averages
SELECT
    order_date,
    total_amount,
    SUM(total_amount) OVER (ORDER BY order_date) AS running_total,
    AVG(total_amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3
FROM orders
ORDER BY order_date;

-- Lead and lag
SELECT
    order_date,
    total_amount,
    LAG(total_amount, 1) OVER (ORDER BY order_date) AS previous_order,
    LEAD(total_amount, 1) OVER (ORDER BY order_date) AS next_order,
    total_amount - LAG(total_amount, 1) OVER (ORDER BY order_date) AS diff_from_previous
FROM orders
ORDER BY order_date;

-- First and last value
SELECT DISTINCT
    category_id,
    FIRST_VALUE(name) OVER (
        PARTITION BY category_id
        ORDER BY unit_price DESC
    ) AS most_expensive,
    LAST_VALUE(name) OVER (
        PARTITION BY category_id
        ORDER BY unit_price DESC
        RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS least_expensive
FROM products;
```

## Query Optimization Basics

### Using EXPLAIN

```sql
-- Basic EXPLAIN
EXPLAIN SELECT * FROM products WHERE unit_price > 100;

-- EXPLAIN with execution statistics
EXPLAIN ANALYZE SELECT * FROM products WHERE unit_price > 100;

-- Verbose output
EXPLAIN (ANALYZE, VERBOSE, BUFFERS)
SELECT p.name, c.name
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE p.unit_price > 100;
```

### Optimization Techniques

```sql
-- Use indexes effectively
CREATE INDEX idx_products_price ON products(unit_price);
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- Avoid SELECT *
-- Bad
SELECT * FROM products;

-- Good
SELECT id, name, unit_price FROM products;

-- Use EXISTS instead of IN for large subqueries
-- Potentially slow
SELECT * FROM products
WHERE category_id IN (
    SELECT id FROM categories WHERE parent_category_id = 1
);

-- Often faster
SELECT * FROM products p
WHERE EXISTS (
    SELECT 1 FROM categories c
    WHERE c.id = p.category_id
      AND c.parent_category_id = 1
);

-- Push conditions into joins
-- Less efficient
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.country = 'USA'
  AND o.order_date > '2024-01-01';

-- More efficient
SELECT *
FROM orders o
JOIN (
    SELECT * FROM customers WHERE country = 'USA'
) c ON o.customer_id = c.id
WHERE o.order_date > '2024-01-01';

-- Use appropriate data types
-- Avoid implicit conversions
-- Bad: comparing string to number
SELECT * FROM products WHERE sku = 12345;

-- Good: correct data type
SELECT * FROM products WHERE sku = '12345';
```

## Common Patterns and Anti-Patterns

### Patterns

```sql
-- Pagination pattern
WITH numbered_results AS (
    SELECT
        *,
        ROW_NUMBER() OVER (ORDER BY created_at DESC) AS rn,
        COUNT(*) OVER () AS total_count
    FROM products
    WHERE discontinued = FALSE
)
SELECT * FROM numbered_results
WHERE rn BETWEEN 21 AND 30;  -- Page 3, 10 items per page

-- Soft delete pattern
ALTER TABLE products ADD COLUMN deleted_at TIMESTAMP;
CREATE INDEX idx_products_active ON products(id) WHERE deleted_at IS NULL;

-- "Delete" by marking
UPDATE products SET deleted_at = CURRENT_TIMESTAMP WHERE id = 1;

-- Query only active records
CREATE VIEW active_products AS
SELECT * FROM products WHERE deleted_at IS NULL;

-- Audit trail pattern
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name VARCHAR(50),
    record_id INTEGER,
    action VARCHAR(10),
    changed_by VARCHAR(100),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    old_values JSONB,
    new_values JSONB
);
```

### Anti-Patterns to Avoid

```sql
-- Anti-pattern: N+1 queries
-- Bad: Fetch orders, then loop to fetch items for each
-- Good: Use JOIN to fetch all at once

-- Anti-pattern: Implicit cross joins
-- Bad
SELECT * FROM products, categories WHERE products.category_id = categories.id;

-- Good
SELECT * FROM products JOIN categories ON products.category_id = categories.id;

-- Anti-pattern: Using HAVING without GROUP BY for filtering
-- Bad
SELECT * FROM products HAVING unit_price > 100;

-- Good
SELECT * FROM products WHERE unit_price > 100;

-- Anti-pattern: Relying on implicit ordering
-- Bad: Assuming order without ORDER BY
SELECT * FROM products;

-- Good: Explicit ordering
SELECT * FROM products ORDER BY id;

-- Anti-pattern: Using DISTINCT to fix bad joins
-- Bad: DISTINCT to hide duplicates from bad join
SELECT DISTINCT p.* FROM products p, order_items oi;

-- Good: Proper join condition
SELECT p.* FROM products p JOIN order_items oi ON p.id = oi.product_id;
```

## Common Table Expressions (CTEs)

CTEs make complex queries readable by breaking them into named, reusable parts.

### Basic CTE

```sql
-- Calculate order statistics in steps
WITH order_stats AS (
    SELECT
        customer_id,
        COUNT(*) as order_count,
        SUM(total_amount) as total_spent,
        AVG(total_amount) as avg_order_value
    FROM orders
    WHERE status = 'completed'
    GROUP BY customer_id
)
SELECT
    c.first_name || ' ' || c.last_name as customer_name,
    os.order_count,
    os.total_spent,
    os.avg_order_value
FROM order_stats os
JOIN customers c ON c.id = os.customer_id
WHERE os.order_count > 5
ORDER BY os.total_spent DESC;
```

### Multiple CTEs

```sql
-- Analyze customer behavior with multiple steps
WITH
customer_orders AS (
    SELECT
        customer_id,
        COUNT(*) as order_count,
        MAX(created_at) as last_order_date
    FROM orders
    GROUP BY customer_id
),
customer_categories AS (
    SELECT
        customer_id,
        CASE
            WHEN order_count >= 10 THEN 'VIP'
            WHEN order_count >= 5 THEN 'Regular'
            ELSE 'New'
        END as category
    FROM customer_orders
)
SELECT
    cc.category,
    COUNT(*) as customer_count,
    AVG(co.order_count) as avg_orders
FROM customer_categories cc
JOIN customer_orders co ON cc.customer_id = co.customer_id
GROUP BY cc.category;
```

### Recursive CTEs

```sql
-- Build organizational hierarchy
WITH RECURSIVE org_chart AS (
    -- Anchor: Start with CEO
    SELECT
        id,
        name,
        manager_id,
        0 as level,
        name as path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: Add direct reports
    SELECT
        e.id,
        e.name,
        e.manager_id,
        oc.level + 1,
        oc.path || ' → ' || e.name
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT
    repeat('  ', level) || name as org_structure,
    level
FROM org_chart
ORDER BY path;

-- Find all subcategories
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, name as full_path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    SELECT c.id, c.name, c.parent_id, ct.full_path || ' > ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY full_path;
```

## Window Functions

Window functions perform calculations across rows while keeping individual row details.

### Ranking Functions

```sql
-- Rank products by sales within each category
SELECT
    category_id,
    name,
    units_sold,
    ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY units_sold DESC) as row_num,
    RANK() OVER (PARTITION BY category_id ORDER BY units_sold DESC) as rank,
    DENSE_RANK() OVER (PARTITION BY category_id ORDER BY units_sold DESC) as dense_rank,
    PERCENT_RANK() OVER (PARTITION BY category_id ORDER BY units_sold DESC) as pct_rank
FROM products;

-- Top 3 products per category
WITH ranked_products AS (
    SELECT
        category_id,
        name,
        price,
        ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY price DESC) as rn
    FROM products
)
SELECT * FROM ranked_products WHERE rn <= 3;
```

### Aggregate Window Functions

```sql
-- Running totals and moving averages
SELECT
    order_date,
    total_amount,
    -- Running total
    SUM(total_amount) OVER (ORDER BY order_date) as running_total,
    -- 7-day moving average
    AVG(total_amount) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as moving_avg_7d,
    -- Month-to-date total
    SUM(total_amount) OVER (
        PARTITION BY DATE_TRUNC('month', order_date)
        ORDER BY order_date
    ) as month_to_date
FROM orders;

-- Compare each sale to department average
SELECT
    department_id,
    employee_name,
    sale_amount,
    AVG(sale_amount) OVER (PARTITION BY department_id) as dept_avg,
    sale_amount - AVG(sale_amount) OVER (PARTITION BY department_id) as diff_from_avg,
    sale_amount::numeric / NULLIF(AVG(sale_amount) OVER (PARTITION BY department_id), 0) as pct_of_avg
FROM sales;
```

### Lead and Lag

```sql
-- Compare with previous and next values
SELECT
    order_date,
    customer_id,
    total_amount,
    LAG(total_amount, 1) OVER (PARTITION BY customer_id ORDER BY order_date) as prev_order,
    LEAD(total_amount, 1) OVER (PARTITION BY customer_id ORDER BY order_date) as next_order,
    total_amount - LAG(total_amount, 1) OVER (PARTITION BY customer_id ORDER BY order_date) as change_from_prev
FROM orders;

-- Calculate time between orders
SELECT
    customer_id,
    order_date,
    order_date - LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) as days_since_last
FROM orders;
```

### First and Last Value

```sql
-- Show first and last order for each customer
SELECT DISTINCT
    customer_id,
    FIRST_VALUE(order_date) OVER (
        PARTITION BY customer_id
        ORDER BY order_date
        RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as first_order,
    LAST_VALUE(order_date) OVER (
        PARTITION BY customer_id
        ORDER BY order_date
        RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as last_order
FROM orders;
```

### NTILE and Distribution

```sql
-- Divide customers into quartiles by spending
SELECT
    customer_id,
    total_spent,
    NTILE(4) OVER (ORDER BY total_spent) as spending_quartile,
    CASE NTILE(4) OVER (ORDER BY total_spent)
        WHEN 1 THEN 'Bottom 25%'
        WHEN 2 THEN 'Lower Middle'
        WHEN 3 THEN 'Upper Middle'
        WHEN 4 THEN 'Top 25%'
    END as segment
FROM (
    SELECT customer_id, SUM(total_amount) as total_spent
    FROM orders
    GROUP BY customer_id
) customer_totals;

-- Cumulative distribution
SELECT
    name,
    salary,
    CUME_DIST() OVER (ORDER BY salary) as cumulative_distribution,
    ROUND(CUME_DIST() OVER (ORDER BY salary) * 100) as percentile
FROM employees;
```

## Exercises

### Exercise 4.1: Basic Queries
Write queries to:
1. Find all products priced between $20 and $50
2. List customers who haven't placed any orders
3. Show the top 5 best-selling products by quantity

### Exercise 4.2: Joins and Aggregation
1. Calculate total revenue per category
2. Find the average order value per customer
3. List products that have never been ordered

### Exercise 4.3: Complex Queries
1. Find the month with the highest revenue
2. Identify customers whose average order value is above the overall average
3. Create a report showing cumulative sales by day

### Exercise 4.4: Data Modification
1. Increase prices by 5% for products with low stock
2. Archive orders older than 2 years to an archive table
3. Implement a quantity check trigger that prevents negative stock

## Summary

You've mastered SQL fundamentals:

✅ **DDL**: Creating and modifying database structures
✅ **SELECT**: Querying data with filters, joins, and aggregations
✅ **DML**: Inserting, updating, and deleting data safely
✅ **Joins**: Combining data from multiple tables
✅ **Subqueries & CTEs**: Building complex queries incrementally
✅ **Window Functions**: Advanced analytics without grouping
✅ **Optimization**: Writing efficient queries
✅ **Patterns**: Common solutions to recurring problems

SQL is the foundation of database work. With these skills, you can extract insights from data, maintain data integrity, and build robust applications.

## What's Next

Chapter 5 explores transactions and concurrency in detail. You'll learn how PostgreSQL ensures data consistency even with hundreds of simultaneous users modifying data.

## Additional Resources

- **PostgreSQL SQL Reference**: [postgresql.org/docs/current/sql.html](https://www.postgresql.org/docs/current/sql.html)
- **SQL Style Guide**: [sqlstyle.guide](https://www.sqlstyle.guide/)
- **Window Functions Tutorial**: [postgresql.org/docs/current/tutorial-window.html](https://www.postgresql.org/docs/current/tutorial-window.html)
- **Query Performance**: [use-the-index-luke.com](https://use-the-index-luke.com/)

---

*"SQL is like chess: easy to learn the moves, a lifetime to master the game."*