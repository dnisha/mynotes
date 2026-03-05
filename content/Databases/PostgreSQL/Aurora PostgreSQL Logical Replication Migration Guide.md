---
longform:
  format: single
  title: Aurora PostgreSQL Logical Replication Migration Guide
title: Aurora PostgreSQL Logical Replication Migration Guide
---

## Architecture Overview

```
┌──────────────────────────────────┐          ┌──────────────────────────────────┐
│     SOURCE (Aurora PostgreSQL)   │          │     TARGET (Aurora PostgreSQL)   │
│         database-1               │          │         database-2               │
│                                  │          │                                  │
│   ┌──────────────────────┐       │          │      ┌──────────────────────┐    │
│   │   Application        │       │          │      │   Application        │    │
│   │   (Writes)           │       │          │      │   (Reads)            │    │
│   └──────────┬───────────┘       │          │      └──────────┬───────────┘    │
│              │                   │          │                 │                 │
│   ┌──────────▼───────────┐       │          │      ┌──────────▼───────────┐    │
│   │   Database           │       │          │      │   Database           │    │
│   │   migration_test     │       │          │      │   migration_test     │    │
│   │                      │       │          │      │                      │    │
│   │   ┌──────────────┐   │       │          │      │   ┌──────────────┐   │    │
│   │   │ Publication  │   │       │          │      │   │ Subscription │   │    │
│   │   │ aurora_pub   │◄──┼───────┼──────────┼──────┼───│ aurora_sub    │   │    │
│   │   └──────────────┘   │       │ Logical │      │   └──────────────┘   │    │
│   │                      │       │ Replication     │                      │    │
│   └──────────────────────┘       │          │      └──────────────────────┘    │
│                                  │          │                                  │
│   ┌──────────────────────┐       │          │      ┌──────────────────────┐    │
│   │   repl_user          │       │          │      │   Schema Restored    │    │
│   │   (Replication)      │       │          │      │   via pg_dump -s     │    │
│   └──────────────────────┘       │          │      └──────────────────────┘    │
│                                  │          │                                  │
└──────────────────────────────────┘          └──────────────────────────────────┘
              │                                              ▲
              │                                              │
              └──────────────  Logical Replication ──────────┘
                                (Initial sync + CDC)
```

## Prerequisites

- **Source**: Aurora PostgreSQL (database-1)
- **Target**: New Aurora PostgreSQL cluster
- **Both clusters** in same VPC or peered
- **Source** `rds.logical_replication` = 1

## Step-by-Step Implementation

### 1. Create Replication User on Source

```sql
-- Connect to source database
CREATE USER repl_user WITH REPLICATION LOGIN PASSWORD 'StrongPass123!';

-- Grant necessary permissions
GRANT CONNECT ON DATABASE migration_test TO repl_user;
GRANT USAGE ON SCHEMA public TO repl_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO repl_user;

-- Ensure future tables are accessible
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
  GRANT SELECT ON TABLES TO repl_user;
```

### 2. Create Publication on Source

```sql
-- Create publication with correct name
CREATE PUBLICATION aurora_pub FOR ALL TABLES;

-- Verify creation
SELECT * FROM pg_publication;
```

### 3. Dump and Restore Schema Only

**On Source (database-1):**
```bash
pg_dump \
  -h database-1.cluster-xxx.ap-south-1.rds.amazonaws.com \
  -U postgres \
  -d migration_test \
  -s \
  -F p \
  -f migration_test_schema.sql
```

**On Target (new cluster):**
```bash
# Restore schema
psql \
  -h target-cluster.xxx.ap-south-1.rds.amazonaws.com \
  -U postgres \
  -d migration_test \
  -f migration_test_schema.sql

# Verify tables created
psql -h target-cluster.xxx.rds.amazonaws.com -U postgres -d migration_test -c "\dt"
```

### 4. Create Subscription on Target

```sql
-- Connect to target database
CREATE SUBSCRIPTION aurora_sub 
  CONNECTION 'host=database-1.cluster-xxx.ap-south-1.rds.amazonaws.com 
              port=5432 
              dbname=migration_test 
              user=repl_user 
              password=StrongPass123!' 
  PUBLICATION aurora_pub;
```

### 5. Monitor Replication Progress

**Check lag on Source:**
```sql
SELECT 
    slot_name,
    pg_current_wal_lsn() AS current_lsn,
    confirmed_flush_lsn,
    pg_size_pretty(pg_current_wal_lsn() - confirmed_flush_lsn) AS lag
FROM pg_replication_slots 
WHERE slot_name = 'aurora_sub';
```

**Check lag on Target:**
```sql
SELECT 
    subname,
    received_lsn,
    latest_end_lsn,
    pg_size_pretty(latest_end_lsn - received_lsn) AS lag
FROM pg_stat_subscription;
```

### 6. Verify Data Consistency

**Step A: LSN Match**
```sql
-- On Source
SELECT pg_current_wal_lsn();

-- On Target (should match source value)
SELECT received_lsn FROM pg_stat_subscription;
```

**Step B: Row Count Comparison**
```sql
-- Run on BOTH source and target
SELECT 
    schemaname,
    tablename,
    n_live_tup AS row_count
FROM pg_stat_user_tables
ORDER BY row_count DESC;
```

**Step C: Checksum Verification (for critical tables)**
```sql
-- Run on BOTH, compare results
SELECT 
    COUNT(*) AS row_count,
    SUM(hashtext(t::text)) AS checksum
FROM your_critical_table;
```

### 7. Cutover to Target

**Pause application writes to source**

**Verify zero lag:**
```sql
-- On Target
SELECT latest_end_lsn - received_lsn = 0 AS synced 
FROM pg_stat_subscription;
```

**Sync sequences:**
```sql
-- Get max IDs from source, set on target
SELECT setval('table_name_id_seq', (SELECT max(id) FROM table_name));
```

**Stop replication:**
```sql
-- On Target
DROP SUBSCRIPTION aurora_sub;
```

**Switch application connection string to target**

## Rollback Plan

If issues occur:
```sql
-- On Target
DROP SUBSCRIPTION aurora_sub;

-- Point application back to source
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Publication not found | Verify publication name matches exactly |
| Permission denied | Check repl_user has SELECT on all tables |
| Connection timeout | Verify security groups allow PostgreSQL port |
| Lag not zero | Wait for initial sync completion |
| Sequence gaps | Manually sync sequences after cutover |

## Key Commands Summary

```sql
-- Source
CREATE USER repl_user WITH REPLICATION LOGIN PASSWORD 'pass';
CREATE PUBLICATION aurora_pub FOR ALL TABLES;

-- Target
CREATE SUBSCRIPTION aurora_sub CONNECTION 'host=source ...' PUBLICATION aurora_pub;

-- Monitoring
SELECT * FROM pg_stat_subscription;
SELECT pg_drop_replication_slot('aurora_sub');
```

## Post-Migration Cleanup

```bash
# Remove replication slot from source
SELECT pg_drop_replication_slot('aurora_sub');

# Rotate passwords
ALTER USER repl_user WITH PASSWORD 'NewStrongPass456!';
```