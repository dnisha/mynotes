---
longform:
  format: single
  title: Shell Scripting Interview Questions
title: Shell Scripting Interview Questions
---
#### **a) Disk Usage Monitor**

Write a script that checks disk usage and sends an alert if usage exceeds 90%.

```
#!/bin/bash

THRESHOLD=90
CURRENT_USAGE=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')

if [ "$CURRENT_USAGE" -gt "$THRESHOLD" ]; then
    echo "WARNING: Disk usage is at ${CURRENT_USAGE}%!" >&2
    exit 1
else
    echo "Disk usage is normal: ${CURRENT_USAGE}%"
    exit 0
fi

```

#### **b) Process Checker**

Check if a critical process (e.g., `nginx`) is running, and restart it if not.

```
#!/bin/bash

PROCESS_NAME="nginx"

if ! pgrep -x "$PROCESS_NAME" > /dev/null; then
    echo "$PROCESS_NAME is not running. Restarting..." >&2
    systemctl restart "$PROCESS_NAME"
else
    echo "$PROCESS_NAME is running."
fi
```

#### **c) Network Connectivity Check**

Test connectivity to a list of servers and log failures.

```
#!/bin/bash

SERVERS=("google.com" "github.com" "example.com")
LOG_FILE="network_check.log"

for server in "${SERVERS[@]}"; do
    if ping -c 1 "$server" &> /dev/null; then
        echo "$(date) - $server: OK" >> "$LOG_FILE"
    else
        echo "$(date) - $server: FAILED" >> "$LOG_FILE"
    fi
done

```