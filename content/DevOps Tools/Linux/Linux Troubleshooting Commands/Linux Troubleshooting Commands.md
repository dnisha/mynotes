---
longform:
  format: scenes
  title: Linux Troubleshooting Commands
  sceneFolder: /
  scenes: []
  ignoredFiles: []
title: Linux Troubleshooting Commands
---
### The Core Concept: I/O Bottlenecks

Before diving in, it's important to understand what we're looking for. A system can be slow not just because the CPU is maxed out, but often because the disks are struggling to keep up with read/write requests. When processes have to wait for slow disk operations, the whole system feels sluggish. These two tools help you identify if that's the case.

---

## 1. iostat

**iostat** (Input/Output Statistics) is a command-line tool for monitoring system input/output device loading by observing the time devices are active in relation to their average transfer rates. It's part of the `sysstat` package.

### What it's best for:
*   Getting a high-level, system-wide overview of disk performance.
*   Seeing aggregate read/write speeds (throughput), request queues, and utilization for each block device (e.g., `sda`, `nvme0n1`).
*   Often used for its CPU utilization report as well.

### Installation (if not already installed):
```bash
# On Debian/Ubuntu
sudo apt install sysstat

# On RHEL/CentOS/Fedora
sudo yum install sysstat    # or `sudo dnf install sysstat`
```

### Common Usage and Output Explanation:

#### Basic Command:
```bash
iostat
```
This will show a brief report since boot for CPU and devices.

#### Most Useful Command:
```bash
iostat -dx 2
```
*   `-d`: Display the device utilization report.
*   `-x`: Display **extended** statistics. This is crucial for detailed analysis.
*   `2`: Update the report every 2 seconds.

#### Sample Output:
```
Linux 5.15.0-... (hostname)     08/25/2023      _x86_64_        (4 CPU)

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
nvme0n1          0.12    5.63      3.20    166.89     0.00     1.12   0.00  16.60    0.58    1.21   0.01    26.66    29.64   0.20   0.11
sda              2.51    1.13    121.87     15.20     0.07     0.21   2.70  15.58    0.36    0.86   0.00    48.57    13.48   0.21   0.08
```

#### Key Columns to Interpret:
*   **`r/s`**, **`w/s`**: Read and write requests (I/O operations) per second.
*   **`rkB/s`**, **`wkB/s`**: Read and write throughput in kilobytes per second. This is the actual data transfer speed.
*   **`r_await`**, **`w_await`**: **Crucial!** The average time (in milliseconds) for read/write requests to be served. This includes time spent in the queue and time being serviced. **High values here (>10-20 ms for HDDs, >~2 ms for SSDs) indicate a stressed disk subsystem.**
*   **`aqu-sz`**: Average queue length. The number of requests waiting to be serviced. A consistently high value indicates a bottleneck.
*   **`%util`**: Percentage of time the device was busy processing requests. **A value consistently above 60-80% is a strong sign of an I/O bottleneck.**

---

## 2. iotop

**iotop** is a top-like utility for displaying real-time disk I/O. It shows I/O usage by processes and threads, making it easy to identify exactly which program is causing high disk I/O.

### What it's best for:
*   Identifying **which specific processes** are generating the most read/write activity.
*   Interactive, real-time monitoring.

### Installation:
```bash
# On Debian/Ubuntu
sudo apt install iotop

# On RHEL/CentOS/Fedora
sudo yum install iotop    # or `sudo dnf install iotop`
```

**Note:** It almost always requires `sudo` privileges to run.

### Common Usage:

#### Basic Command:
```bash
sudo iotop
```
This starts an interactive, real-time interface.

#### Useful Options:
*   `-o` or `--only`: Only show processes or threads actually doing I/O. **This is the most useful option!**
*   `-P` or `--processes`: Only show processes (not threads).
*   `-a` or `--accumulated`: Show accumulated I/O instead of bandwidth.

#### Example to only show active I/O processes:
```bash
sudo iotop -o
```

#### Sample Output:
```
Total DISK READ: 15.24 K/s | Total DISK WRITE: 0.00 B/s
Current DISK READ: 15.24 K/s | Current DISK WRITE: 0.00 B/s
    TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
  12345 be/4 root       15.24 K/s    0.00 B/s  0.00 %  1.23 % [kworker/u12:2+flush]
   6789 be/4 mysql       0.00 B/s    7.61 K/s  0.00 %  0.12 % mysqld --daemonize
```

#### Key Columns to Interpret:
*   **`DISK READ`**, **`DISK WRITE`**: The current data read/write speed for that process.
*   **`IO>`**: The I/O percentage of time the process/thread spent performing I/O. This is the most direct indicator of how "I/O hungry" a process is.
*   **`TID`**: Thread ID (use `-P` option to see PID instead for a cleaner view).
*   **`USER`** and **`COMMAND`**: Who is doing it and what command it is.

---

## Summary: iostat vs. iotop

| Feature | iostat | iotop |
| :--- | :--- | :--- |
| **Perspective** | **System-wide, device-level** | **Process-level** |
| **Best For** | Answering: "**Is there** a disk bottleneck? Which disk is slow?" | Answering: "**Which process** is causing the disk bottleneck?" |
| **Output** | Numerical statistics for each block device | Real-time, top-like list of processes |
| **Key Metrics** | `%util`, `r_await`, `w_await`, `aqu-sz` | `DISK READ`, `DISK WRITE`, `IO%` |
| **Analogy** | A weather report showing overall pressure systems. | A live camera showing individual cars on a highway. |

## Typical Troubleshooting Workflow

1.  You notice a system is slow.
2.  Run `iostat -dx 2` to check overall disk health.
    *   If `%util` is high and `r_await`/`w_await` are high, you have a confirmed I/O bottleneck.
3.  Immediately run `sudo iotop -o` in another terminal to find the culprit process(es).
4.  Investigate why that specific process is so I/O intensive (e.g., misconfigured database, runaway log writing, backup job, etc.).