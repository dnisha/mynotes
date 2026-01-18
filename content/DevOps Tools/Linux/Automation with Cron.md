---
longform:
  format: single
  title: Automation with Cron
title: Automation with Cron
---
## **Cron Jobs**

```bash

# Edit user's crontab
crontab -e                # Edit crontab
crontab -l                # List cron jobs
crontab -r                # Remove all cron jobs

# System cron directories
/etc/cron.hourly/
/etc/cron.daily/
/etc/cron.weekly/
/etc/cron.monthly/

```

## **Crontab Format**

```bash

* * * * * command_to_execute
│ │ │ │ │
│ │ │ │ └─ Day of week (0-7, Sunday=0/7)
│ │ │ └─── Month (1-12)
│ │ └───── Day of month (1-31)
│ └─────── Hour (0-23)
└───────── Minute (0-59)

```

#### **Examples**:

```bash

# Run every minute
* * * * * /path/to/script.sh

# Run at 2:30 AM daily
30 2 * * * /path/to/script.sh

# Run every Monday at 5 PM
0 17 * * 1 /path/to/script.sh

# Run every 10 minutes
*/10 * * * * /path/to/script.sh

```