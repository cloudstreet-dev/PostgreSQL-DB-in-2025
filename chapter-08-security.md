# Chapter 8: Security and Access Control - Protecting Your Data

## Chapter Overview

In this chapter, you'll learn:
- Authentication methods and configuration
- Role-based access control (RBAC)
- Row-level security (RLS)
- Column-level security
- Data encryption strategies
- SQL injection prevention
- Auditing and compliance
- Network security
- Security best practices

By the end of this chapter, you'll know how to secure PostgreSQL against threats while maintaining appropriate access for legitimate users.

## Authentication Methods

### Understanding pg_hba.conf

The `pg_hba.conf` file controls who can connect to your database and how they authenticate.

```bash
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections
local   all             postgres                                peer
local   all             all                                     scram-sha-256

# IPv4 connections
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             192.168.1.0/24          scram-sha-256

# IPv6 connections
host    all             all             ::1/128                 scram-sha-256

# SSL connections only
hostssl all             all             0.0.0.0/0               scram-sha-256

# Reject connections
host    all             all             0.0.0.0/0               reject
```

### Authentication Methods Explained

```sql
-- scram-sha-256 (recommended)
-- Most secure password-based authentication
CREATE USER secure_user WITH PASSWORD 'StrongPassword123!' USING SCRAM-SHA-256;

-- md5 (legacy, avoid if possible)
-- Less secure, but widely compatible
CREATE USER legacy_user WITH PASSWORD 'password' USING MD5;

-- Certificate authentication
-- Requires SSL certificates
-- In pg_hba.conf: hostssl all all 0.0.0.0/0 cert

-- LDAP authentication
-- In pg_hba.conf: host all all 0.0.0.0/0 ldap ldapserver=ldap.company.com

-- peer (local only)
-- Uses OS username, no password
-- In pg_hba.conf: local all all peer

-- trust (NEVER in production)
-- No authentication required
-- In pg_hba.conf: local all all trust
```

### Password Policies

```sql
-- Install password check extension
CREATE EXTENSION passwordcheck;

-- Set password encryption
SET password_encryption = 'scram-sha-256';

-- Password expiration
CREATE ROLE user_with_expiry LOGIN PASSWORD 'TempPass123!' VALID UNTIL '2024-12-31';

-- Force password change on next login
ALTER USER existing_user PASSWORD 'NewPass123!' VALID UNTIL 'now';

-- Check password age
SELECT
    rolname,
    rolpassword,
    rolvaliduntil,
    CASE
        WHEN rolvaliduntil < CURRENT_TIMESTAMP THEN 'expired'
        WHEN rolvaliduntil < CURRENT_TIMESTAMP + INTERVAL '7 days' THEN 'expiring soon'
        ELSE 'valid'
    END AS status
FROM pg_authid
WHERE rolcanlogin = true
ORDER BY rolvaliduntil NULLS LAST;
```

## Role-Based Access Control (RBAC)

### Role Hierarchy

```sql
-- Create role hierarchy
-- Superuser (avoid using)
CREATE ROLE admin_role SUPERUSER;

-- Database administrator
CREATE ROLE dba_role CREATEDB CREATEROLE;

-- Application roles
CREATE ROLE app_read_role;
CREATE ROLE app_write_role;
CREATE ROLE app_admin_role;

-- Grant permissions to roles
GRANT CONNECT ON DATABASE myapp TO app_read_role, app_write_role, app_admin_role;
GRANT USAGE ON SCHEMA public TO app_read_role, app_write_role, app_admin_role;

-- Read-only permissions
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read_role;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO app_read_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO app_read_role;

-- Read-write permissions
GRANT app_read_role TO app_write_role;  -- Inherit read permissions
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_write_role;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_write_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT INSERT, UPDATE, DELETE ON TABLES TO app_write_role;

-- Admin permissions
GRANT app_write_role TO app_admin_role;  -- Inherit read-write permissions
GRANT TRUNCATE, REFERENCES, TRIGGER ON ALL TABLES IN SCHEMA public TO app_admin_role;
GRANT CREATE ON SCHEMA public TO app_admin_role;

-- Create users and assign roles
CREATE USER alice LOGIN PASSWORD 'AlicePass123!' IN ROLE app_read_role;
CREATE USER bob LOGIN PASSWORD 'BobPass123!' IN ROLE app_write_role;
CREATE USER charlie LOGIN PASSWORD 'CharliePass123!' IN ROLE app_admin_role;
```

### Schema-Level Security

```sql
-- Create separate schemas for security
CREATE SCHEMA sensitive_data;
CREATE SCHEMA public_data;
CREATE SCHEMA audit_logs;

-- Grant schema permissions
GRANT USAGE ON SCHEMA public_data TO PUBLIC;
GRANT USAGE ON SCHEMA sensitive_data TO app_admin_role;
GRANT USAGE ON SCHEMA audit_logs TO dba_role;

-- Revoke public schema access
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON ALL TABLES IN SCHEMA public FROM PUBLIC;

-- Set search path for users
ALTER ROLE app_read_role SET search_path = public_data, public;
ALTER ROLE app_admin_role SET search_path = sensitive_data, public_data, public;
```

### Fine-Grained Permissions

```sql
-- Table-specific permissions
GRANT SELECT, INSERT ON orders TO order_processor_role;
GRANT SELECT ON customers TO customer_service_role;
GRANT UPDATE (status, shipped_date) ON orders TO shipping_role;

-- Column-specific permissions
GRANT SELECT (id, name, email) ON customers TO marketing_role;
GRANT UPDATE (email, phone) ON customers TO customer_service_role;

-- Sequence permissions
GRANT USAGE ON SEQUENCE orders_id_seq TO order_processor_role;

-- Function permissions
GRANT EXECUTE ON FUNCTION calculate_discount(DECIMAL, DECIMAL) TO sales_role;

-- Revoke specific permissions
REVOKE DELETE ON orders FROM app_write_role;
REVOKE UPDATE (salary) ON employees FROM hr_role;
```

## Row-Level Security (RLS)

### Basic RLS Implementation

```sql
-- Enable RLS on table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Create policies
-- Sales reps can only see their own orders
CREATE POLICY sales_rep_orders ON orders
    FOR ALL
    TO sales_role
    USING (sales_rep_id = current_setting('app.current_user_id')::INT);

-- Managers can see all orders in their region
CREATE POLICY manager_orders ON orders
    FOR SELECT
    TO manager_role
    USING (
        region_id IN (
            SELECT region_id FROM managers
            WHERE user_id = current_setting('app.current_user_id')::INT
        )
    );

-- Customers can only see their own orders
CREATE POLICY customer_orders ON orders
    FOR SELECT
    TO customer_role
    USING (customer_id = current_setting('app.current_user_id')::INT);

-- Admin bypass (careful!)
CREATE POLICY admin_all ON orders
    TO admin_role
    USING (true)
    WITH CHECK (true);
```

### Multi-Tenant RLS

```sql
-- Multi-tenant setup
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Create tenant isolation policy
CREATE POLICY tenant_isolation ON customers
    USING (tenant_id = current_setting('app.tenant_id')::INT);

CREATE POLICY tenant_isolation ON products
    USING (tenant_id = current_setting('app.tenant_id')::INT);

CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::INT);

-- Application sets tenant context
SET app.tenant_id = '123';
-- Now all queries automatically filter by tenant

-- Force RLS for table owners too
ALTER TABLE customers FORCE ROW LEVEL SECURITY;
```

### Time-Based RLS

```sql
-- Archive old records with time-based access
CREATE POLICY recent_records ON transactions
    FOR SELECT
    USING (
        created_at > CURRENT_DATE - INTERVAL '90 days'
        OR current_user IN ('auditor', 'admin')
    );

-- Different retention by role
CREATE POLICY data_retention ON logs
    FOR SELECT
    USING (
        CASE current_user
            WHEN 'analyst' THEN created_at > CURRENT_DATE - INTERVAL '30 days'
            WHEN 'manager' THEN created_at > CURRENT_DATE - INTERVAL '180 days'
            WHEN 'auditor' THEN true
            ELSE created_at > CURRENT_DATE - INTERVAL '7 days'
        END
    );
```

## Data Encryption

### Encryption at Rest

```bash
# File system encryption (recommended)
# Use LUKS on Linux or FileVault on macOS
# PostgreSQL data directory on encrypted volume

# Transparent Data Encryption (TDE)
# Available in EnterpriseDB and other commercial distributions
```

### Encryption in PostgreSQL

```sql
-- Install pgcrypto extension
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Symmetric encryption (same key for encrypt/decrypt)
-- Store encrypted data
CREATE TABLE sensitive_info (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    encrypted_ssn BYTEA,
    encrypted_credit_card BYTEA
);

-- Encrypt data
INSERT INTO sensitive_info (user_id, encrypted_ssn, encrypted_credit_card)
VALUES (
    1,
    pgp_sym_encrypt('123-45-6789', 'MySecretKey'),
    pgp_sym_encrypt('4111-1111-1111-1111', 'MySecretKey')
);

-- Decrypt data
SELECT
    user_id,
    pgp_sym_decrypt(encrypted_ssn, 'MySecretKey') AS ssn,
    pgp_sym_decrypt(encrypted_credit_card, 'MySecretKey') AS credit_card
FROM sensitive_info
WHERE user_id = 1;

-- Hashing for passwords
CREATE TABLE user_auth (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(100) NOT NULL
);

-- Store hashed password
INSERT INTO user_auth (username, password_hash)
VALUES ('alice', crypt('AlicePassword123!', gen_salt('bf', 8)));

-- Verify password
SELECT id, username
FROM user_auth
WHERE username = 'alice'
  AND password_hash = crypt('AlicePassword123!', password_hash);

-- One-way encryption for PII
CREATE TABLE user_profiles (
    id SERIAL PRIMARY KEY,
    email_hash VARCHAR(64) NOT NULL,
    phone_hash VARCHAR(64) NOT NULL
);

-- Store hashed PII (searchable but not reversible)
INSERT INTO user_profiles (email_hash, phone_hash)
VALUES (
    encode(digest('user@example.com', 'sha256'), 'hex'),
    encode(digest('555-0123', 'sha256'), 'hex')
);

-- Search by hash
SELECT * FROM user_profiles
WHERE email_hash = encode(digest('user@example.com', 'sha256'), 'hex');
```

### SSL/TLS Configuration

```bash
# postgresql.conf
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'root.crt'
ssl_crl_file = 'root.crl'
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'
ssl_prefer_server_ciphers = on
ssl_ecdh_curve = 'prime256v1'
ssl_min_protocol_version = 'TLSv1.2'
```

```sql
-- Require SSL for specific users
ALTER USER secure_user SET sslmode = 'require';

-- Check SSL connections
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    ssl,
    ssl_version,
    ssl_cipher,
    ssl_bits
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid)
WHERE ssl = true;
```

## SQL Injection Prevention

### Parameterized Queries

```python
# Python example - SAFE
import psycopg2

conn = psycopg2.connect("dbname=mydb")
cur = conn.cursor()

# Safe: Parameterized query
user_id = request.get('user_id')
cur.execute(
    "SELECT * FROM users WHERE id = %s",
    (user_id,)  # Parameters separately
)

# UNSAFE - Never do this!
# cur.execute(f"SELECT * FROM users WHERE id = {user_id}")

# Safe: Multiple parameters
cur.execute(
    "SELECT * FROM orders WHERE customer_id = %s AND status = %s",
    (customer_id, status)
)
```

### Stored Procedures for Security

```sql
-- Create secure procedures
CREATE OR REPLACE FUNCTION get_user_orders(p_user_id INT)
RETURNS TABLE(order_id INT, order_date TIMESTAMP, total DECIMAL)
LANGUAGE plpgsql
SECURITY DEFINER  -- Run with function owner's privileges
AS $$
BEGIN
    -- Validate input
    IF p_user_id IS NULL OR p_user_id < 1 THEN
        RAISE EXCEPTION 'Invalid user ID';
    END IF;

    -- Check permissions
    IF NOT has_table_privilege(current_user, 'orders', 'SELECT') THEN
        RAISE EXCEPTION 'Access denied';
    END IF;

    RETURN QUERY
    SELECT id, order_date, total_amount
    FROM orders
    WHERE customer_id = p_user_id;
END;
$$;

-- Grant execute permission only
REVOKE ALL ON FUNCTION get_user_orders(INT) FROM PUBLIC;
GRANT EXECUTE ON FUNCTION get_user_orders(INT) TO app_role;
```

### Input Validation

```sql
-- Create domains with validation
CREATE DOMAIN email_address AS VARCHAR(255)
CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z]{2,}$');

CREATE DOMAIN safe_string AS VARCHAR(100)
CHECK (VALUE !~ '[<>''";]');  -- Reject potentially dangerous characters

CREATE DOMAIN positive_int AS INTEGER
CHECK (VALUE > 0);

-- Use in tables
CREATE TABLE safe_users (
    id SERIAL PRIMARY KEY,
    email email_address NOT NULL,
    username safe_string NOT NULL,
    age positive_int
);

-- Function with input validation
CREATE OR REPLACE FUNCTION safe_search(search_term TEXT)
RETURNS TABLE(id INT, name TEXT)
LANGUAGE plpgsql
AS $$
BEGIN
    -- Sanitize input
    search_term := regexp_replace(search_term, '[^\w\s]', '', 'g');

    -- Limit length
    IF length(search_term) > 100 THEN
        RAISE EXCEPTION 'Search term too long';
    END IF;

    RETURN QUERY
    SELECT p.id, p.name
    FROM products p
    WHERE p.name ILIKE '%' || search_term || '%'
    LIMIT 100;  -- Prevent resource exhaustion
END;
$$;
```

## Auditing and Compliance

### Audit Logging

```sql
-- Create audit schema
CREATE SCHEMA audit;

-- Comprehensive audit table
CREATE TABLE audit.activity_log (
    id BIGSERIAL PRIMARY KEY,
    event_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    user_name VARCHAR(100) DEFAULT current_user,
    client_addr INET DEFAULT inet_client_addr(),
    session_id VARCHAR(50) DEFAULT pg_backend_pid()::TEXT,
    action VARCHAR(20) NOT NULL,
    table_name VARCHAR(100),
    record_id TEXT,
    old_values JSONB,
    new_values JSONB,
    query TEXT,
    success BOOLEAN DEFAULT true,
    error_message TEXT
);

-- Create audit trigger function
CREATE OR REPLACE FUNCTION audit.log_activity()
RETURNS TRIGGER AS $$
DECLARE
    audit_row audit.activity_log;
BEGIN
    audit_row.action := TG_OP;
    audit_row.table_name := TG_TABLE_SCHEMA || '.' || TG_TABLE_NAME;

    IF TG_OP = 'DELETE' THEN
        audit_row.old_values := to_jsonb(OLD);
        audit_row.record_id := OLD.id::TEXT;
    ELSIF TG_OP = 'UPDATE' THEN
        audit_row.old_values := to_jsonb(OLD);
        audit_row.new_values := to_jsonb(NEW);
        audit_row.record_id := NEW.id::TEXT;
    ELSIF TG_OP = 'INSERT' THEN
        audit_row.new_values := to_jsonb(NEW);
        audit_row.record_id := NEW.id::TEXT;
    END IF;

    INSERT INTO audit.activity_log
    (action, table_name, record_id, old_values, new_values)
    VALUES
    (audit_row.action, audit_row.table_name, audit_row.record_id,
     audit_row.old_values, audit_row.new_values);

    RETURN NULL;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Apply audit triggers
CREATE TRIGGER audit_customers
AFTER INSERT OR UPDATE OR DELETE ON customers
FOR EACH ROW EXECUTE FUNCTION audit.log_activity();

CREATE TRIGGER audit_orders
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION audit.log_activity();

-- Log failed login attempts
CREATE OR REPLACE FUNCTION audit.log_login_attempt(
    p_username VARCHAR,
    p_success BOOLEAN,
    p_error TEXT DEFAULT NULL
)
RETURNS VOID AS $$
BEGIN
    INSERT INTO audit.activity_log (action, user_name, success, error_message)
    VALUES ('LOGIN', p_username, p_success, p_error);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### Compliance Reporting

```sql
-- GDPR compliance: Right to be forgotten
CREATE OR REPLACE PROCEDURE anonymize_user(p_user_id INT)
LANGUAGE plpgsql
AS $$
BEGIN
    -- Log the anonymization
    INSERT INTO audit.activity_log (action, table_name, record_id)
    VALUES ('ANONYMIZE', 'users', p_user_id::TEXT);

    -- Anonymize personal data
    UPDATE users
    SET
        email = 'deleted-' || id || '@example.com',
        first_name = 'DELETED',
        last_name = 'USER',
        phone = NULL,
        date_of_birth = NULL,
        address = NULL,
        anonymized_at = CURRENT_TIMESTAMP
    WHERE id = p_user_id;

    -- Remove from other tables or anonymize related records
    DELETE FROM user_sessions WHERE user_id = p_user_id;
    DELETE FROM user_preferences WHERE user_id = p_user_id;

    COMMIT;
END;
$$;

-- PCI compliance: Credit card data masking
CREATE VIEW masked_payments AS
SELECT
    id,
    user_id,
    CASE
        WHEN current_user IN ('admin', 'auditor') THEN credit_card_number
        ELSE 'XXXX-XXXX-XXXX-' || RIGHT(credit_card_number, 4)
    END AS credit_card_number,
    expiry_date,
    amount
FROM payments;

-- SOX compliance: Segregation of duties
CREATE OR REPLACE FUNCTION check_segregation_of_duties()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.approved_by = NEW.created_by THEN
        RAISE EXCEPTION 'Segregation of duties violation: same person cannot create and approve';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER enforce_sod
BEFORE INSERT OR UPDATE ON financial_transactions
FOR EACH ROW EXECUTE FUNCTION check_segregation_of_duties();
```

## Network Security

### Connection Limits

```sql
-- Set connection limits per user
ALTER USER web_app CONNECTION LIMIT 20;
ALTER USER reporting_tool CONNECTION LIMIT 5;

-- Set connection limits per database
ALTER DATABASE production CONNECTION LIMIT 100;

-- Monitor connections
SELECT
    usename,
    count(*) as connection_count,
    max(backend_start) as latest_connection
FROM pg_stat_activity
GROUP BY usename
ORDER BY connection_count DESC;

-- Terminate idle connections
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND state_change < CURRENT_TIMESTAMP - INTERVAL '30 minutes';
```

### Firewall Rules

```bash
# PostgreSQL listen configuration
# postgresql.conf
listen_addresses = 'localhost,192.168.1.10'  # Specific IPs only
port = 5432

# iptables rules
# Allow only specific IPs
iptables -A INPUT -p tcp --dport 5432 -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 5432 -j DROP

# UFW (Ubuntu)
ufw allow from 192.168.1.0/24 to any port 5432
ufw deny 5432
```

### VPN and SSH Tunneling

```bash
# SSH tunnel for secure remote access
ssh -L 5433:localhost:5432 user@database-server.com

# Connect through tunnel
psql -h localhost -p 5433 -U dbuser -d mydb

# PgBouncer with SSL
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_addr = *
listen_port = 6432
server_tls_sslmode = require
server_tls_ca_file = /path/to/ca.crt
client_tls_sslmode = require
client_tls_cert_file = /path/to/server.crt
client_tls_key_file = /path/to/server.key
```

## Security Best Practices

### Regular Security Tasks

```sql
-- 1. Review user permissions
CREATE VIEW security_review AS
SELECT
    r.rolname,
    r.rolsuper,
    r.rolinherit,
    r.rolcreaterole,
    r.rolcreatedb,
    r.rolcanlogin,
    r.rolconnlimit,
    r.rolvaliduntil,
    ARRAY(
        SELECT b.rolname
        FROM pg_catalog.pg_auth_members m
        JOIN pg_catalog.pg_roles b ON (m.roleid = b.oid)
        WHERE m.member = r.oid
    ) AS memberof
FROM pg_catalog.pg_roles r
ORDER BY r.rolname;

-- 2. Check for default passwords
-- Never store passwords in plain text!
-- This is just for demonstration
CREATE TABLE password_blacklist (
    password_hash VARCHAR(100)
);

INSERT INTO password_blacklist VALUES
    (crypt('password', gen_salt('bf', 8))),
    (crypt('123456', gen_salt('bf', 8))),
    (crypt('admin', gen_salt('bf', 8)));

-- 3. Find unused privileges
SELECT
    nsp.nspname as schema,
    cls.relname as table,
    rol.rolname as role,
    has_table_privilege(rol.oid, cls.oid, 'SELECT') as select,
    has_table_privilege(rol.oid, cls.oid, 'INSERT') as insert,
    has_table_privilege(rol.oid, cls.oid, 'UPDATE') as update,
    has_table_privilege(rol.oid, cls.oid, 'DELETE') as delete
FROM pg_class cls
JOIN pg_namespace nsp ON cls.relnamespace = nsp.oid
CROSS JOIN pg_roles rol
WHERE cls.relkind = 'r'
  AND nsp.nspname NOT IN ('pg_catalog', 'information_schema')
  AND rol.rolcanlogin = true
  AND (
    has_table_privilege(rol.oid, cls.oid, 'SELECT') OR
    has_table_privilege(rol.oid, cls.oid, 'INSERT') OR
    has_table_privilege(rol.oid, cls.oid, 'UPDATE') OR
    has_table_privilege(rol.oid, cls.oid, 'DELETE')
  )
ORDER BY schema, table, role;
```

### Security Checklist

```sql
-- Security audit checklist procedure
CREATE OR REPLACE FUNCTION security_audit()
RETURNS TABLE(
    check_name TEXT,
    status TEXT,
    details TEXT
) AS $$
BEGIN
    -- Check 1: Superuser accounts
    RETURN QUERY
    SELECT
        'Superuser accounts'::TEXT,
        CASE WHEN COUNT(*) > 1 THEN 'WARNING' ELSE 'OK' END,
        'Count: ' || COUNT(*)::TEXT
    FROM pg_roles WHERE rolsuper = true;

    -- Check 2: Users with no password expiry
    RETURN QUERY
    SELECT
        'Users without password expiry'::TEXT,
        CASE WHEN COUNT(*) > 0 THEN 'WARNING' ELSE 'OK' END,
        'Count: ' || COUNT(*)::TEXT
    FROM pg_roles
    WHERE rolcanlogin = true
      AND rolvaliduntil IS NULL;

    -- Check 3: Databases allowing public access
    RETURN QUERY
    SELECT
        'Databases with public access'::TEXT,
        CASE WHEN COUNT(*) > 0 THEN 'WARNING' ELSE 'OK' END,
        'Count: ' || COUNT(*)::TEXT
    FROM pg_database
    WHERE has_database_privilege('public', oid, 'CONNECT');

    -- Check 4: SSL status
    RETURN QUERY
    SELECT
        'SSL enabled'::TEXT,
        CASE WHEN setting = 'on' THEN 'OK' ELSE 'CRITICAL' END,
        'SSL is ' || setting
    FROM pg_settings WHERE name = 'ssl';

    -- Check 5: Password encryption
    RETURN QUERY
    SELECT
        'Password encryption'::TEXT,
        CASE WHEN setting = 'scram-sha-256' THEN 'OK' ELSE 'WARNING' END,
        'Method: ' || setting
    FROM pg_settings WHERE name = 'password_encryption';

    -- Check 6: Logging enabled
    RETURN QUERY
    SELECT
        'Connection logging'::TEXT,
        CASE WHEN setting = 'on' THEN 'OK' ELSE 'WARNING' END,
        'log_connections is ' || setting
    FROM pg_settings WHERE name = 'log_connections';

END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Run security audit
SELECT * FROM security_audit();
```

## Exercises

### Exercise 8.1: Implement RBAC
1. Design a role hierarchy for a hospital system
2. Implement appropriate permissions for doctors, nurses, and administrators
3. Test access controls with different user scenarios

### Exercise 8.2: Row-Level Security
1. Create a multi-tenant application with RLS
2. Implement policies for tenant isolation
3. Verify no data leakage between tenants

### Exercise 8.3: Audit System
1. Build a comprehensive audit system
2. Track all data modifications
3. Create compliance reports

### Exercise 8.4: Security Hardening
1. Run a security audit on your database
2. Fix all identified vulnerabilities
3. Implement monitoring for security events

## Summary

You've learned comprehensive PostgreSQL security:

✅ **Authentication**: Multiple methods and proper configuration
✅ **RBAC**: Role-based access control implementation
✅ **RLS**: Row-level security for fine-grained access
✅ **Encryption**: At-rest and in-transit data protection
✅ **SQL Injection**: Prevention techniques and best practices
✅ **Auditing**: Compliance and activity tracking
✅ **Network Security**: Connection management and firewall rules
✅ **Best Practices**: Regular security tasks and checklists

Security is not a one-time setup but an ongoing process. Regular audits, updates, and monitoring are essential to maintain a secure database environment.

## What's Next

Chapter 9 covers high availability and replication, ensuring your secure database is also highly available and resilient to failures.

## Additional Resources

- **PostgreSQL Security Documentation**: [postgresql.org/docs/current/security.html](https://www.postgresql.org/docs/current/security.html)
- **OWASP Database Security**: [owasp.org/www-project-database-security/](https://owasp.org/www-project-database-security/)
- **CIS PostgreSQL Benchmark**: [cisecurity.org/benchmark/postgresql](https://www.cisecurity.org/benchmark/postgresql)
- **PostgreSQL Audit Extension**: [pgaudit.org](https://www.pgaudit.org/)

---

*"Security is not a product, but a process. In PostgreSQL, it's a well-documented, thoroughly tested process."*