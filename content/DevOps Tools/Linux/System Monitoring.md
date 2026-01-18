---
longform:
  format: single
  title: System Monitoring
title: System Monitoring
---
## **Performance Monitoring**

```bash

# top/htop - Already covered
# vmstat - Virtual Memory Statistics
vmstat 1                  # Update every second
vmstat -s                # Summary statistics

# iostat - I/O Statistics
iostat -x 1              # Extended stats every second
iostat -d sda            # Specific device

# Other useful tools
uptime                    # System uptime and load
free -h                  # Memory usage
lscpu                    # CPU information
lsblk                    # Block device information

```

## **Log Monitoring**

```bash

# System logs location
/var/log/syslog          # General system messages
/var/log/auth.log        # Authentication logs
/var/log/kern.log        # Kernel messages
/var/log/dmesg           # Boot messages

# View logs
dmesg | tail -20         # Recent kernel messages
tail -f /var/log/syslog  # Follow log in real-time
journalctl -xe           # Systemd journal
journalctl -u service_name  # Specific service logs

```