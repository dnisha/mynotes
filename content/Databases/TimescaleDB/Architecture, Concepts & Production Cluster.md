---
longform:
  format: single
  title: Architecture, Concepts & Production Cluster
title: Architecture, Concepts & Production Cluster
---

## 1. What Is TimescaleDB?

TimescaleDB is a **PostgreSQL extension** — not a separate database. It lives inside PostgreSQL, hooks deep into its query planner, data model, and execution engine, and adds time-series superpowers without abandoning the full SQL ecosystem. You get ACID transactions, JOINs, rich indexes, and full tool compatibility (Grafana, Metabase, JDBC drivers, etc.) — all while handling billions of time-series rows efficiently.

---

## 2. Core Data Storage Architecture

### 2.1 Hypertables

From a user perspective, TimescaleDB exposes what look like singular tables, called **hypertables**, that are actually an abstraction or a virtual view of many individual tables holding the data, called **chunks**.

You interact with a hypertable exactly like a regular PostgreSQL table — INSERT, SELECT, JOIN, ALTER — and TimescaleDB handles the magic underneath.

**Creating a hypertable:**

```sql
-- Step 1: Create a regular table
CREATE TABLE sensor_data (
  time        TIMESTAMPTZ   NOT NULL,
  sensor_id   TEXT          NOT NULL,
  temperature DOUBLE PRECISION,
  humidity    DOUBLE PRECISION
);

-- Step 2: Convert to hypertable, partitioned by time
SELECT create_hypertable('sensor_data', 'time');

-- Optional: Add space partitioning on sensor_id
SELECT create_hypertable('sensor_data', 'time',
  partitioning_column => 'sensor_id',
  number_partitions => 4);
```

### 2.2 Chunks

Chunks are created by partitioning the hypertable's data into one or multiple dimensions: all hypertables are partitioned by a time interval, and can additionally be partitioned by a key such as device ID, location, user id, etc.

Each chunk is a real PostgreSQL table with its own indexes. This provides:

- **Chunk exclusion** — queries that filter by time skip irrelevant chunks entirely, avoiding full table scans
- **Local indexes** — indexes stay small and fit in memory regardless of total dataset size
- **Parallel I/O** — multiple chunks can be read in parallel

```
Hypertable: sensor_data
├── chunk_1  (Jan 1–7)
├── chunk_2  (Jan 8–14)
├── chunk_3  (Jan 15–21)    ← only this chunk is scanned for Jan 15–20 queries
└── chunk_4  (Jan 22–28)
```

### 2.3 Chunk Interval Sizing

The default chunk interval is **7 days**. Proper sizing is critical for performance:

```sql
-- Set chunk interval at creation
SELECT create_hypertable('sensor_data', 'time',
  chunk_time_interval => INTERVAL '1 day');

-- Or change it after creation
SELECT set_chunk_time_interval('sensor_data', INTERVAL '12 hours');
```

**Rule of thumb:** Each uncompressed chunk should fit in about **25% of RAM** so the most recent chunk (the hot write path) stays in memory.

### 2.4 Row Store vs. Column Store (Compression)

TimescaleDB supports two physical storage formats within the same hypertable:

|Storage Type|Best For|Compression|
|---|---|---|
|**Rowstore** (default)|Recent/hot data, frequent writes|None|
|**Columnstore**|Cold/historical data, analytics|90%+ typical|

While many systems support this concept of data cooling, TimescaleDB ensures that the data can still be queried from the same hypertable regardless of its current location.

**Enabling the columnstore (compression):**

```sql
-- Set compression policy (compress after 7 days)
ALTER TABLE sensor_data SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'sensor_id',
  timescaledb.compress_orderby = 'time DESC'
);

-- Automate compression
SELECT add_compression_policy('sensor_data', INTERVAL '7 days');
```

The `segmentby` column groups related rows (e.g., all readings for one sensor) into a single columnstore batch — enabling vectorized scans that only touch relevant data.

---

## 3. Key Features & Concepts

### 3.1 Continuous Aggregates

Continuous aggregates make real-time analytics run faster on very large datasets. They continuously and incrementally refresh a query in the background, so that when you run such query, only the data that has changed needs to be computed, not the entire dataset. This is what makes them different from regular PostgreSQL materialized views, which cannot be incrementally materialized and have to be rebuilt from scratch every time you want to refresh them.

```sql
-- Create a continuous aggregate (hourly rollup)
CREATE MATERIALIZED VIEW hourly_stats
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 hour', time) AS hour,
  sensor_id,
  AVG(temperature) AS avg_temp,
  MAX(temperature) AS max_temp,
  COUNT(*) AS reading_count
FROM sensor_data
GROUP BY hour, sensor_id;

-- Automate refresh
SELECT add_continuous_aggregate_policy('hourly_stats',
  start_offset => INTERVAL '3 hours',
  end_offset   => INTERVAL '1 hour',
  schedule_interval => INTERVAL '1 hour');
```

### 3.2 Data Retention Policies

```sql
-- Automatically drop chunks older than 1 year
SELECT add_retention_policy('sensor_data', INTERVAL '1 year');
```

### 3.3 Tiered Storage (S3)

TimescaleDB can tier old chunks to object storage (S3) for bottomless, low-cost archival — while still querying through the same hypertable SQL interface.

### 3.4 `time_bucket()` Function

The core time-series aggregation primitive:

```sql
-- Aggregate per 15 minutes
SELECT
  time_bucket('15 minutes', time) AS bucket,
  sensor_id,
  AVG(temperature)
FROM sensor_data
WHERE time > NOW() - INTERVAL '24 hours'
GROUP BY bucket, sensor_id
ORDER BY bucket DESC;
```

---

## 4. Storage Architecture Diagram

```
┌──────────────────────────────────────────────────────────┐
│                    PostgreSQL Process                    │
│  ┌────────────────────────────────────────────────────┐  │
│  │              TimescaleDB Extension                 │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │   Hypertable (virtual unified table)        │  │  │
│  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐   │  │  │
│  │  │  │ Chunk 1  │ │ Chunk 2  │ │ Chunk 3  │   │  │  │
│  │  │  │ (row)    │ │ (row)    │ │(columnar)│   │  │  │
│  │  │  │ Jan 1-7  │ │ Jan 8-14 │ │Dec 1-31  │   │  │  │
│  │  │  │ HOT DATA │ │ WARM     │ │COMPRESSED│   │  │  │
│  │  │  └──────────┘ └──────────┘ └──────────┘   │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  │  ┌──────────────────────────────────────────────┐  │  │
│  │  │  Continuous Aggregates (Materialized Views)  │  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

---

## 5. Production Cluster Architecture

### 5.1 Replication Strategy

Streaming replication is the primary method of replication supported by TimescaleDB. It works by having the primary database server stream its write-ahead log (WAL) entries to its database replicas. Each replica then replays these log entries against its own database to reach a state consistent with the primary database.

> **Note:** TimescaleDB is **not** currently compatible with logical replication. Streaming (physical) replication is the standard approach.

### 5.2 High Availability with Patroni

Ultimately, Timescale went with Patroni because it combined robust and reliable failover with a simple architecture and easy-to-use interface. Patroni uses **etcd** as its distributed consensus store.

Etcd must be deployed in an HA fashion that allows its individual nodes to reach a quorum about the state of the cluster. This requires a minimum deployment of 3 etcd nodes. Leader election is handled by attempting to set an expiring key in etcd.

### 5.3 Production Cluster Architecture Diagram

```
                        ┌──────────────────────┐
                        │   Application Layer  │
                        │  (App Servers / APIs)│
                        └──────────┬───────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │         Load Balancer        │
                    │   HAProxy / PgBouncer        │
                    │  Writes → Primary            │
                    │  Reads  → Replicas           │
                    └────┬──────────────────┬──────┘
                         │                  │
          ┌──────────────▼───┐   ┌──────────▼───────────┐
          │  PRIMARY NODE    │   │  REPLICA NODE(s)     │
          │  TimescaleDB     │──▶│  TimescaleDB         │
          │  (Read + Write)  │WAL│  (Read-only)         │
          │  AZ-1            │   │  AZ-2, AZ-3          │
          └──────────────────┘   └──────────────────────┘
                    │
          ┌─────────▼──────────────────────┐
          │      etcd Cluster (3 nodes)    │
          │  Consensus / Leader Election   │
          │  (Patroni coordination)        │
          └────────────────────────────────┘
```

### 5.4 Synchronous vs. Asynchronous Replication

Synchronous replication: the primary commits its next write once the replica confirms that the previous write is complete. There is no lag between the primary and the replica. They are in the same state at all times. This is preferable if you need the highest level of data integrity. However, this affects the primary ingestion time. Asynchronous replication: the primary commits its next write without the confirmation of the previous write completion. The asynchronous replicas often have a lag, in both time and data, compared to the primary. This is preferable if you need the shortest primary ingest time.

---

## 6. Step-by-Step: Self-Hosted Production Setup

### Step 1: Install TimescaleDB on all nodes

```bash
# Ubuntu/Debian
sudo apt install -y gnupg postgresql-common apt-transport-https lsb-release wget
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh

# Add Timescale repo
echo "deb https://packagecloud.io/timescale/timescaledb/ubuntu/ $(lsb_release -c -s) main" \
  | sudo tee /etc/apt/sources.list.d/timescaledb.list

sudo apt update && sudo apt install -y timescaledb-2-postgresql-16

# Tune PostgreSQL for TimescaleDB
sudo timescaledb-tune --quiet --yes

sudo systemctl restart postgresql
```

### Step 2: Install Patroni and etcd

```bash
# On all DB nodes
pip install patroni[etcd] psycopg2-binary

# On etcd nodes (3 minimum)
sudo apt install -y etcd
```

### Step 3: Configure etcd cluster

```yaml
# /etc/etcd/etcd.conf on each etcd node
name: etcd-node1
data-dir: /var/lib/etcd
listen-peer-urls: http://10.0.0.1:2380
listen-client-urls: http://10.0.0.1:2379,http://127.0.0.1:2379
advertise-client-urls: http://10.0.0.1:2379
initial-advertise-peer-urls: http://10.0.0.1:2380
initial-cluster: etcd-node1=http://10.0.0.1:2380,etcd-node2=http://10.0.0.2:2380,etcd-node3=http://10.0.0.3:2380
initial-cluster-state: new
```

### Step 4: Configure Patroni

```yaml
# /etc/patroni/patroni.yml on primary node
scope: timescale-cluster
namespace: /db/
name: pg-node1

etcd:
  hosts: 10.0.0.1:2379,10.0.0.2:2379,10.0.0.3:2379

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.1.1:8008

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576  # 1MB
    postgresql:
      use_pg_rewind: true
  pg_hba:
    - host replication replicator 0.0.0.0/0 md5
    - host all all 0.0.0.0/0 md5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.1:5432
  data_dir: /var/lib/postgresql/16/main
  bin_dir: /usr/lib/postgresql/16/bin
  authentication:
    replication:
      username: replicator
      password: strongpassword
    superuser:
      username: postgres
      password: strongpassword
  parameters:
    shared_preload_libraries: 'timescaledb'
    wal_level: replica
    hot_standby: 'on'
    max_wal_senders: 10
    max_replication_slots: 10
    wal_log_hints: 'on'
    # TimescaleDB tuning
    max_connections: 200
    shared_buffers: 8GB          # 25% of RAM
    effective_cache_size: 24GB   # 75% of RAM
    work_mem: 50MB
    maintenance_work_mem: 2GB
    checkpoint_completion_target: 0.9
    timescaledb.max_background_workers: 16
```

### Step 5: Configure HAProxy for connection routing

```
# /etc/haproxy/haproxy.cfg
frontend timescale_write
  bind *:5000
  default_backend primary_node

frontend timescale_read
  bind *:5001
  default_backend replica_nodes

backend primary_node
  option httpchk GET /primary
  server pg-node1 10.0.1.1:5432 check port 8008
  server pg-node2 10.0.1.2:5432 check port 8008 backup
  server pg-node3 10.0.1.3:5432 check port 8008 backup

backend replica_nodes
  balance roundrobin
  option httpchk GET /replica
  server pg-node2 10.0.1.2:5432 check port 8008
  server pg-node3 10.0.1.3:5432 check port 8008
```

Patroni exposes `/primary` and `/replica` HTTP endpoints that HAProxy polls to know who to route traffic to.

### Step 6: Add PgBouncer for connection pooling

```ini
# /etc/pgbouncer/pgbouncer.ini
[databases]
* = host=127.0.0.1 port=5432

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0
auth_type = md5
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
```

---

## 7. Hypertable Initialization in Production

Once the cluster is up, enable TimescaleDB and set up your schema:

```sql
-- Enable extension
CREATE EXTENSION IF NOT EXISTS timescaledb CASCADE;

-- Create hypertable with space partitioning
CREATE TABLE metrics (
  time        TIMESTAMPTZ   NOT NULL,
  device_id   TEXT          NOT NULL,
  metric_name TEXT          NOT NULL,
  value       DOUBLE PRECISION
);

SELECT create_hypertable('metrics', 'time',
  partitioning_column => 'device_id',
  number_partitions => 4,
  chunk_time_interval => INTERVAL '1 day');

-- Add indexes
CREATE INDEX ON metrics (device_id, time DESC);
CREATE INDEX ON metrics (metric_name, time DESC);

-- Compression policy (columnstore after 7 days)
ALTER TABLE metrics SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'device_id',
  timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('metrics', INTERVAL '7 days');

-- Continuous aggregate (hourly pre-aggregation)
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 hour', time) AS hour,
  device_id,
  metric_name,
  AVG(value) AS avg_val,
  MAX(value) AS max_val
FROM metrics
GROUP BY hour, device_id, metric_name;

SELECT add_continuous_aggregate_policy('metrics_hourly',
  start_offset => INTERVAL '3 hours',
  end_offset   => INTERVAL '1 hour',
  schedule_interval => INTERVAL '1 hour');

-- Data retention (drop raw data older than 90 days)
SELECT add_retention_policy('metrics', INTERVAL '90 days');
```

---

## 8. Production Best Practices Summary

|Area|Recommendation|
|---|---|
|**HA**|Patroni + etcd (3 etcd nodes minimum)|
|**Replication**|Streaming replication (async for high ingestion, sync for high integrity)|
|**Connection pooling**|PgBouncer in transaction mode in front of every node|
|**Load balancing**|HAProxy routing writes to `/primary`, reads to `/replica`|
|**Chunk interval**|Size so each uncompressed chunk ≈ 25% of available RAM|
|**Compression**|Enable columnstore after 7–30 days with `segmentby` on your main filter column|
|**Continuous aggregates**|Pre-aggregate at every granularity you query (1min, 1hr, 1day)|
|**Monitoring**|Prometheus + `pg_stat_statements` + TimescaleDB telemetry views|
|**Backups**|`pgBackRest` with S3 backend + PITR enabled|
|**Kubernetes**|Use the Zalando postgres-operator or CloudNativePG with `timescaledb-ha` image|

This architecture gives you horizontal read scaling, automatic failover in ~30 seconds, full SQL query capability, and the time-series performance features (chunking, compression, continuous aggregates) that make TimescaleDB purpose-built for high-volume time-series workloads.