# Appendix B: Troubleshooting Guide

*By Claude Code Opus 4.1*

When PostgreSQL misbehaves, this guide helps you diagnose and fix the problem quickly. Listed from most common to most obscure.

## Connection Issues

### "FATAL: password authentication failed"

**Symptoms:**
```
psql: FATAL: password authentication failed for user "postgres"
```

**Diagnosis:**
```bash
# Check pg_hba.conf settings
sudo cat /etc/postgresql/16/main/pg_hba.conf | grep -v "^#"

# Verify user exists
sudo -u postgres psql -c "\du"
```

**Solutions:**
```bash
# Solution 1: Reset password
sudo -u postgres psql
ALTER USER postgres PASSWORD 'new_password';

# Solution 2: Fix pg_hba.conf
# Change "md5" or "scram-sha-256" to "trust" temporarily
sudo vim /etc/postgresql/16/main/pg_hba.conf
# local   all   all   trust
sudo systemctl reload postgresql

# Solution 3: Create missing user
sudo -u postgres createuser --interactive --pwprompt username
```

### "FATAL: too many connections"

**Symptoms:**
```
psql: FATAL: sorry, too many clients already
```

**Diagnosis:**
```sql
-- Check current connections
SELECT count(*) FROM pg_stat_activity;

-- See connection limit
SHOW max_connections;

-- Identify connection hogs
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    state_change
FROM pg_stat_activity
ORDER BY state_change;
```

**Solutions:**
```sql
-- Immediate: Kill idle connections
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
    AND state_change < now() - interval '10 minutes';

-- Short-term: Increase connections
ALTER SYSTEM SET max_connections = 200;
SELECT pg_reload_conf();
-- Restart required!

-- Long-term: Use connection pooler
# Install PgBouncer
apt-get install pgbouncer
```

### "could not connect to server: Connection refused"

**Diagnosis:**
```bash
# Is PostgreSQL running?
systemctl status postgresql

# Check if listening on correct port
sudo netstat -plnt | grep 5432

# Check PostgreSQL logs
sudo tail -f /var/log/postgresql/postgresql-16-main.log
```

**Solutions:**
```bash
# Start PostgreSQL
sudo systemctl start postgresql

# Fix listen_addresses in postgresql.conf
listen_addresses = '*'  # or specific IP

# Fix firewall
sudo ufw allow 5432/tcp

# Fix Docker networking
docker run -p 5432:5432 postgres:16
```

## Performance Issues

### Slow Queries

**Diagnosis:**
```sql
-- Enable query logging
ALTER SYSTEM SET log_min_duration_statement = 1000; -- Log queries over 1s
SELECT pg_reload_conf();

-- Find slow queries
SELECT
    mean_exec_time,
    calls,
    total_exec_time,
    query
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Check currently running queries
SELECT
    pid,
    now() - query_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;
```

**Solutions:**
```sql
-- Add missing index
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
CREATE INDEX CONCURRENTLY idx_table_column ON table(column);

-- Update statistics
ANALYZE table_name;

-- Increase work_mem for sorting
SET work_mem = '256MB'; -- Session only
ALTER SYSTEM SET work_mem = '256MB'; -- Persistent

-- Rewrite query
-- Instead of NOT IN
SELECT * FROM a WHERE id NOT IN (SELECT id FROM b);
-- Use NOT EXISTS
SELECT * FROM a WHERE NOT EXISTS (SELECT 1 FROM b WHERE b.id = a.id);
```

### High CPU Usage

**Diagnosis:**
```bash
# System level
top -u postgres
htop

# PostgreSQL level
SELECT
    pid,
    usename,
    application_name,
    state,
    query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY backend_start;
```

**Solutions:**
```sql
-- Kill runaway query
SELECT pg_cancel_backend(pid); -- Gentle
SELECT pg_terminate_backend(pid); -- Force

-- Fix autovacuum
ALTER SYSTEM SET autovacuum_max_workers = 4;
ALTER SYSTEM SET autovacuum_naptime = '30s';

-- Tune parallel workers
ALTER SYSTEM SET max_parallel_workers_per_gather = 4;
ALTER SYSTEM SET max_parallel_workers = 8;
```

### Out of Memory

**Symptoms:**
```
ERROR: out of memory
DETAIL: Failed on request of size 134217728
```

**Diagnosis:**
```sql
-- Check memory settings
SHOW shared_buffers;
SHOW work_mem;
SHOW maintenance_work_mem;

-- Find memory-hungry queries
SELECT
    pid,
    usename,
    query,
    state
FROM pg_stat_activity
WHERE state = 'active';
```

**Solutions:**
```sql
-- Reduce work_mem
ALTER SYSTEM SET work_mem = '4MB';

-- Reduce shared_buffers
ALTER SYSTEM SET shared_buffers = '256MB';

-- Add swap (emergency only!)
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

## Data Corruption

### "ERROR: could not read block"

**Diagnosis:**
```sql
-- Check for corruption
CREATE EXTENSION amcheck;
SELECT bt_index_check('index_name');

-- Full database check
REINDEX DATABASE mydb;
```

**Solutions:**
```bash
# Option 1: Restore from backup
pg_restore -d mydb backup.dump

# Option 2: Use zero_damaged_pages (DATA LOSS!)
SET zero_damaged_pages = on;
VACUUM FULL damaged_table;
SET zero_damaged_pages = off;

# Option 3: pg_dump what you can
pg_dump --table=good_table -f good_data.sql mydb
```

### "ERROR: duplicate key value"

**Diagnosis:**
```sql
-- Find duplicates
SELECT column, COUNT(*)
FROM table
GROUP BY column
HAVING COUNT(*) > 1;

-- Check constraint definition
\d table_name
```

**Solutions:**
```sql
-- Remove duplicates
DELETE FROM table a
USING table b
WHERE a.ctid < b.ctid
    AND a.duplicate_column = b.duplicate_column;

-- Rebuild constraint
ALTER TABLE table DROP CONSTRAINT constraint_name;
ALTER TABLE table ADD CONSTRAINT constraint_name UNIQUE (column);
```

## Replication Issues

### Replication Lag

**Diagnosis:**
```sql
-- On primary
SELECT
    client_addr,
    state,
    sync_state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- On replica
SELECT
    now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

**Solutions:**
```sql
-- Increase WAL senders
ALTER SYSTEM SET max_wal_senders = 10;

-- Increase WAL size
ALTER SYSTEM SET wal_keep_size = '2GB';

-- Switch to async replication (temporary)
ALTER SYSTEM SET synchronous_commit = 'local';
```

### Replica Not Connecting

**Diagnosis:**
```bash
# Check replica logs
tail -f /var/log/postgresql/postgresql-16-main.log

# Test connection from replica
psql -h primary_host -U replicator -d postgres -c "SELECT 1"

# Check replication slots
SELECT * FROM pg_replication_slots;
```

**Solutions:**
```sql
-- Recreate replication slot
SELECT pg_drop_replication_slot('replica_1');
SELECT pg_create_physical_replication_slot('replica_1');

-- Fix pg_hba.conf on primary
host replication replicator replica_ip/32 scram-sha-256

-- Rebuild replica
pg_basebackup -h primary -U replicator -D /var/lib/postgresql/16/main -P -v -R
```

## Disk Space Issues

### "ERROR: could not extend file"

**Diagnosis:**
```bash
# Check disk space
df -h
du -sh /var/lib/postgresql/*

# Find large tables
```

```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;
```

**Solutions:**
```sql
-- Clean up
VACUUM FULL large_table;
TRUNCATE TABLE temp_data;

-- Drop old partitions
DROP TABLE IF EXISTS data_2023_01;

-- Clear WAL files
-- Checkpoint first
CHECKPOINT;
-- Then clean
pg_archivecleanup /var/lib/postgresql/16/main/pg_wal/ 000000010000000000000010

-- Move to tablespace on different disk
CREATE TABLESPACE bigdata LOCATION '/mnt/ssd/postgres';
ALTER TABLE large_table SET TABLESPACE bigdata;
```

## Lock Issues

### Deadlocks

**Diagnosis:**
```sql
-- Enable deadlock logging
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET deadlock_timeout = '1s';

-- Find blocking queries
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event_type = 'Lock';
```

**Solutions:**
```sql
-- Kill blocking session
SELECT pg_terminate_backend(blocking_pid);

-- Prevent deadlocks
-- Always lock in same order
BEGIN;
SELECT * FROM table_a WHERE id = 1 FOR UPDATE;
SELECT * FROM table_b WHERE id = 1 FOR UPDATE;
COMMIT;

-- Use advisory locks
SELECT pg_advisory_lock(12345);
-- Do work
SELECT pg_advisory_unlock(12345);
```

## Backup/Recovery Issues

### pg_dump Hanging

**Diagnosis:**
```bash
# Check what it's doing
strace -p $(pgrep pg_dump)

# Check locks
psql -c "SELECT * FROM pg_stat_activity WHERE wait_event IS NOT NULL"
```

**Solutions:**
```bash
# Use --no-synchronized-snapshots
pg_dump --no-synchronized-snapshots -d mydb -f backup.sql

# Exclude problematic table
pg_dump --exclude-table=problem_table -d mydb -f backup.sql

# Use directory format for parallel dump
pg_dump -j 4 -Fd -f backup_dir mydb
```

### Recovery Failing

**Diagnosis:**
```bash
# Check recovery status
psql -c "SELECT pg_is_in_recovery()"

# Check logs
grep ERROR /var/log/postgresql/*.log
```

**Solutions:**
```bash
# Fix recovery.conf / postgresql.auto.conf
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2024-01-01 12:00:00'

# Use pg_resetwal (DANGEROUS!)
pg_resetwal -f /var/lib/postgresql/16/main

# Point-in-time recovery
pg_basebackup -D /recovery -R
echo "recovery_target_time = '2024-01-01 12:00:00'" >> recovery.conf
```

## Common Error Codes

### Quick Reference

| Error Code | Meaning | Quick Fix |
|------------|---------|-----------|
| 08001 | Connection error | Check network/firewall |
| 08003 | Connection doesn't exist | Reconnect |
| 08006 | Connection failure | Check server status |
| 23505 | Unique violation | Handle duplicates |
| 23503 | Foreign key violation | Check references |
| 40001 | Serialization failure | Retry transaction |
| 40P01 | Deadlock detected | Retry transaction |
| 53100 | Disk full | Free space |
| 53200 | Out of memory | Reduce work_mem |
| 53300 | Too many connections | Use pooler |
| 57014 | Query cancelled | Check statement_timeout |

## Emergency Procedures

### Database Won't Start

```bash
#!/bin/bash
# Emergency startup script

# 1. Check obvious issues
df -h  # Disk space
free -h  # Memory
systemctl status postgresql

# 2. Try manual start
sudo -u postgres /usr/lib/postgresql/16/bin/postgres \
    -D /var/lib/postgresql/16/main \
    -c config_file=/etc/postgresql/16/main/postgresql.conf

# 3. Single-user mode
sudo -u postgres /usr/lib/postgresql/16/bin/postgres \
    --single -D /var/lib/postgresql/16/main mydb

# 4. Reset WAL if corrupted
pg_resetwal -f /var/lib/postgresql/16/main

# 5. Last resort: initdb new cluster
mv /var/lib/postgresql/16/main /var/lib/postgresql/16/main.broken
sudo -u postgres initdb -D /var/lib/postgresql/16/main
```

### Data Recovery Checklist

1. **Stop writes immediately**
   ```sql
   ALTER SYSTEM SET default_transaction_read_only = on;
   SELECT pg_reload_conf();
   ```

2. **Backup current state**
   ```bash
   tar -czf emergency_backup.tar.gz /var/lib/postgresql/16/main
   ```

3. **Check backup availability**
   ```bash
   ls -la /backups/
   barman list-backup all
   wal-g backup-list
   ```

4. **Test recovery on separate server**
   ```bash
   pg_restore -d test_recovery backup.dump
   ```

5. **Perform recovery**
   ```bash
   systemctl stop postgresql
   mv /var/lib/postgresql/16/main /var/lib/postgresql/16/main.old
   pg_basebackup -h backup_server -D /var/lib/postgresql/16/main -P -R
   systemctl start postgresql
   ```

## Monitoring Commands Cheat Sheet

```sql
-- Quick health check
CREATE OR REPLACE FUNCTION quick_health_check()
RETURNS TABLE(
    metric text,
    value text,
    status text
) AS $$
BEGIN
    RETURN QUERY
    -- Connections
    SELECT 'Active Connections',
           COUNT(*)::text,
           CASE WHEN COUNT(*) > 100 THEN 'WARNING' ELSE 'OK' END
    FROM pg_stat_activity

    UNION ALL

    -- Database size
    SELECT 'Database Size',
           pg_size_pretty(pg_database_size(current_database())),
           CASE WHEN pg_database_size(current_database()) > 100000000000
                THEN 'WARNING' ELSE 'OK' END

    UNION ALL

    -- Replication lag
    SELECT 'Max Replication Lag',
           COALESCE(MAX(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn))::text, '0'),
           CASE WHEN MAX(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) > 1000000
                THEN 'WARNING' ELSE 'OK' END
    FROM pg_stat_replication

    UNION ALL

    -- Cache hit ratio
    SELECT 'Cache Hit Ratio',
           ROUND(100.0 * SUM(blks_hit) / NULLIF(SUM(blks_hit) + SUM(blks_read), 0), 2)::text || '%',
           CASE WHEN 100.0 * SUM(blks_hit) / NULLIF(SUM(blks_hit) + SUM(blks_read), 0) < 90
                THEN 'WARNING' ELSE 'OK' END
    FROM pg_stat_database

    UNION ALL

    -- Long-running queries
    SELECT 'Long Running Queries',
           COUNT(*)::text,
           CASE WHEN COUNT(*) > 0 THEN 'WARNING' ELSE 'OK' END
    FROM pg_stat_activity
    WHERE state = 'active'
        AND now() - query_start > interval '5 minutes';
END;
$$ LANGUAGE plpgsql;

-- Run it
SELECT * FROM quick_health_check();
```

## Getting Help

When all else fails:

1. **PostgreSQL Logs**: Your first stop
   ```bash
   journalctl -u postgresql -n 100
   tail -f /var/log/postgresql/*.log
   ```

2. **PostgreSQL Slack**: https://postgres-slack.herokuapp.com/
3. **Stack Overflow**: Tag with `postgresql`
4. **PostgreSQL Mailing Lists**: https://www.postgresql.org/list/
5. **Commercial Support**: EnterpriseDB, 2ndQuadrant, Percona

Remember: Always have backups. Always test recovery. Always monitor proactively. The best troubleshooting is preventing issues before they happen.