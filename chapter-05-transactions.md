# Chapter 5: Transactions and Concurrency - ACID in Practice

## Chapter Overview

In this chapter, you'll learn:
- How transactions guarantee data consistency
- ACID properties in depth with PostgreSQL examples
- Isolation levels and their trade-offs
- Handling concurrent access patterns
- Deadlock detection and resolution
- Advanced transaction control features
- Best practices for transaction design

By the end of this chapter, you'll understand how PostgreSQL ensures data integrity even with hundreds of concurrent users modifying data simultaneously.

## Understanding Transactions

### What Is a Transaction?

A transaction is a sequence of one or more SQL operations treated as a single unit of work. Either all operations succeed, or none of them do.

```sql
-- Without transactions (DANGEROUS):
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Power failure here = money disappears!
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- With transactions (SAFE):
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- Power failure anywhere = both updates rolled back
```

### Transaction Lifecycle

```sql
BEGIN;                    -- Start transaction
-- Your SQL operations here
COMMIT;                   -- Make changes permanent
-- OR
ROLLBACK;                 -- Undo all changes
```

Every SQL statement in PostgreSQL runs in a transaction, even if you don't explicitly use BEGIN/COMMIT:

```sql
-- These are equivalent:
INSERT INTO logs (message) VALUES ('User login');

-- PostgreSQL automatically wraps it:
BEGIN;
INSERT INTO logs (message) VALUES ('User login');
COMMIT;
```

## ACID Properties in Depth

### Atomicity: All or Nothing

Atomicity ensures that partial transactions never occur.

```sql
-- Setup
CREATE TABLE inventory (
    product_id INTEGER PRIMARY KEY,
    quantity INTEGER NOT NULL CHECK (quantity >= 0)
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES inventory(product_id),
    quantity INTEGER NOT NULL
);

INSERT INTO inventory VALUES (1, 10);

-- Atomic transaction
BEGIN;
INSERT INTO orders (product_id, quantity) VALUES (1, 5);
UPDATE inventory SET quantity = quantity - 5 WHERE product_id = 1;
-- If this fails due to constraint:
UPDATE inventory SET quantity = quantity - 20 WHERE product_id = 1;
-- ERROR: new row violates check constraint
-- The entire transaction rolls back, including the INSERT
ROLLBACK;

-- Check: nothing changed
SELECT * FROM orders;     -- No new order
SELECT * FROM inventory;  -- Still 10 items
```

### Consistency: Rules Always Apply

Consistency ensures your data always follows defined rules.

```sql
-- Define consistency rules
CREATE TABLE bank_accounts (
    id SERIAL PRIMARY KEY,
    account_number VARCHAR(20) UNIQUE NOT NULL,
    balance DECIMAL(15,2) NOT NULL,
    CONSTRAINT positive_balance CHECK (balance >= 0)
);

CREATE TABLE transactions (
    id SERIAL PRIMARY KEY,
    from_account INTEGER REFERENCES bank_accounts(id),
    to_account INTEGER REFERENCES bank_accounts(id),
    amount DECIMAL(15,2) NOT NULL CHECK (amount > 0),
    created_at TIMESTAMP DEFAULT NOW(),
    CONSTRAINT different_accounts CHECK (from_account != to_account)
);

-- This transaction maintains consistency
BEGIN;
-- Check balance before transfer
SELECT balance FROM bank_accounts WHERE id = 1; -- Returns 1000

-- Attempt transfer
UPDATE bank_accounts SET balance = balance - 1500 WHERE id = 1;
-- ERROR: violates check constraint "positive_balance"
-- Transaction automatically rolled back, consistency maintained
ROLLBACK;
```

### Isolation: Concurrent Transactions Don't Interfere

Isolation determines how concurrent transactions see each other's changes.

```sql
-- Session 1
BEGIN;
SELECT balance FROM accounts WHERE id = 1; -- Returns 1000
UPDATE accounts SET balance = 800 WHERE id = 1;
-- Don't commit yet

-- Session 2 (concurrent)
BEGIN;
SELECT balance FROM accounts WHERE id = 1; -- Still returns 1000!
-- Session 2 doesn't see uncommitted changes

-- Session 1
COMMIT;

-- Session 2
SELECT balance FROM accounts WHERE id = 1; -- Now returns 800
COMMIT;
```

### Durability: Committed = Permanent

Once PostgreSQL confirms COMMIT, the data survives any failure.

```sql
BEGIN;
INSERT INTO audit_log (event, details)
VALUES ('critical_event', 'System configuration changed');
COMMIT;
-- After this point, even if the server crashes immediately,
-- this audit log entry will be there when it restarts
```

## Isolation Levels

PostgreSQL supports four isolation levels, each with different guarantees and performance characteristics.

### Read Uncommitted (Not Really in PostgreSQL)

PostgreSQL treats READ UNCOMMITTED as READ COMMITTED. It never allows dirty reads.

```sql
-- This is treated as READ COMMITTED in PostgreSQL
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

### Read Committed (Default)

Each statement sees only committed data at statement start.

```sql
-- Session 1
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT COUNT(*) FROM products; -- Returns 100

-- Session 2
BEGIN;
INSERT INTO products (name) VALUES ('New Product');
COMMIT;

-- Session 1 (still in transaction)
SELECT COUNT(*) FROM products; -- Returns 101 (sees committed data)
COMMIT;
```

**Phenomena Prevented**: Dirty reads
**Phenomena Possible**: Non-repeatable reads, phantom reads

### Repeatable Read

Transaction sees a consistent snapshot throughout.

```sql
-- Session 1
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT COUNT(*) FROM products; -- Returns 100

-- Session 2
BEGIN;
INSERT INTO products (name) VALUES ('New Product');
DELETE FROM products WHERE id = 1;
UPDATE products SET price = price * 1.1;
COMMIT;

-- Session 1 (still in transaction)
SELECT COUNT(*) FROM products; -- Still returns 100!
-- Sees consistent snapshot from transaction start
COMMIT;
```

**Phenomena Prevented**: Dirty reads, non-repeatable reads
**Phenomena Possible**: Phantom reads (though PostgreSQL prevents these too)

### Serializable

Highest isolation level - transactions execute as if serial.

```sql
-- Prevents write skew anomalies
CREATE TABLE doctors (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    on_call BOOLEAN DEFAULT FALSE
);

INSERT INTO doctors (name, on_call) VALUES
    ('Dr. Smith', TRUE),
    ('Dr. Jones', TRUE);

-- Session 1
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT COUNT(*) FROM doctors WHERE on_call = TRUE; -- Returns 2
-- Decides it's safe for Dr. Smith to go off call
UPDATE doctors SET on_call = FALSE WHERE name = 'Dr. Smith';

-- Session 2 (concurrent)
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT COUNT(*) FROM doctors WHERE on_call = TRUE; -- Returns 2
-- Decides it's safe for Dr. Jones to go off call
UPDATE doctors SET on_call = FALSE WHERE name = 'Dr. Jones';
COMMIT;

-- Session 1
COMMIT;
-- ERROR: could not serialize access due to concurrent update
-- PostgreSQL prevents both doctors from going off call!
```

### Choosing Isolation Levels

| Isolation Level | Use Case | Performance Impact |
|-----------------|----------|-------------------|
| Read Committed | Default, general use | Lowest overhead |
| Repeatable Read | Reports, consistent reads | Moderate overhead |
| Serializable | Financial transactions, critical consistency | Highest overhead |

```sql
-- Set for current transaction
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- ... your queries ...
COMMIT;

-- Set session default
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Set database default
ALTER DATABASE mydb SET DEFAULT_TRANSACTION_ISOLATION = 'repeatable read';
```

## Locking Mechanisms

### Row-Level Locks

PostgreSQL automatically acquires row locks during updates:

```sql
-- Session 1
BEGIN;
UPDATE products SET price = 29.99 WHERE id = 1;
-- Row is now locked

-- Session 2
UPDATE products SET price = 31.99 WHERE id = 1;
-- Waits for Session 1 to commit or rollback

-- Session 1
COMMIT;

-- Session 2 proceeds immediately after
```

### Explicit Locking

#### SELECT FOR UPDATE

```sql
BEGIN;
-- Lock rows for update
SELECT * FROM inventory
WHERE product_id = 1
FOR UPDATE;

-- Other transactions can read but not update these rows
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 1;
COMMIT;
```

#### SELECT FOR SHARE

```sql
BEGIN;
-- Lock rows against updates but allow other shared locks
SELECT * FROM products
WHERE category = 'electronics'
FOR SHARE;

-- Other transactions can also get shared locks but can't update
-- Useful for ensuring referenced data doesn't change
COMMIT;
```

#### NOWAIT and SKIP LOCKED

```sql
-- Don't wait for locks
BEGIN;
SELECT * FROM inventory
WHERE product_id = 1
FOR UPDATE NOWAIT;
-- ERROR immediately if locked instead of waiting

-- Skip locked rows
SELECT * FROM job_queue
WHERE status = 'pending'
FOR UPDATE SKIP LOCKED
LIMIT 1;
-- Gets next available job, skipping any locked by other workers
```

### Table-Level Locks

```sql
-- Exclusive lock (blocks everything)
BEGIN;
LOCK TABLE products IN EXCLUSIVE MODE;
-- Perform bulk updates
UPDATE products SET price = price * 1.1;
COMMIT;

-- Share lock (blocks writes, allows reads)
BEGIN;
LOCK TABLE products IN SHARE MODE;
-- Generate report while preventing changes
SELECT category, COUNT(*), AVG(price) FROM products GROUP BY category;
COMMIT;
```

## Deadlock Handling

### Understanding Deadlocks

Deadlocks occur when transactions wait for each other in a cycle:

```sql
-- Session 1
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Has lock on account 1

-- Session 2
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 2;
-- Has lock on account 2

-- Session 1
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- Waits for Session 2

-- Session 2
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
-- Waits for Session 1
-- DEADLOCK DETECTED!
-- ERROR: deadlock detected
```

### Preventing Deadlocks

1. **Always lock resources in the same order**:

```sql
-- Always lock accounts in ID order
CREATE OR REPLACE FUNCTION transfer_money(
    from_id INT,
    to_id INT,
    amount DECIMAL
) RETURNS VOID AS $$
DECLARE
    first_id INT;
    second_id INT;
BEGIN
    -- Determine lock order
    IF from_id < to_id THEN
        first_id := from_id;
        second_id := to_id;
    ELSE
        first_id := to_id;
        second_id := from_id;
    END IF;

    -- Lock in consistent order
    PERFORM * FROM accounts WHERE id = first_id FOR UPDATE;
    PERFORM * FROM accounts WHERE id = second_id FOR UPDATE;

    -- Now do the transfer
    UPDATE accounts SET balance = balance - amount WHERE id = from_id;
    UPDATE accounts SET balance = balance + amount WHERE id = to_id;
END;
$$ LANGUAGE plpgsql;
```

2. **Keep transactions short**:

```sql
-- Bad: Long transaction
BEGIN;
SELECT * FROM large_table;  -- Time consuming
-- ... lots of processing ...
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- Good: Short transaction
-- Do the SELECT outside transaction
SELECT * FROM large_table;
-- ... process data ...
-- Quick transaction for update only
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

3. **Use advisory locks for complex scenarios**:

```sql
-- Application-level locking
SELECT pg_advisory_lock(12345);  -- Get exclusive advisory lock

-- Do complex multi-table operations
-- Other sessions calling pg_advisory_lock(12345) will wait

SELECT pg_advisory_unlock(12345); -- Release lock
```

## Advanced Transaction Features

### Savepoints

Savepoints allow partial rollback within a transaction:

```sql
BEGIN;
INSERT INTO orders (customer_id, total) VALUES (1, 100);

SAVEPOINT before_items;
INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 1, 5);
INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 999, 2);
-- ERROR: foreign key violation

ROLLBACK TO before_items;
-- Only the order_items inserts are rolled back

INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 1, 3);
COMMIT;
-- Order is saved with only valid items
```

### Two-Phase Commit

For distributed transactions:

```sql
-- Prepare transaction with unique ID
BEGIN;
INSERT INTO accounts (name, balance) VALUES ('New User', 1000);
PREPARE TRANSACTION 'transfer_123';

-- Later, from any connection:
COMMIT PREPARED 'transfer_123';
-- Or rollback:
-- ROLLBACK PREPARED 'transfer_123';

-- View prepared transactions
SELECT * FROM pg_prepared_xacts;
```

### Transaction ID Wraparound

PostgreSQL uses 32-bit transaction IDs, requiring periodic maintenance:

```sql
-- Check for wraparound risk
SELECT datname, age(datfrozenxid)
FROM pg_database
ORDER BY age DESC;

-- Force aggressive vacuum if needed
VACUUM FREEZE;

-- Monitor autovacuum
SELECT relname, last_vacuum, last_autovacuum, n_dead_tup
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000;
```

## Concurrency Patterns

### Optimistic Locking

Use version numbers or timestamps to detect conflicts:

```sql
-- Add version column
ALTER TABLE products ADD COLUMN version INTEGER DEFAULT 1;

-- Update with optimistic lock check
UPDATE products
SET price = 29.99,
    version = version + 1
WHERE id = 1
  AND version = 5;  -- Expected version

-- Check if update succeeded
GET DIAGNOSTICS rows_affected = ROW_COUNT;
IF rows_affected = 0 THEN
    RAISE EXCEPTION 'Concurrent modification detected';
END IF;
```

### Pessimistic Locking

Lock resources before reading:

```sql
CREATE OR REPLACE FUNCTION get_next_job()
RETURNS jobs AS $$
DECLARE
    next_job jobs;
BEGIN
    -- Pessimistic lock - lock before processing
    SELECT * INTO next_job
    FROM jobs
    WHERE status = 'pending'
    ORDER BY priority DESC, created_at ASC
    FOR UPDATE SKIP LOCKED
    LIMIT 1;

    IF FOUND THEN
        UPDATE jobs SET status = 'processing' WHERE id = next_job.id;
    END IF;

    RETURN next_job;
END;
$$ LANGUAGE plpgsql;
```

### Queue Processing

Efficient concurrent queue processing:

```sql
-- Job queue table
CREATE TABLE job_queue (
    id SERIAL PRIMARY KEY,
    payload JSONB NOT NULL,
    status TEXT DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW(),
    started_at TIMESTAMP,
    completed_at TIMESTAMP
);

-- Worker function
CREATE OR REPLACE FUNCTION claim_job()
RETURNS job_queue AS $$
DECLARE
    job job_queue;
BEGIN
    -- Atomic claim
    UPDATE job_queue
    SET status = 'processing',
        started_at = NOW()
    WHERE id = (
        SELECT id FROM job_queue
        WHERE status = 'pending'
        ORDER BY created_at
        FOR UPDATE SKIP LOCKED
        LIMIT 1
    )
    RETURNING * INTO job;

    RETURN job;
END;
$$ LANGUAGE plpgsql;
```

## Best Practices

### 1. Keep Transactions Short

```sql
-- Bad: Long transaction holds locks
BEGIN;
SELECT * FROM large_table; -- Slow
-- Process data in application
UPDATE summary_table SET ...;
COMMIT;

-- Good: Minimize lock time
-- Read outside transaction
SELECT * FROM large_table;
-- Process data
-- Quick update transaction
BEGIN;
UPDATE summary_table SET ...;
COMMIT;
```

### 2. Handle Retry Logic

```python
# Python example with retry logic
import psycopg2
from psycopg2 import OperationalError
import time

def transfer_with_retry(conn, from_id, to_id, amount, max_retries=3):
    for attempt in range(max_retries):
        try:
            with conn.cursor() as cur:
                cur.execute("BEGIN")
                cur.execute(
                    "UPDATE accounts SET balance = balance - %s WHERE id = %s",
                    (amount, from_id)
                )
                cur.execute(
                    "UPDATE accounts SET balance = balance + %s WHERE id = %s",
                    (amount, to_id)
                )
                cur.execute("COMMIT")
                return True
        except OperationalError as e:
            if "deadlock detected" in str(e) or "could not serialize" in str(e):
                conn.rollback()
                time.sleep(0.1 * (2 ** attempt))  # Exponential backoff
                continue
            raise
    return False
```

### 3. Monitor Lock Contention

```sql
-- View current locks
SELECT
    pid,
    usename,
    pg_blocking_pids(pid) AS blocked_by,
    query,
    state
FROM pg_stat_activity
WHERE state != 'idle';

-- Find blocking queries
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));

-- Long-running transactions
SELECT
    pid,
    now() - xact_start AS duration,
    query,
    state
FROM pg_stat_activity
WHERE state != 'idle'
  AND xact_start IS NOT NULL
ORDER BY duration DESC;
```

## Exercises

### Exercise 5.1: Transaction Atomicity
Create a banking transfer function that safely moves money between accounts, handling all edge cases (insufficient funds, non-existent accounts, same account transfer).

### Exercise 5.2: Isolation Levels
Design an experiment that demonstrates the difference between READ COMMITTED and REPEATABLE READ isolation levels. Include concurrent sessions.

### Exercise 5.3: Deadlock Prevention
Create a scenario that causes deadlocks, then redesign it to prevent them using proper lock ordering.

### Exercise 5.4: Queue Implementation
Implement a job queue that multiple workers can process concurrently without processing the same job twice.

## Summary

You've learned how PostgreSQL ensures data consistency through:

✅ **Transactions**: Atomic units of work with ACID guarantees
✅ **Isolation Levels**: Control over concurrent transaction visibility
✅ **Locking**: Row and table-level concurrency control
✅ **Deadlock Handling**: Detection and prevention strategies
✅ **Advanced Features**: Savepoints, two-phase commit, advisory locks
✅ **Patterns**: Optimistic locking, pessimistic locking, queue processing

Understanding transactions and concurrency is essential for building reliable applications. PostgreSQL provides powerful tools to ensure your data remains consistent even under heavy concurrent load.

## What's Next

Chapter 6 explores PostgreSQL's advanced features that set it apart from basic relational databases: JSONB, arrays, full-text search, and more. These features let you solve complex problems without leaving PostgreSQL.

## Additional Resources

- **PostgreSQL Concurrency Control**: [postgresql.org/docs/current/mvcc.html](https://www.postgresql.org/docs/current/mvcc.html)
- **Transaction Isolation**: [postgresql.org/docs/current/transaction-iso.html](https://www.postgresql.org/docs/current/transaction-iso.html)
- **Lock Monitoring**: [wiki.postgresql.org/wiki/Lock_Monitoring](https://wiki.postgresql.org/wiki/Lock_Monitoring)

---

*"In a world of eventual consistency, PostgreSQL promises immediate consistency—and delivers."*