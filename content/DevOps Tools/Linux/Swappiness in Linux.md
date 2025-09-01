---
longform:
  format: single
  title: "Swappiness in Linux "
title: Swappiness in Linux
---
Swappiness in Linux is a kernel parameter that determines how aggressively the system moves pages from physical RAM to swap space on disk. It's a value ranging from 0 to 100 in most distributions (some references cite 0–200, but 0–100 is most common). The default value is usually 60.

- **Lower swappiness (closer to 0):** The system avoids swapping, preferring to keep data in RAM as much as possible.
    
- **Higher swappiness (closer to 100):** The system swaps out inactive processes more readily, freeing up RAM for active tasks but potentially increasing disk I/O and latency.[phoenixnap+2](https://phoenixnap.com/kb/swappiness)
## Checking the Swappiness Value

You can check the current swappiness value using:

```
cat /proc/sys/vm/swappiness
```

or

```
sysctl vm.swappiness

```

## Changing the Swappiness Value

- **Temporarily (does not persist after reboot):**

```
sudo sysctl vm.swappiness=10
```

- **Persistently (survives reboot):**  

Edit `/etc/sysctl.conf` and add or change the line:

```
vm.swappiness = 10

```

Then, reboot or reload sysctl settings.

## Best Practices

- For general desktops, the default value (60) is usually fine.

- For database servers or workloads sensitive to swapping, a lower value (5–10, or even 1) is recommended to reduce the risk of performance degradation due to excessive swapping.[ibm+2](https://www.ibm.com/docs/en/license-metric-tool/9.2.0?topic=tdad-configuring-swappiness-in-linux-hosting-db2-database-server)

Swappiness tuning can help optimize the balance between RAM and swap space usage depending on your workload and hardware configuration.