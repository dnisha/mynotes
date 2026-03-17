---
longform:
  format: single
  title: TimescaleDB Hypertable Conversion & Validation Guide
title: TimescaleDB Hypertable Conversion & Validation Guide
---
## 1. Objective

This document provides a standardized approach to:

- Identify tables eligible for hypertable conversion
- Validate schema readiness
- Convert tables to hypertables
- Verify correctness post-conversion
- Optimize for high-ingestion workloads

---

## 2. Pre-Conversion Compatibility Check

Run the following query to classify tables based on hypertable readiness:

```sql
WITH table_columns AS (
    SELECT
        table_schema,
        table_name,
        column_name,
        data_type
    FROM information_schema.columns
    WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
),
time_column_check AS (
    SELECT
        table_schema,
        table_name,
        array_agg(column_name) FILTER (
            WHERE data_type IN (
                'timestamp without time zone',
                'timestamp with time zone',
                'date'
            )
        ) AS time_columns
    FROM table_columns
    GROUP BY table_schema, table_name
),
primary_key_check AS (
    SELECT
        kcu.table_schema,
        kcu.table_name,
        array_agg(kcu.column_name) AS pk_columns
    FROM information_schema.table_constraints tc
    JOIN information_schema.key_column_usage kcu
      ON tc.constraint_name = kcu.constraint_name
      AND tc.table_schema = kcu.table_schema
    WHERE tc.constraint_type = 'PRIMARY KEY'
    GROUP BY kcu.table_schema, kcu.table_name
)
SELECT
    t.table_schema,
    t.table_name,
    COALESCE(array_to_string(t.time_columns, ', '), 'None') AS time_columns,
    COALESCE(array_to_string(pk.pk_columns, ', '), 'No PK') AS primary_key_columns,
    CASE
        WHEN t.time_columns IS NULL THEN 'Not Compatible'
        WHEN pk.pk_columns IS NULL OR pk.pk_columns && t.time_columns THEN 'Fully Compatible'
        ELSE 'Partially Compatible (PK missing time column)'
    END AS hypertable_compatibility
FROM time_column_check t
LEFT JOIN primary_key_check pk
  ON t.table_schema = pk.table_schema
  AND t.table_name = pk.table_name
ORDER BY hypertable_compatibility DESC, table_schema, table_name;
```

---

## 3. Mandatory Requirements

### 3.1 Time Column

- Table must contain at least one column of type:
    - `timestamp`
    - `timestamptz`
    - `date`
---

### 3.2 Primary Key

- Recommended: composite primary key including time column  
Example:

```sql
PRIMARY KEY (id, created_at)
```

- If missing:

```sql
ALTER TABLE table_name
ADD PRIMARY KEY (id, time_column);
```

---

### 3.3 Constraints & Limitations

|Constraint Type|Recommendation|
|---|---|
|Unique constraints|Must include time column|
|Foreign keys|Allowed but may impact ingestion|
|Triggers|Keep minimal|
|Partitioned tables|Not supported for conversion|

---

### 3.4 Table Type

- Must be a regular table (not already partitioned)
- Avoid UNLOGGED tables in production ingestion systems

---

## 4. High-Ingestion Design Guidelines

### 4.1 Chunk Interval Selection

|Ingestion Volume|Recommended Interval|
|---|---|
|< 1M rows/day|1 day|
|1M – 100M/day|1–6 hours|
|> 100M/day|5–30 minutes|

Example:

```sql
SELECT create_hypertable(
    'schema.table_name',
    'created_at',
    chunk_time_interval => INTERVAL '1 hour'
);
```

---

### 4.2 Indexing Strategy

- Keep indexes minimal
- Prefer time-based and composite indexes

```sql
CREATE INDEX ON table_name (time_column DESC);
CREATE INDEX ON table_name (dimension_column, time_column DESC);
```

---

### 4.3 Compression Strategy

```sql
ALTER TABLE table_name SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'dimension_column'
);

SELECT add_compression_policy('schema.table_name', INTERVAL '7 days');
```

---

### 4.4 Retention Policy

```sql
SELECT add_retention_policy('schema.table_name', INTERVAL '90 days');
```

---

## 5. Hypertable Conversion

### 5.1 Basic Conversion

```sql
SELECT create_hypertable(
    'schema.table_name',
    'time_column',
    if_not_exists => TRUE
);
```

---

### 5.2 Conversion with Data Migration

```sql
SELECT create_hypertable(
    'schema.table_name',
    'time_column',
    migrate_data => TRUE
);
```

---

## 6. Post-Conversion Validation

### 6.1 Hypertable Metadata

```sql
SELECT 
    hypertable_name,
    hypertable_schema,
    num_chunks
FROM timescaledb_information.hypertables 
WHERE hypertable_name = 'apnamart_daily_shopify_product_insights';
```

---

### 6.2 Chunk Distribution

```sql
SELECT 
    chunk_name,
    chunk_schema,
    range_start,
    range_end,
    is_compressed
FROM timescaledb_information.chunks 
WHERE hypertable_name = 'apnamart_daily_shopify_product_insights'
ORDER BY range_start;
```

---

### 6.3 Table Size

```sql
SELECT pg_size_pretty(
    pg_total_relation_size('manthan_prod.apnamart_daily_shopify_product_insights')
) AS total_size;
```

---

### 6.4 Detailed Size Analysis

```sql
SELECT * 
FROM hypertable_detailed_size('manthan_prod.apnamart_daily_shopify_product_insights');

SELECT * 
FROM chunks_detailed_size('manthan_prod.apnamart_daily_shopify_product_insights');
```

---

## 7. Performance Validation

### 7.1 Insert Throughput Test

```sql
INSERT INTO table_name
SELECT now(), generate_series(1,1000000);
```

Monitor:

- Insert latency
- CPU usage
- WAL generation

---

### 7.2 Chunk Creation Monitoring

```sql
SELECT count(*) 
FROM timescaledb_information.chunks 
WHERE hypertable_name = 'table_name';
```

---

### 7.3 Chunk Size Distribution

```sql
SELECT 
    c.range_start,
    c.range_end,
    pg_size_pretty(s.total_bytes)
FROM timescaledb_information.chunks c
JOIN chunks_detailed_size('schema.table_name') s
  ON c.chunk_name = s.chunk_name
ORDER BY s.total_bytes DESC;
```

---

## 8. Common Pitfalls

- Excessive indexing reduces ingestion throughput
- Incorrect chunk interval leads to performance degradation
- Missing time column in primary key causes constraint issues
- Lack of compression leads to rapid storage growth
- High update workloads are inefficient (prefer append-only design)

---

## 9. Advanced Optimization

### 9.1 Background Workers

```sql
SET timescaledb.max_background_workers = 16;
```

---

### 9.2 Continuous Aggregates

```sql
CREATE MATERIALIZED VIEW table_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time_column),
    count(*)
FROM table_name
GROUP BY 1;
```

---

### 9.3 Reorder Policy

```sql
SELECT add_reorder_policy('schema.table_name', 'index_name');
```

---