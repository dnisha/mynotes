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

These are Linux/Unix commands for **job control** - managing processes (jobs) in your terminal. Let me explain each:

## **& (ampersand)**
- **Example:** `sleep 60 &`
- Runs a command in the background immediately
- Returns a job ID and PID (process ID)
- You get your prompt back right away
```bash
$ sleep 300 &
[1] 12345  # [job ID] [process ID]
```

## **Ctrl+Z**
- **Keyboard shortcut** (not a typed command)
- Suspends/pauses the currently running foreground job
- Puts it in a stopped state and returns you to the prompt
- Useful for pausing long-running commands temporarily

## **bg (background)**
- Resumes a suspended/stopped job but runs it in the background
- **Example:** After Ctrl+Z, type `bg`
- You can also specify: `bg %1` (resume job #1)

## **fg (foreground)**
- Brings a background or suspended job to the foreground
- **Example:** `fg %2` (bring job #2 to foreground)
- Without arguments, brings the current job (marked with `+`)

## **jobs**
- Lists all jobs associated with your terminal session
- Shows job ID, status, and command
```bash
$ jobs
[1]-  Running    sleep 300 &
[2]+  Stopped    vim file.txt
```

## **Typical Workflow:**

1. **Start a long job in background:**
   ```bash
   $ find / -name "*.txt" > output.txt &
   ```

2. **Suspend a running job:**
   - Start `ping google.com` (runs continuously)
   - Press **Ctrl+Z** → job suspends

3. **Resume it in background:**
   ```bash
   $ bg  # or bg %1
   ```

4. **Check job status:**
   ```bash
   $ jobs
   [1]+  Running    ping google.com &
   ```

5. **Bring back to foreground when needed:**
   ```bash
   $ fg %1
   ```

## **Job Status Indicators:**
- `Running` - Active in background
- `Stopped` - Suspended (Ctrl+Z)
- `Done` - Completed
- `Terminated` - Killed