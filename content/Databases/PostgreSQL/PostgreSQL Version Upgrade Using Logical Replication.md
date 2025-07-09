---
longform:
  format: single
  title: PostgreSQL Version Upgrade Using Logical Replication
title: PostgreSQL Version Upgrade Using Logical Replication
---
# PostgreSQL Version Upgrade Using Logical Replication

This article explains how to perform PostgreSQL version upgrades with minimal downtime using logical replication. Here are the key points:

## Logical vs Physical Replication

- **Logical replication**: Selective table replication, standby can accept writes, can replicate between different versions
- **Physical (streaming) replication**: Block-level replication, replicates entire databases, standby is read-only, requires same major version

## Two Methods for Logical Replication

1. **Built-in logical replication** (PostgreSQL 10+ to 11)
   - Uses publish/subscribe model
   - Publisher sends changes, subscriber receives them
   - Can replicate selected tables or all tables
   - Requires primary keys or replica identity for UPDATE/DELETE operations

2. **pglogical extension** (PostgreSQL 9.4+ to 11)
   - Works with older PostgreSQL versions
   - Similar concept but implemented as an extension
   - Also requires primary keys for tables being replicated

## Implementation Steps

### For PostgreSQL 10/11 built-in replication:

1. On publisher:
   ```sql
   CREATE PUBLICATION percpub FOR ALL TABLES;  -- or specific tables
   ```

2. On subscriber:
   ```bash
   pg_dump -h publisher -p 5432 -d dbname -Fc -s -U postgres | pg_restore -d dbname -h subscriber -p 5432 -U postgres
   ```
   
   ```sql
   CREATE SUBSCRIPTION percsub CONNECTION 'host=publisher dbname=dbname user=postgres password=secret port=5432' PUBLICATION percpub;
   ```

### For PostgreSQL 9.4+ using pglogical:

1. On publisher (9.4):
   ```sql
   CREATE EXTENSION pglogical_origin;
   CREATE EXTENSION pglogical;
   SELECT pglogical.create_node(node_name := 'provider1', dsn := 'host=publisher port=5432 dbname=dbname');
   ```

2. On subscriber (11):
   ```sql
   CREATE EXTENSION pglogical;
   SELECT pglogical.create_node(node_name := 'subscriber1', dsn := 'host=subscriber port=5432 dbname=dbname');
   SELECT pglogical.create_subscription(subscription_name := 'sub1', provider_dsn := 'host=publisher port=5432 dbname=dbname');
   ```

## Important Considerations

- Tables must have primary keys or replica identity for UPDATE/DELETE operations
- Schema must exist on subscriber before creating subscription
- Initial data can be copied automatically or manually
- Monitor replication using `pg_stat_replication` or pglogical status tables

## CDC (Change Data Capture) with pglogical

Regarding your question about copying old data with CDC using pglogical: Yes, pglogical can be used for CDC scenarios. When setting up the subscription, you can choose whether to copy existing data (using the `copy_data` parameter). The replication will then capture all subsequent changes (inserts, updates, deletes) from the publisher.

For CDC purposes, you might:
1. Set up replication without copying initial data (`copy_data = false`)
2. Use another method to load historical data
3. Then let pglogical capture only the changes since replication began

This approach is useful when you need to maintain a replica with only recent changes or when initial data loading is handled separately.