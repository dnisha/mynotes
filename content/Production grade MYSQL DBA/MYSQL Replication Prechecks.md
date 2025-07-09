---
longform:
  format: single
  title: MYSQL Replication Prechecks
title: MYSQL Replication Prechecks
---
# Binary Log Expiration

```
SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';
```

```
SHOW VARIABLES LIKE 'expire_logs_days';
```

`expire_logs_days = 0` means **binary logs are never automatically deleted** based on age.

# Check replication format

```
SHOW VARIABLES LIKE 'binlog_format';
```

|Value|Meaning|
|---|---|
|`STATEMENT`|Uses **SQL statements** in the binary log.|
|`ROW`|Logs **each changed row** explicitly.|
|`MIXED`|MySQL chooses `STATEMENT` or `ROW` automatically based on the context.
