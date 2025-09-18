# Chapter 3: Installation and Setup - Getting Started Right

## Chapter Overview

In this chapter, you'll learn:
- How to install PostgreSQL on any platform
- Initial configuration for development and production
- Connection methods and tools
- Database cluster initialization
- User and database creation
- Verification and testing procedures
- Common installation issues and solutions

By the end of this chapter, you'll have a working PostgreSQL installation configured properly for your environment.

## Understanding PostgreSQL Architecture

Before installing, let's understand what we're setting up:

### Components

```
┌─────────────────────────────────────┐
│          Client Applications         │
│    (psql, pgAdmin, your app)        │
└─────────────┬───────────────────────┘
              │ Port 5432 (default)
┌─────────────▼───────────────────────┐
│       PostgreSQL Server Process      │
│          (postmaster)                │
├─────────────────────────────────────┤
│   Background Processes               │
│   - Writer (writes dirty buffers)   │
│   - Checkpointer (creates checkpts) │
│   - Autovacuum (maintenance)        │
│   - Stats Collector (statistics)    │
│   - WAL Writer (write-ahead log)    │
├─────────────────────────────────────┤
│         Shared Memory                │
│   - Shared Buffers                  │
│   - WAL Buffers                     │
│   - Lock Tables                     │
├─────────────────────────────────────┤
│         Data Directory               │
│   - Database Files                  │
│   - WAL Files                       │
│   - Configuration Files             │
└─────────────────────────────────────┘
```

### Key Concepts

- **Cluster**: A collection of databases managed by a single PostgreSQL server
- **Database**: Isolated collection of schemas, tables, and other objects
- **Schema**: Namespace within a database (default: `public`)
- **Data Directory**: File system location storing all data (`PGDATA`)

## Platform-Specific Installation

### macOS Installation

#### Option 1: Homebrew (Recommended)

```bash
# Install Homebrew if you haven't
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install PostgreSQL
brew install postgresql@16

# Start PostgreSQL
brew services start postgresql@16

# Verify installation
psql --version
# PostgreSQL 16.x

# Create your user database
createdb `whoami`

# Connect
psql
```

#### Option 2: Postgres.app (GUI Approach)

```bash
# Download from https://postgresapp.com/
# Drag to Applications
# Launch Postgres.app
# Click "Initialize"

# Add to PATH in ~/.zshrc or ~/.bash_profile
echo 'export PATH="/Applications/Postgres.app/Contents/Versions/latest/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# Verify
psql --version
```

#### Option 3: Official Installer

```bash
# Download from https://www.postgresql.org/download/macosx/
# Run the installer
# Follow GUI prompts
# Note: Installs to /Library/PostgreSQL/16/
```

### Linux Installation

#### Ubuntu/Debian

```bash
# Add PostgreSQL official repository
sudo sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# Add repository key
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Update package list
sudo apt update

# Install PostgreSQL
sudo apt install postgresql-16 postgresql-contrib-16

# Verify service is running
sudo systemctl status postgresql

# Switch to postgres user
sudo -u postgres psql
```

#### RHEL/CentOS/Fedora

```bash
# Install repository RPM
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Disable built-in PostgreSQL module
sudo dnf -qy module disable postgresql

# Install PostgreSQL
sudo dnf install -y postgresql16-server postgresql16

# Initialize database
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb

# Enable and start
sudo systemctl enable postgresql-16
sudo systemctl start postgresql-16

# Verify
sudo -u postgres psql
```

#### Arch Linux

```bash
# Install PostgreSQL
sudo pacman -S postgresql

# Initialize database cluster
sudo -u postgres initdb -D /var/lib/postgres/data

# Start and enable service
sudo systemctl enable postgresql
sudo systemctl start postgresql

# Create your user
sudo -u postgres createuser --interactive
```

### Windows Installation

#### Option 1: Official Installer

1. Download installer from [postgresql.org/download/windows/](https://www.postgresql.org/download/windows/)
2. Run as Administrator
3. Installation steps:
   ```
   Installation Directory: C:\Program Files\PostgreSQL\16
   Data Directory: C:\Program Files\PostgreSQL\16\data
   Password: [Choose strong password for postgres user]
   Port: 5432 (default)
   Locale: Default
   ```
4. Optionally install Stack Builder for extensions

#### Option 2: Chocolatey

```powershell
# Install Chocolatey if needed
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Install PostgreSQL
choco install postgresql16

# Set postgres user password
psql -U postgres -c "ALTER USER postgres PASSWORD 'your_password';"
```

#### Verify Windows Installation

```powershell
# Check version
psql --version

# Connect as postgres user
psql -U postgres

# Create your user
createuser -U postgres -d -e your_username

# Create your database
createdb -U postgres -O your_username your_database
```

### Docker Installation

#### Single Container

```bash
# Run PostgreSQL container
docker run --name postgres16 \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  -d postgres:16

# Connect to container
docker exec -it postgres16 psql -U myuser -d mydb

# Or connect from host
psql -h localhost -U myuser -d mydb
```

#### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:16
    container_name: postgres_dev
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-postgres}
      POSTGRES_DB: ${POSTGRES_DB:-dev_db}
    ports:
      - "${POSTGRES_PORT:-5432}:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

```bash
# Start services
docker-compose up -d

# View logs
docker-compose logs -f postgres

# Connect
docker-compose exec postgres psql -U postgres
```

## Initial Configuration

### Locating Configuration Files

```sql
-- Find configuration file locations
SHOW config_file;    -- postgresql.conf
SHOW hba_file;       -- pg_hba.conf
SHOW ident_file;     -- pg_ident.conf
SHOW data_directory; -- Data directory location
```

### Essential postgresql.conf Settings

```bash
# Development settings
# Edit: /usr/local/var/postgresql@16/postgresql.conf (macOS Homebrew)
# Or: /etc/postgresql/16/main/postgresql.conf (Linux)

# Connection Settings
listen_addresses = 'localhost'     # Only local connections
port = 5432
max_connections = 100              # Adjust based on needs

# Memory Settings (Development - 8GB RAM machine)
shared_buffers = 2GB               # 25% of RAM
work_mem = 8MB                     # Per operation
maintenance_work_mem = 512MB      # For VACUUM, CREATE INDEX

# Logging
log_destination = 'stderr'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB
log_line_prefix = '%t [%p]: user=%u,db=%d '
log_statement = 'all'              # Development only!
log_duration = on
log_min_duration_statement = 100  # Log queries > 100ms

# Performance
random_page_cost = 1.1             # SSD optimization
effective_io_concurrency = 200     # SSD optimization
```

```bash
# Production settings (adjust based on hardware)
# 32GB RAM server example

shared_buffers = 8GB
work_mem = 32MB
maintenance_work_mem = 2GB
effective_cache_size = 24GB
max_connections = 200
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
log_statement = 'ddl'              # Only log schema changes
log_min_duration_statement = 1000  # Only slow queries
```

### Authentication Configuration (pg_hba.conf)

```bash
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections
local   all             all                                     peer    # Linux
local   all             all                                     scram-sha-256  # macOS

# IPv4 local connections
host    all             all             127.0.0.1/32            scram-sha-256

# IPv6 local connections
host    all             all             ::1/128                 scram-sha-256

# Allow connections from local network (development)
host    all             all             192.168.1.0/24          scram-sha-256

# Specific database/user combination
host    myapp_prod      myapp_user      10.0.0.0/8             scram-sha-256

# Reject all others
host    all             all             0.0.0.0/0               reject
```

### Apply Configuration Changes

```bash
# Reload configuration (no downtime)
sudo systemctl reload postgresql

# Or from SQL
SELECT pg_reload_conf();

# Some settings require restart
sudo systemctl restart postgresql

# Check current settings
psql -c "SHOW ALL;"
```

## Creating Users and Databases

### User Management

```sql
-- Create a superuser
CREATE USER admin_user WITH SUPERUSER PASSWORD 'strong_password';

-- Create application user
CREATE USER app_user WITH
    LOGIN
    PASSWORD 'app_password'
    CREATEDB
    CONNECTION LIMIT 10;

-- Create read-only user
CREATE USER readonly_user WITH
    LOGIN
    PASSWORD 'readonly_password'
    CONNECTION LIMIT 5;

-- Grant privileges
GRANT CONNECT ON DATABASE myapp TO readonly_user;
GRANT USAGE ON SCHEMA public TO readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;

-- Alter existing user
ALTER USER app_user WITH PASSWORD 'new_password';
ALTER USER app_user VALID UNTIL '2025-01-01';

-- View users
\du
-- Or
SELECT usename, usesuper, usecreatedb, usecreatRole
FROM pg_user;
```

### Database Management

```sql
-- Create database with specific options
CREATE DATABASE myapp
    WITH
    OWNER = app_user
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.utf8'
    LC_CTYPE = 'en_US.utf8'
    TABLESPACE = pg_default
    CONNECTION LIMIT = 100;

-- Create from template
CREATE DATABASE myapp_test
    WITH
    TEMPLATE = myapp
    OWNER = app_user;

-- Change owner
ALTER DATABASE myapp OWNER TO new_owner;

-- Set default configuration for database
ALTER DATABASE myapp SET timezone TO 'UTC';
ALTER DATABASE myapp SET statement_timeout TO '30s';

-- View databases
\l
-- Or
SELECT datname, pg_size_pretty(pg_database_size(datname)) as size
FROM pg_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY pg_database_size(datname) DESC;
```

## Connection Methods and Tools

### Connection Strings

```bash
# Format
postgresql://[user[:password]@][host][:port][/dbname][?param1=value1&...]

# Examples
postgresql://localhost/mydb
postgresql://user:secret@localhost
postgresql://other@localhost/otherdb?connect_timeout=10&application_name=myapp

# Environment variables
export PGHOST=localhost
export PGPORT=5432
export PGDATABASE=mydb
export PGUSER=myuser
export PGPASSWORD=mypassword  # Not recommended

# .pgpass file (secure alternative)
# ~/.pgpass (chmod 600)
localhost:5432:mydb:myuser:mypassword
```

### Command Line Tools

```bash
# psql - Interactive terminal
psql -h localhost -U myuser -d mydb
psql "postgresql://user:pass@localhost:5432/mydb"

# Common psql commands
\?              # Help
\l              # List databases
\c dbname       # Connect to database
\dt             # List tables
\d tablename    # Describe table
\du             # List users
\q              # Quit
\i script.sql   # Execute script
\o output.txt   # Send output to file
\timing on      # Show query timing

# Non-interactive execution
psql -c "SELECT version();"
psql -f script.sql
echo "SELECT * FROM users;" | psql

# Other utilities
createdb mydb
dropdb mydb
createuser myuser
dropuser myuser
pg_dump mydb > backup.sql
pg_restore -d mydb backup.dump
```

### GUI Tools

#### pgAdmin 4
```bash
# Install on macOS
brew install --cask pgadmin4

# Install on Linux
curl -fsS https://www.pgadmin.org/static/packages_pgadmin_org.pub | sudo gpg --dearmor -o /usr/share/keyrings/packages-pgadmin-org.gpg
sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/packages-pgadmin-org.gpg] https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/$(lsb_release -cs) pgadmin4 main" > /etc/apt/sources.list.d/pgadmin4.list'
sudo apt update
sudo apt install pgadmin4
```

#### Other Popular GUI Tools
- **DBeaver**: Free, multi-database support
- **TablePlus**: Modern, clean interface (paid)
- **DataGrip**: JetBrains IDE (paid)
- **Postico**: macOS native (freemium)
- **Beekeeper Studio**: Open source, cross-platform

### Programming Language Connections

#### Python (psycopg2)
```python
import psycopg2
from psycopg2.extras import RealDictCursor

# Connect
conn = psycopg2.connect(
    host="localhost",
    database="mydb",
    user="myuser",
    password="mypassword",
    cursor_factory=RealDictCursor
)

# Use connection
with conn.cursor() as cur:
    cur.execute("SELECT * FROM users WHERE active = %s", (True,))
    users = cur.fetchall()

conn.close()
```

#### Node.js (pg)
```javascript
const { Pool } = require('pg');

const pool = new Pool({
  host: 'localhost',
  database: 'mydb',
  user: 'myuser',
  password: 'mypassword',
  port: 5432,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Query
const result = await pool.query('SELECT * FROM users WHERE id = $1', [userId]);

// Clean shutdown
await pool.end();
```

#### Java (JDBC)
```java
import java.sql.*;

String url = "jdbc:postgresql://localhost:5432/mydb";
Properties props = new Properties();
props.setProperty("user", "myuser");
props.setProperty("password", "mypassword");
props.setProperty("ssl", "false");

try (Connection conn = DriverManager.getConnection(url, props)) {
    PreparedStatement pst = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
    pst.setInt(1, userId);
    ResultSet rs = pst.executeQuery();
}
```

## Verification and Testing

### Basic Health Checks

```bash
# Check if PostgreSQL is running
pg_isready
# Output: /tmp:5432 - accepting connections

# Check version
psql -V
postgres -V

# System status
sudo systemctl status postgresql

# Test connection
psql -c "SELECT 1;"
```

### Performance Baseline

```sql
-- Create test database
CREATE DATABASE benchmark;
\c benchmark

-- Initialize pgbench
pgbench -i -s 10 benchmark

-- Run benchmark
pgbench -c 10 -T 60 benchmark

-- Results interpretation:
-- tps = transactions per second (higher is better)
-- latency = response time (lower is better)
```

### Verification Script

```bash
#!/bin/bash
# verify_postgres.sh

echo "PostgreSQL Installation Verification"
echo "====================================="

# Check if PostgreSQL is installed
if ! command -v psql &> /dev/null; then
    echo "❌ PostgreSQL is not installed or not in PATH"
    exit 1
fi
echo "✅ PostgreSQL is installed"

# Check version
VERSION=$(psql --version | awk '{print $3}')
echo "✅ Version: $VERSION"

# Check if service is running
if pg_isready &> /dev/null; then
    echo "✅ PostgreSQL service is running"
else
    echo "❌ PostgreSQL service is not running"
    exit 1
fi

# Test connection
if psql -U postgres -c "SELECT 1;" &> /dev/null; then
    echo "✅ Can connect to PostgreSQL"
else
    echo "⚠️  Cannot connect as postgres user (may need password)"
fi

# Check data directory
DATA_DIR=$(psql -U postgres -t -c "SHOW data_directory;" 2>/dev/null | xargs)
if [ ! -z "$DATA_DIR" ]; then
    echo "✅ Data directory: $DATA_DIR"
fi

# Check configuration
CONFIG_FILE=$(psql -U postgres -t -c "SHOW config_file;" 2>/dev/null | xargs)
if [ ! -z "$CONFIG_FILE" ]; then
    echo "✅ Config file: $CONFIG_FILE"
fi

echo "====================================="
echo "Installation verified successfully!"
```

## Troubleshooting Common Issues

### Connection Refused

```bash
# Error: could not connect to server: Connection refused
# Is the server running on host "localhost" and accepting TCP/IP connections on port 5432?

# Solution 1: Start PostgreSQL
sudo systemctl start postgresql  # Linux
brew services start postgresql@16  # macOS
pg_ctl start -D /usr/local/var/postgresql@16  # Manual

# Solution 2: Check if running on different port
netstat -an | grep 5432
lsof -i :5432

# Solution 3: Check postgresql.conf
# Ensure: listen_addresses = '*' or 'localhost'
```

### Authentication Failed

```bash
# Error: FATAL: password authentication failed for user "myuser"

# Solution 1: Reset password
sudo -u postgres psql
ALTER USER myuser PASSWORD 'new_password';

# Solution 2: Check pg_hba.conf
# Change 'md5' to 'trust' temporarily for local connections
# Then reset password and change back

# Solution 3: Create .pgpass file
echo "localhost:5432:*:myuser:mypassword" >> ~/.pgpass
chmod 600 ~/.pgpass
```

### Permission Denied

```bash
# Error: FATAL: permission denied for database "mydb"

# Solution: Grant permissions
sudo -u postgres psql
GRANT ALL PRIVILEGES ON DATABASE mydb TO myuser;
GRANT ALL ON SCHEMA public TO myuser;
```

### Disk Space Issues

```sql
-- Check database sizes
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) as size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

-- Check table sizes
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;

-- Clean up
VACUUM FULL;  -- Reclaim space
```

### Port Already in Use

```bash
# Error: could not bind IPv4 address "127.0.0.1": Address already in use

# Find process using port
sudo lsof -i :5432
sudo netstat -tulpn | grep 5432

# Kill process (carefully!)
sudo kill -9 <PID>

# Or change PostgreSQL port in postgresql.conf
port = 5433
```

## Security Hardening

### Post-Installation Security Steps

```sql
-- 1. Change default postgres password
ALTER USER postgres PASSWORD 'VeryStrongPassword123!';

-- 2. Remove unnecessary databases
DROP DATABASE IF EXISTS template1;  -- Usually keep this
-- DROP DATABASE template0;  -- Never drop this

-- 3. Revoke PUBLIC schema access
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE postgres FROM PUBLIC;

-- 4. Create separate schemas for applications
CREATE SCHEMA myapp;
GRANT USAGE ON SCHEMA myapp TO app_user;
GRANT CREATE ON SCHEMA myapp TO app_user;

-- 5. Enable SSL (postgresql.conf)
-- ssl = on
-- ssl_cert_file = 'server.crt'
-- ssl_key_file = 'server.key'
```

### Firewall Configuration

```bash
# UFW (Ubuntu)
sudo ufw allow from 192.168.1.0/24 to any port 5432
sudo ufw reload

# firewalld (RHEL/CentOS)
sudo firewall-cmd --zone=public --add-port=5432/tcp --permanent
sudo firewall-cmd --reload

# iptables
sudo iptables -A INPUT -p tcp --dport 5432 -s 192.168.1.0/24 -j ACCEPT
sudo iptables-save
```

## Exercises

### Exercise 3.1: Multi-Environment Setup
Set up three PostgreSQL environments:
1. Development (relaxed security, full logging)
2. Testing (moderate security, error logging)
3. Production (strict security, minimal logging)

Document the configuration differences.

### Exercise 3.2: Connection Pool Testing
1. Create a test script that opens 100 connections
2. Observe behavior with default max_connections
3. Implement connection pooling with PgBouncer
4. Compare performance

### Exercise 3.3: Backup and Restore
1. Create a sample database with tables and data
2. Perform a full backup using pg_dump
3. Drop the database
4. Restore from backup
5. Verify data integrity

### Exercise 3.4: SSL Configuration
1. Generate self-signed SSL certificates
2. Configure PostgreSQL to require SSL
3. Test connections with and without SSL
4. Verify encrypted connections

## Summary

You've learned how to:

✅ **Install PostgreSQL** on any major platform
✅ **Configure** for development and production environments
✅ **Create and manage** users and databases
✅ **Connect** using various tools and languages
✅ **Verify** successful installation
✅ **Troubleshoot** common issues
✅ **Secure** your installation

Your PostgreSQL installation is now ready for use. Whether you're developing locally or preparing for production, you have the foundation properly configured.

## What's Next

Chapter 4 dives into SQL fundamentals. Now that PostgreSQL is running, we'll learn how to create tables, insert data, and write queries that answer complex questions about your data.

## Additional Resources

- **PostgreSQL Installation Wiki**: [wiki.postgresql.org/wiki/Detailed_installation_guides](https://wiki.postgresql.org/wiki/Detailed_installation_guides)
- **Docker Official Images**: [hub.docker.com/_/postgres](https://hub.docker.com/_/postgres)
- **Configuration Generator**: [pgtune.leopard.in.ua](https://pgtune.leopard.in.ua/)
- **Security Best Practices**: [postgresql.org/docs/current/security.html](https://www.postgresql.org/docs/current/security.html)

---

*"A properly configured database is like a well-tuned instrument—ready to perform when called upon."*