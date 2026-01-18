---
longform:
  format: single
  title: Process Management
title: Process Management
---
## 📌 1. What is a Process?

A **process** is a running instance of a program. Each process has:

- A unique **PID** (Process ID)
    
- A **parent process**
    
- Assigned system **resources** (CPU, memory)
    
- A **state** (running, sleeping, stopped, zombie)
    

---

## ⚙️ 2. Key Process Commands

### 🔍 Viewing Processes

|Command|Description|
|---|---|
|`ps aux`|Shows all processes|
|`top`|Live, real-time process view|
|`htop`|Enhanced version of top with UI|
|`pstree`|Visual tree of processes|

### 🔫 Controlling Processes

|Command|Description|
|---|---|
|`kill PID`|Sends signal to terminate a process|
|`kill -9 PID`|Force kills a process (SIGKILL)|
|`killall name`|Kill all processes by name|
|`nice` / `renice`|Adjust process priority|

---

## 🧠 3. Daemon Processes

- **Daemons** run in the background, detached from the terminal.
    
- Usually started at **boot** by the **init system** (e.g., systemd).
    
- Examples: `sshd`, `nginx`, `crond`
    

### Characteristics:

- No controlling terminal
    
- Long-running and persistent
    
- Usually ends with a `d` (e.g., `sshd`)
    

---

## 🛠️ 4. Managing Services (Daemons) with systemd

Use `systemctl` for managing systemd services:

|Command|Purpose|
|---|---|
|`systemctl status nginx`|Check status of nginx|
|`systemctl start nginx`|Start nginx|
|`systemctl stop nginx`|Stop nginx|
|`systemctl restart nginx`|Restart nginx|
|`systemctl enable nginx`|Enable on boot|
|`systemctl disable nginx`|Disable on boot|
|`journalctl -u nginx`|View logs for nginx|

---

## 📁 5. Process States (Common)

- **R** – Running
    
- **S** – Sleeping
    
- **Z** – Zombie
    
- **T** – Stopped
    
- **D** – Waiting (uninterruptible sleep)
    

Use `ps -eo pid,stat,cmd` to see process states.

---

## Background/Foreground Jobs

```bash

command &                 # Run in background
Ctrl+Z                   # Suspend foreground job
bg                       # Resume in background
fg                       # Bring to foreground
jobs                     # List background jobs

```