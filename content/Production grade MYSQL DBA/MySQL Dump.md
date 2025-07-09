---
longform:
  format: single
  title: " MySQL Dump"
title: MySQL Dump
---
# MySQL Dump Command Documentation
### 1. Schema Export (Structure Only)
```bash
mysqldump -h 172.16.1.6 -u DB_USERNAME -p'DB_PASSWORD' \
--skip-add-drop-table --set-gtid-purged=OFF --no-data --skip-set-charset\ DATABASE_NAME > database_schema.sql
```

### 2. Data Export (Background Process)
```bash
nohup mysqldump -h 172.16.1.6 -u DB_USERNAME --password='DB_PASSWORD' \
--databases DATABASE_NAME --no-create-info --single-transaction \
--max_allowed_packet=500M --insert-ignore --set-gtid-purged=OFF \
--skip-triggers --no-create-db 2>/dev/null > database_backup.sql &
```

## Security Recommendations

1. **For Password Security** (better approaches):

   ```bash
   # Option 1: Prompt for password (most secure)
   mysqldump -h 172.16.1.6 -u DB_USERNAME -p DATABASE_NAME
   # (will prompt for password)

   # Option 2: Use MySQL config file
   # Create ~/.my.cnf with:
   # [client]
   # user = DB_USERNAME
   # password = DB_PASSWORD
   # host = 172.16.1.6
   # Then run simply:
   mysqldump DATABASE_NAME
   ```

2. **For Specific Tables** (add to either command):

   ```bash
   DATABASE_NAME table1 table2 table3
   ```

## Example with Values

```bash
# Schema for 'inventory_db' (no data)
mysqldump -h 172.16.1.6 -u app_user -p'S3cur3P@ss!' \
--skip-add-drop-table --set-gtid-purged=OFF --no-data \
inventory_db products suppliers > inventory_schema.sql

# Data backup for specific tables (background)
nohup mysqldump -h 172.16.1.6 -u app_user --password='S3cur3P@ss!' \
inventory_db orders order_items --no-create-info --single-transaction \
--max_allowed_packet=500M --insert-ignore --set-gtid-purged=OFF \
--skip-triggers --no-create-db 2>/dev/null > orders_backup.sql &
```
