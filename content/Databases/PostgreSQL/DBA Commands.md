---
longform:
  format: single
  title: DBA Commands
title: DBA Commands
---


### **1. Restart Specific Table Replication and CDC**

To reinitialize the replication for specific tables and trigger a fresh data copy, you can update the `pg_subscription_rel` table. This will set the state of the replication to `'i'` (initialize), which will prompt a full reload of the data for those tables.

#### **SQL Query:**

```sql
UPDATE pg_subscription_rel 
SET srsubstate = 'i'  -- 'i' = initialize (will trigger fresh data copy)
WHERE srrelid IN (
  SELECT oid FROM pg_class 
  WHERE relname IN (
    'secondary_unit_lookup', 'pagc_rules', 'geocode_settings_default',
    'state_lookup', 'loader_platform', 'direction_lookup', 'spatial_ref_sys',
    'street_type_lookup', 'pagc_gaz', 'pagc_lex', 'loader_lookuptables'
  )
);
```

---

### **2. Create Subscriber with Restricted Replication Slot**

To create a new subscriber on the target database with a specific replication slot, use the `CREATE SUBSCRIPTION` statement. Ensure that you specify the slot name and set `create_slot = false` if you don't want PostgreSQL to create the slot for you.

#### **SQL Query:**

```sql
CREATE SUBSCRIPTION aurora_sub
  CONNECTION 'host=mydb.com
              port=5432
              dbname=mydatabase
              user=postgres
              password=strongpwd!'
  PUBLICATION aurora_pub
  WITH (
    slot_name = 'aurora_slot_1',
    create_slot = false,
    copy_data = true,
    enabled = true
  );

-- Check the subscription status
SELECT * FROM pg_stat_subscription;
```

---

### **3. Check Specific Table Size**

To monitor the size of a specific table in your database, use the following query. This provides both the individual table size and the total size (including indexes).

#### **SQL Query:**

```sql
SELECT 
    relname AS table_name,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_catalog.pg_statio_user_tables
WHERE relname = 'CqItemAttributions';
```

---

### **4. Create Replication Slot on Source Database**

You can create a logical replication slot on the source database by using `pg_create_logical_replication_slot()`. This is required to enable replication between the source and the subscriber.

#### **SQL Query:**

```sql
SELECT * 
FROM pg_create_logical_replication_slot(
    'aurora_slot_1',
    'pgoutput'
);
```

---

### **5. Check Subscription Replication Status**

You can check the current replication status, including the state of each table's subscription, by querying the `pg_subscription` and `pg_subscription_rel` system catalogs.

#### **SQL Query:**

```sql
SELECT
  sub.subname,
  rel.srrelid::regclass AS table_name,
  rel.srsubstate,
  CASE rel.srsubstate
    WHEN 'i' THEN 'initialize'
    WHEN 'd' THEN 'data copy'
    WHEN 'f' THEN 'finished copy'
    WHEN 's' THEN 'synchronized'
    WHEN 'r' THEN 'ready (replicating)'
  END AS state_desc,
  rel.srsublsn,
  pg_current_wal_lsn() AS source_lsn,
  pg_wal_lsn_diff(pg_current_wal_lsn(), rel.srsublsn) AS lag_bytes
FROM pg_subscription sub
JOIN pg_subscription_rel rel ON rel.srsubid = sub.oid;
```

---

### **6. Enable / Disable Subscription**

To enable or disable a subscription (e.g., to temporarily pause replication), you can use the `ALTER SUBSCRIPTION` command.

#### **SQL Queries:**

```sql
-- Enable Subscription
ALTER SUBSCRIPTION aurora_sub ENABLE;

-- Disable Subscription
ALTER SUBSCRIPTION aurora_sub DISABLE;
```

---

### **7. Check for Deadlocks**

To troubleshoot deadlocks, you can query the `pg_stat_activity` view to find processes that are waiting for locks and potentially causing a deadlock.

#### **SQL Query:**

```sql
SELECT pid, usename, state, query, wait_event, wait_event_type
FROM pg_stat_activity
WHERE datname = 'arisinfra'
  AND pid <> pg_backend_pid();  -- Avoid showing the current session
```

---

### **8. Check Dead Tuples for Specific Table**

Dead tuples are rows that are marked for deletion but are not yet physically removed from the table due to PostgreSQL's MVCC (Multi-Version Concurrency Control) mechanism. To check the table size and dead tuple counts:

#### **SQL Query (Table Size Check):**

```sql
SELECT 
    relname AS table_name,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_catalog.pg_statio_user_tables
WHERE relname = 'CqItemAttributions';
```

---

### **9. Additional Considerations:**

- **Replication Slot Management:** Make sure to monitor and clean up replication slots periodically to avoid consuming unnecessary resources. Unused replication slots can accumulate and cause performance issues.
- **Subscription Health:** Keep an eye on the replication lag and subscription states. If you notice any issues with the synchronization of data, it might be due to network or system resource issues.
- **Deadlock Handling:** Always try to resolve deadlocks by optimizing queries, reducing transaction size, or properly indexing tables to avoid unnecessary locking.


## Databases 

- #### Size of all databases 

	To find the size of all databases in a PostgreSQL instance in gigabytes (GB), you can modify the query to convert the size to GB:

```
SELECT 
    pg_database.datname AS database_name,
    pg_database_size(pg_database.datname) / (1024 * 1024 * 1024) AS size_gb
FROM 
    pg_database
ORDER BY 
    pg_database_size(pg_database.datname) DESC;
```

## Table

- #### View Table Structure

	To view the structure of a table in PostgreSQL, you can use the `\d` command in the `psql` command-line interface. This command displays the schema of a table, including column names, data types, indexes, and constraints.

```
\d table_name
```

## Switchover To Master

Promote the standby server to become the new primary by running the following command.

```
SELECT pg_promote();
```

## PG_BASE_BACKUP

Take postgresql backup in background .

```
nohup pg_basebackup -h 172.31.33.236 -U replica_user -X stream -C -S slaveslot5 -P -w -v -R -D ./main/ > pg_basebackup.log 2>&1 &
```

- #### Use `pg_basebackup` with the `.pgpass` file

When running `pg_basebackup`, specify the host and user directly in the command. The password will be automatically picked from the `.pgpass` file.

```
localhost:5432:*:myuser:mypassword
```

**Set permissions** (important for security):

```
chmod 600 ~/.pgpass
```

## Active connection


```
SELECT
    usename,
    ssl,
    client_addr,
    application_name,
    state
FROM
    pg_stat_ssl
JOIN
    pg_stat_activity
ON
    pg_stat_ssl.pid = pg_stat_activity.pid;
```
