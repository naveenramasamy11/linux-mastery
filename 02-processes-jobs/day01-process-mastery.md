# ⚙️ Day 01 — Process Mastery: ps, top, signals & job control

> Every process tells a story. Learn to read it.

---

## What is a Process?

A process = a running instance of a program. Each has:
- **PID** (Process ID)
- **PPID** (Parent PID)
- **State** (running, sleeping, zombie, stopped)
- **File descriptors**, **memory maps**, **CPU/memory usage**

---

## ps — The Swiss Army Knife of Process Inspection

```bash
# Most useful invocations
ps aux                      # All processes, BSD style
ps -ef                      # All processes, full format (PPID visible)
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu   # Custom columns, sorted by CPU
ps -eo pid,user,comm,lstart --sort=lstart    # Sort by start time
ps -C nginx                 # Find process by name
ps -p 1234                  # Info on specific PID
ps --forest                 # Process tree (ASCII art)
ps -u www-data              # Processes by user

# Find zombie processes
ps aux | awk '$8=="Z"'

# Find all children of a PID
ps --ppid 1234
pstree -p 1234              # Visual tree of children
```

---

## top / htop — Live Process Monitor

```bash
# top interactive keys
top
  P   → Sort by CPU
  M   → Sort by Memory
  T   → Sort by Time
  k   → Kill a process (enter PID)
  r   → Renice a process
  1   → Toggle per-CPU stats
  f   → Field selector
  u   → Filter by user
  H   → Show threads
  q   → Quit

# htop (install: apt/yum install htop)
htop
  F2  → Setup / customize
  F4  → Filter by name
  F5  → Tree view
  F6  → Sort column
  F9  → Kill (choose signal)
  Space → Tag multiple processes

# Resource-specific monitoring
iotop -ao          # I/O per process (cumulative)
iftop              # Network per process
nethogs            # Network bandwidth by process
```

---

## Process States

```
R  Running (or runnable, waiting for CPU)
S  Sleeping (interruptible — waiting for I/O, signal)
D  Uninterruptible sleep (usually I/O — can't be killed!)
Z  Zombie (finished, waiting for parent to collect exit code)
T  Stopped (Ctrl+Z or SIGSTOP)
X  Dead
```

> ⚠️ **D state** is dangerous — often means NFS hangs, disk I/O stalls, or kernel bugs. `kill` doesn't work on D-state processes. You often need to reboot.

---

## Signals — Talking to Processes

```bash
# List all signals
kill -l

# Most important signals
kill -1  PID    # SIGHUP  — Reload config (nginx, sshd, etc.)
kill -2  PID    # SIGINT  — Ctrl+C (graceful interrupt)
kill -9  PID    # SIGKILL — Immediate kill (cannot be caught/ignored)
kill -15 PID    # SIGTERM — Graceful termination (default)
kill -19 PID    # SIGSTOP — Pause process (cannot be caught)
kill -18 PID    # SIGCONT — Continue paused process
kill -11 PID    # SIGSEGV — Segfault (usually sent by kernel)

# Kill by name
pkill nginx             # Kill all processes named nginx
pkill -u www-data       # Kill all processes of a user
killall -HUP nginx      # Send SIGHUP to all nginx processes

# Kill all processes matching a pattern
pgrep -la "python"      # List matching PIDs and names first
pkill -9 -f "python my_script.py"   # -f matches full command line

# Check if process is running
kill -0 $PID && echo "running" || echo "not running"
```

> 💡 **Rule:** Always try `-15` (SIGTERM) before `-9` (SIGKILL). Apps need time to flush buffers, close connections, release locks.

---

## Job Control — Background & Foreground

```bash
# Start a job in background
long-running-command &

# Suspend current foreground job
Ctrl+Z          # Sends SIGSTOP — job pauses

# List jobs
jobs -l         # -l shows PIDs too

# Resume in background
bg %1           # Resume job #1 in background
bg              # Resume last suspended job

# Bring to foreground
fg %1
fg              # Last job

# Disown — survive terminal close
long-cmd &
disown %1       # Remove from shell's job table (survives logout)

# nohup — run immune to hangups
nohup ./deploy.sh > deploy.log 2>&1 &

# Even better — run in the background AND get a PID
nohup python3 server.py &
echo $!         # PID of last background process
```

---

## /proc — The Process Oracle

Every running process has a directory at `/proc/<PID>/`:

```bash
PID=1234

cat /proc/$PID/cmdline | tr '\0' ' '   # Full command line
cat /proc/$PID/status                  # State, memory, threads
cat /proc/$PID/environ | tr '\0' '\n'  # Environment variables!
ls -la /proc/$PID/fd/                  # Open file descriptors
cat /proc/$PID/maps                    # Memory map
cat /proc/$PID/net/tcp                 # Network connections (hex)
cat /proc/$PID/limits                  # Process limits (ulimits)

# How long has a process been running?
ps -p $PID -o lstart=,etime=

# What files does a process have open?
lsof -p $PID
lsof -p $PID | grep -i deleted        # Files deleted but still open (disk leak!)

# What syscalls is it making RIGHT NOW?
strace -p $PID -c                      # Summary of syscalls
strace -p $PID -e trace=network        # Only network syscalls
```

---

## Process Priority — nice & renice

```bash
# nice value range: -20 (highest priority) to +19 (lowest)
# Default: 0

# Start with lower priority
nice -n 10 ./backup.sh

# Start with higher priority (needs root)
nice -n -10 ./critical-job.sh

# Change priority of running process
renice -n 5 -p 1234             # Lower priority
renice -n -5 -p 1234            # Higher priority (needs root)
renice -n 10 -u www-data        # All processes of a user

# Real-time scheduling
chrt -r 99 -p $PID              # Set real-time FIFO, priority 99
chrt -p $PID                    # Check scheduling policy
```

---

## Finding & Killing Runaway Processes

```bash
# Top CPU consumers
ps aux --sort=-%cpu | head -10

# Top memory consumers
ps aux --sort=-%mem | head -10

# Find processes using most memory
smem -r -k | head -10

# Find what's holding a port
lsof -i :8080
ss -tlnp | grep 8080
fuser 8080/tcp

# Kill process on a port
fuser -k 8080/tcp

# Zombie cleanup (kill parent to let kernel collect)
ps aux | awk '$8=="Z" {print $3}' | xargs kill -SIGCHLD
```

---

## cgroups — Resource Limits per Process Group

```bash
# Check if cgroups are mounted
mount | grep cgroup

# Current cgroup of a process
cat /proc/$PID/cgroup

# systemd cgroup management
systemctl set-property nginx.service MemoryLimit=512M
systemctl set-property nginx.service CPUQuota=50%

# Manual cgroup (v1)
cgcreate -g memory,cpu:/myapp
echo $((512*1024*1024)) > /sys/fs/cgroup/memory/myapp/memory.limit_in_bytes
cgexec -g memory,cpu:/myapp ./myapp
```

---

## ⚡ One-Liners

```bash
# Find the process holding the most file descriptors
ls /proc/*/fd | awk -F/ '{print $3}' | sort | uniq -c | sort -rn | head -5

# Watch a process in real time
watch -n 1 "ps aux | grep nginx"

# Alert when a process dies
while kill -0 $PID 2>/dev/null; do sleep 1; done; echo "Process $PID died!" | mail -s alert admin@example.com

# Count processes per user
ps aux | awk 'NR>1 {print $1}' | sort | uniq -c | sort -rn

# See process tree of entire system
pstree -ap

# Profile a command (time + resource usage)
/usr/bin/time -v ./script.sh
```

---

> **Next up:** [Day 02 — Filesystem Tricks & inode Magic](../03-filesystem-tricks/)
