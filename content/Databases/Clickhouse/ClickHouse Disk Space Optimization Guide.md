---
longform:
  format: single
  title: ClickHouse Disk Space Optimization Guide
title: ClickHouse Disk Space Optimization Guide
---

## Table of Contents
1. [Understanding ClickHouse Disk Usage](#understanding-clickhouse-disk-usage)
2. [Emergency Recovery](#emergency-recovery)
3. [Preventive Strategies](#preventive-strategies)
4. [Monitoring & Maintenance](#monitoring--maintenance)
5. [Best Practices](#best-practices)

---

## Understanding ClickHouse Disk Usage

### System Tables That Consume Space

| System Table | Typical Growth Rate | Purpose |
|-------------|-------------------|---------|
| `trace_log` | Very High (10GB+/week) | Query profiling, stack traces |
| `text_log` | High (8GB+/week) | Server logs stored in DB |
| `metric_log` | Moderate (3GB+/week) | System metrics |
| `query_log` | Moderate-High | Query history |
| `asynchronous_metric_log` | Moderate | Async metrics |
| `query_thread_log` | Moderate | Thread-level query info |
| `part_log` | Low | Merge/partition operations |

### Where ClickHouse Stores Data

```bash
/var/lib/clickhouse/
├── data/                    # Table data
│   ├── system/              # System tables
│   └── {databases}/         # User databases
├── tmp/                     # Temporary files (merges, sorts)
├── metadata/                # Table metadata
├── preprocessed_configs/    # Processed configs
├── store/                   # Modern storage format
└── access/                  # Access control
```

---

## Emergency Recovery

### 1. Immediate Action: Filesystem-Level Cleanup

When disk is 100% full and ClickHouse won't respond:

```bash
# Check disk usage
df -h
du -sh /var/lib/clickhouse/* --exclude=store 2>/dev/null

# Find largest offenders
sudo find /var/lib/clickhouse/data/system -type d -name "*" -exec du -sh {} \; 2>/dev/null | sort -hr | head -20

# Emergency cleanup (direct deletion while ClickHouse is running)
sudo rm -rf /var/lib/clickhouse/data/system/trace_log/*/
sudo rm -rf /var/lib/clickhouse/data/system/text_log/*/
sudo rm -rf /var/lib/clickhouse/data/system/metric_log/*/
sudo rm -rf /var/lib/clickhouse/data/system/asynchronous_metric_log/*/
sudo rm -rf /var/lib/clickhouse/data/system/latency_log/*/

# Clear temporary files
sudo rm -rf /var/lib/clickhouse/tmp/*
sudo rm -rf /var/lib/clickhouse/preprocessed_configs/*
```

### 2. Verify Space Freed

```bash
df -h
sudo clickhouse-client --query "SELECT free_space FROM system.disks"
```

### 3. Recover ClickHouse Operation

```sql
-- Reload configuration
SYSTEM RELOAD CONFIG;

-- Check system tables size
SELECT 
    database,
    table,
    formatReadableSize(sum(bytes_on_disk)) AS size,
    count() AS parts
FROM system.parts 
WHERE active = 1 AND database = 'system'
GROUP BY database, table 
ORDER BY sum(bytes_on_disk) DESC;
```

---

## Preventive Strategies

### 1. Table-Level TTL Configuration

Apply TTL to system tables:

```sql
-- Query log (keep 30 days)
ALTER TABLE system.query_log 
    MODIFY TTL event_date + INTERVAL 30 DAY DELETE;

-- Trace log (keep 7 days, usually enough for debugging)
ALTER TABLE system.trace_log 
    MODIFY TTL event_date + INTERVAL 7 DAY DELETE;

-- Text log (keep 7 days)
ALTER TABLE system.text_log 
    MODIFY TTL event_date + INTERVAL 7 DAY DELETE;

-- Metric logs (keep 15 days)
ALTER TABLE system.metric_log 
    MODIFY TTL event_date + INTERVAL 15 DAY DELETE;

ALTER TABLE system.asynchronous_metric_log 
    MODIFY TTL event_date + INTERVAL 15 DAY DELETE;
```

### 2. Disable Unnecessary Logs

Create `/etc/clickhouse-server/config.d/log_optimization.xml`:

```xml
<clickhouse>
    <!-- Disable high-volume logs not needed -->
    <trace_log remove="1"/>
    <latency_log remove="1"/>
    <processors_profile_log remove="1"/>
    <query_metric_log remove="1"/>
    
    <!-- Keep essential logs with reduced settings -->
    <text_log>
        <level>warning</level>
        <ttl>event_date + INTERVAL 7 DAY DELETE</ttl>
        <partition_by>toYYYYMM(event_date)</partition_by>
    </text_log>

    <metric_log>
        <collect_interval_milliseconds>60000</collect_interval_milliseconds>
        <ttl>event_date + INTERVAL 15 DAY DELETE</ttl>
        <partition_by>toYYYYMM(event_date)</partition_by>
    </metric_log>

    <query_log>
        <ttl>event_date + INTERVAL 30 DAY DELETE</ttl>
        <partition_by>toYYYYMM(event_date)</partition_by>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </query_log>

    <!-- Reduce server log verbosity -->
    <logger>
        <level>warning</level>
        <size>500M</size>
        <count>3</count>
    </logger>
</clickhouse>
```

### 3. Compression Optimization

Enable better compression for system tables:

```sql
-- Recompress system tables with better compression
ALTER TABLE system.query_log MODIFY SETTING compression = 'lz4hc(9)';
ALTER TABLE system.trace_log MODIFY SETTING compression = 'zstd(3)';
ALTER TABLE system.metric_log MODIFY SETTING compression = 'lz4hc(5)';
```

### 4. Partition Strategy

Implement smart partitioning:

```sql
-- For user tables, partition by date
CREATE TABLE my_table (
    event_date Date,
    ...
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)  -- Monthly partitions
ORDER BY (event_date, ...)
TTL event_date + INTERVAL 90 DAY DELETE;  -- Auto-delete after 90 days
```

---

## Monitoring & Maintenance

### 1. Disk Usage Monitoring Script

Create `/usr/local/bin/clickhouse-disk-check.sh`:

```bash
#!/bin/bash

THRESHOLD=80  # Alert at 80% usage
CLICKHOUSE_DATA="/var/lib/clickhouse"

# Check disk usage
USAGE=$(df -h $CLICKHOUSE_DATA | awk 'NR==2 {print $5}' | sed 's/%//')

if [ $USAGE -gt $THRESHOLD ]; then
    echo "WARNING: ClickHouse disk usage is at ${USAGE}%"
    
    # Find top 10 largest tables
    clickhouse-client --query "
        SELECT 
            database,
            table,
            formatReadableSize(sum(bytes_on_disk)) AS size
        FROM system.parts 
        WHERE active = 1
        GROUP BY database, table 
        ORDER BY sum(bytes_on_disk) DESC 
        LIMIT 10;
    "
    
    # Check if TTL is working
    clickhouse-client --query "
        SELECT 
            table,
            formatReadableSize(sum(data_uncompressed_bytes)) as size,
            min(event_date) as oldest,
            max(event_date) as newest
        FROM system.parts 
        WHERE database = 'system' AND active = 1
        GROUP BY table;
    "
fi

# Check for stale temporary files
TMP_SIZE=$(du -sb /var/lib/clickhouse/tmp 2>/dev/null | cut -f1)
if [ $TMP_SIZE -gt 1073741824 ]; then  # 1GB
    echo "Large tmp directory: $(du -sh /var/lib/clickhouse/tmp)"
fi
```

Add to crontab:
```bash
*/30 * * * * /usr/local/bin/clickhouse-disk-check.sh
```

### 2. Automated Cleanup Job

Create `/etc/cron.daily/clickhouse-cleanup`:

```bash
#!/bin/bash

# Force TTL cleanup
clickhouse-client --query "OPTIMIZE TABLE system.query_log FINAL" 2>/dev/null
clickhouse-client --query "OPTIMIZE TABLE system.trace_log FINAL" 2>/dev/null

# Clean old detached parts
find /var/lib/clickhouse/data/*/*/detached -type f -mtime +7 -delete 2>/dev/null

# Truncate server logs if too large
LOG_DIR="/var/log/clickhouse-server"
if [ -f "$LOG_DIR/clickhouse-server.log" ]; then
    SIZE=$(stat -c%s "$LOG_DIR/clickhouse-server.log" 2>/dev/null || echo 0)
    if [ $SIZE -gt 1073741824 ]; then  # 1GB
        sudo truncate -s 0 "$LOG_DIR/clickhouse-server.log"
    fi
fi
```

### 3. Proactive Alerts

Set up monitoring queries:

```sql
-- Daily growth report
SELECT 
    toDate(event_date) as date,
    formatReadableSize(sum(bytes_on_disk)) as daily_growth
FROM system.parts
WHERE database != 'system'
GROUP BY date
ORDER BY date DESC
LIMIT 30;

-- Tables without TTL
SELECT 
    database,
    table,
    engine
FROM system.tables 
WHERE database NOT IN ('system') 
  AND engine LIKE '%MergeTree%'
  AND metadata_modification_time > now() - INTERVAL 30 DAY
  AND NOT has(metadata_modification_time, 'TTL');
```

---

## Best Practices

### 1. Configuration Best Practices

```xml
<!-- /etc/clickhouse-server/config.d/disk_optimizations.xml -->
<clickhouse>
    <!-- Merge settings -->
    <merge_tree>
        <max_bytes_to_merge_at_max_space_in_pool>107374182400</max_bytes_to_merge_at_max_space_in_pool>  <!-- 100GB -->
        <max_bytes_to_merge_at_min_space_in_pool>10485760</max_bytes_to_merge_at_min_space_in_pool>  <!-- 10MB -->
        <parts_to_throw_insert>300</parts_to_throw_insert>
        <parts_to_delay_insert>150</parts_to_delay_insert>
    </merge_tree>

    <!-- Storage settings -->
    <storage_configuration>
        <disks>
            <default>
                <keep_free_space_bytes>10737418240</keep_free_space_bytes>  <!-- Keep 10GB free -->
            </default>
        </disks>
    </storage_configuration>

    <!-- Query log rotation -->
    <query_log>
        <database>system</database>
        <table>query_log</table>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
        <partition_by>toYYYYMM(event_date)</partition_by>
        <ttl>event_date + INTERVAL 30 DAY DELETE</ttl>
    </query_log>
</clickhouse>
```

### 2. Table Design Best Practices

```sql
-- 1. Always include TTL
CREATE TABLE user_events (
    event_date Date,
    user_id UInt64,
    event_type String,
    value Float64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id)
TTL event_date + INTERVAL 90 DAY DELETE  -- Auto cleanup
SETTINGS 
    index_granularity = 8192,
    compression = 'lz4hc(9)';

-- 2. Use appropriate data types
-- Bad: String for enums
-- Good: Enum8/16
event_type Enum8('click' = 1, 'view' = 2, 'purchase' = 3)

-- 3. Use LowCardinality for low-cardinality strings
city LowCardinality(String)
```

### 3. Operational Best Practices

- **Regular maintenance**: Run `OPTIMIZE TABLE ... FINAL` monthly
- **Monitor part count**: Alert if `system.parts` > 1000 per table
- **Keep 20% free space**: Always maintain buffer for merges
- **Separate disks**: Put system tables on fast, smaller SSD and user data on larger HDD
- **Regular TTL review**: Check TTL effectiveness quarterly

### 4. Emergency Response Plan

1. **Immediate** (0-5 minutes): Filesystem cleanup, delete system log parts
2. **Recovery** (5-15 minutes): Verify space, restart ClickHouse if needed
3. **Prevention** (15-30 minutes): Apply TTL, disable unnecessary logs
4. **Monitoring** (30+ minutes): Set up alerts, automate cleanup

---

## Expected Space Savings

| Optimization | Expected Space Reduction |
|--------------|-------------------------|
| System log TTL (7-30 days) | 50-70% of system tables |
| Disabling unnecessary logs | 20-40% of system tables |
| Better compression | 10-30% on user data |
| Regular TTL cleanup | 30-50% on time-series data |
| OS log management | 1-5GB depending on retention |

---

## Quick Reference Commands

```bash
# Check disk usage
df -h
du -sh /var/lib/clickhouse/* | sort -hr

# Largest tables
clickhouse-client --query "SELECT database, table, formatReadableSize(total_bytes) FROM system.tables WHERE total_bytes > 0 ORDER BY total_bytes DESC LIMIT 20"

# Force TTL cleanup
clickhouse-client --query "OPTIMIZE TABLE system.query_log FINAL"

# Clean detached parts
find /var/lib/clickhouse -type d -name "detached" -exec rm -rf {}/ \;

# Check TTL configuration
clickhouse-client --query "SELECT database, table, engine_full FROM system.tables WHERE engine_full LIKE '%TTL%'"

# Monitor growth
watch -n 60 'clickhouse-client --query "SELECT database, table, formatReadableSize(sum(bytes_on_disk)) FROM system.parts WHERE active = 1 GROUP BY database, table ORDER BY sum(bytes_on_disk) DESC"'
```

---

This guide provides both emergency recovery procedures and long-term optimization strategies. Implement the preventive measures first, then set up monitoring to avoid future emergencies.