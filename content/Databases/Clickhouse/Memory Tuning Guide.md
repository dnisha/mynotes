---
longform:
  format: single
  title: Memory Tuning Guide
title: Memory Tuning Guide
---
# ClickHouse Memory Tuning Guide

## Introduction
This document provides best practices for memory configuration in ClickHouse, particularly for resource-constrained environments. Proper memory tuning prevents out-of-memory errors while maintaining acceptable query performance.

## Server-Wide Memory Settings

### 1. max_server_memory_usage
- **Purpose**: Limits total memory ClickHouse can use
- **Recommended**: 60-80% of physical RAM
- **Example**: 
  ```xml
  <max_server_memory_usage>2576980377</max_server_memory_usage> <!-- 2.4GB -->
  ```

### 2. max_server_memory_usage_to_ram_ratio
- **Purpose**: Alternative to absolute value, ratio of total RAM
- **Recommended**: 0.6 for small servers (leaves room for OS)
- **Example**:
  ```xml
  <max_server_memory_usage_to_ram_ratio>0.6</max_server_memory_usage_to_ram_ratio>
  ```

## Query-Level Memory Limits

### 3. max_memory_usage
- **Purpose**: Maximum memory a single query can use
- **Recommended**: 25% of max_server_memory_usage
- **Example**:
  ```xml
  <max_memory_usage>1073741824</max_memory_usage> <!-- 1GB -->
  ```

### 4. max_memory_usage_for_user
- **Purpose**: Total memory all queries from one user can use
- **Recommended**: 50-75% of max_server_memory_usage
- **Example**:
  ```xml
  <max_memory_usage_for_user>2147483648</max_memory_usage_for_user> <!-- 2GB -->
  ```

## Concurrency Control

### 5. max_threads
- **Purpose**: Limits parallel query execution threads
- **Recommended**: Number of CPU cores for small servers
- **Example**:
  ```xml
  <max_threads>4</max_threads>
  ```

### 6. max_concurrent_queries
- **Purpose**: Limits simultaneous queries
- **Recommended**: 2-4x CPU cores for small servers
- **Example**:
  ```xml
  <max_concurrent_queries>8</max_concurrent_queries>
  ```

## Memory Overcommit Settings

### 7. memory_overcommit_ratio_denominator
- **Purpose**: Allows temporary memory overcommit (higher = less overcommit)
- **Recommended**: 4 (25% overcommit) for small servers
- **Example**:
  ```xml
  <memory_overcommit_ratio_denominator>4</memory_overcommit_ratio_denominator>
  ```

### 8. memory_usage_overcommit_max_wait_microseconds
- **Purpose**: How long to wait when memory is exceeded
- **Recommended**: 500ms for interactive workloads
- **Example**:
  ```xml
  <memory_usage_overcommit_max_wait_microseconds>500000</memory_usage_overcommit_max_wait_microseconds>
  ```

## Insert-Specific Settings

### 9. max_partitions_per_insert_block
- **Purpose**: Limits partitions in single INSERT
- **Recommended**: 50-100 for small servers
- **Example**:
  ```xml
  <max_partitions_per_insert_block>50</max_partitions_per_insert_block>
  ```

## Background Processing

### 10. background_pool_size
- **Purpose**: Threads for background merges and mutations
- **Recommended**: 4 for small servers
- **Example**:
  ```xml
  <background_pool_size>4</background_pool_size>
  ```

## Monitoring Memory Usage

Use these queries to monitor memory usage:

```sql
-- Current memory usage
SELECT * FROM system.metrics 
WHERE metric LIKE '%Memory%';

-- Memory-related events
SELECT event, value FROM system.events 
WHERE event LIKE '%Memory%' OR event LIKE '%Overcommit%';

-- Query memory usage
SELECT query_id, elapsed, memory_usage, query 
FROM system.processes 
ORDER BY memory_usage DESC 
LIMIT 10;
```

## Best Practices

1. **Start Conservative**: Begin with lower limits and increase only when needed
2. **Monitor Regularly**: Watch for memory-related events and query failures
3. **Prioritize Stability**: On small servers, stability is more important than peak performance
4. **Adjust for Workload**:
   - Analytical workloads: Higher per-query memory
   - High concurrency: Lower per-query, higher total limits
5. **Leave RAM for OS**: At least 1GB for the operating system on small servers

## Sample Configuration for 4GB Server

```xml
<yandex>
  <profiles>
    <default>
      <max_memory_usage>1073741824</max_memory_usage> <!-- 1GB -->
      <max_memory_usage_for_user>2147483648</max_memory_usage_for_user> <!-- 2GB -->
      <memory_overcommit_ratio_denominator>4</memory_overcommit_ratio_denominator>
      <memory_usage_overcommit_max_wait_microseconds>500000</memory_usage_overcommit_max_wait_microseconds>
      <max_threads>4</max_threads>
      <max_concurrent_queries>8</max_concurrent_queries>
      <max_partitions_per_insert_block>50</max_partitions_per_insert_block>
    </default>
  </profiles>
  
  <max_server_memory_usage>2576980377</max_server_memory_usage> <!-- 2.4GB -->
  <max_server_memory_usage_to_ram_ratio>0.6</max_server_memory_usage_to_ram_ratio>
  <background_pool_size>4</background_pool_size>
</yandex>
```

## Troubleshooting

**Symptoms of Memory Issues**:
- Frequent "Memory limit exceeded" errors
- System becomes unresponsive during queries
- High swap usage

**Solutions**:
1. Reduce `max_memory_usage` and `max_threads`
2. Add more restrictive WHERE clauses to queries
3. Consider simplifying complex queries
4. Increase server memory if consistently hitting limits