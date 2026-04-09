# 🐧 lsof & /proc/PID Deep Dive — Linux Mastery

> **Every production incident involving a "deleted file still consuming disk" or a "port that won't release" is solved with lsof and /proc — know these tools cold.**

## 📖 Concept

`lsof` (list open files) is one of the most powerful diagnostic tools on Linux. The name is slightly misleading — on Linux *everything is a file*, so lsof shows you open regular files, directories, sockets (TCP/UDP/Unix), pipes, device nodes, memory-mapped files, and more. When a process holds an open file descriptor, lsof exposes the relationship between the process, the FD number, the inode, and the filesystem path.

The classic scenario: a log file is deleted with `rm` but the disk space doesn't free up. Why? Because a running process still holds an open FD pointing to the deleted inode. The file's name is gone from the directory, but its data remains on disk until the last FD is closed. `lsof` lets you find the offending process; sending it SIGHUP or restarting it closes the FD and releases the space. This pattern is extremely common on long-running Java apps, Elasticsearch clusters, and CloudWatch agent instances in AWS.

The `/proc/<PID>/` pseudo-filesystem is the kernel's live window into every running process. It's not on disk — the kernel generates this content on demand when you read it. Inside you find the process's memory maps, open file descriptors, command line arguments, environment variables, CPU and memory stats, current working directory, and much more. Tools like `ps`, `top`, `strace`, and `lsof` themselves are implemented by reading `/proc`.

Understanding `/proc/PID` directly gives you information no higher-level tool exposes — like exactly what environment variables a container process was started with, or what shared libraries a suspicious process has loaded.

---

## 💡 Real-World Use Cases

- Identify which process holds a deleted log file open (disk not freeing after log rotation)
- Find all processes with open connections to an RDS endpoint before a maintenance window
- Debug "address already in use" errors by pinpointing what owns a port, down to the PID and binary
- Inspect the full environment of a running Kubernetes pod's PID 1 without `kubectl exec` access
- Detect a compromise: find processes with open network sockets that don't correspond to known services

---

## 🔧 Commands & Examples

### lsof Basics

```bash
# All open files (can be slow — tens of thousands of lines)
lsof | head -50

# Open files for a specific PID
lsof -p 1234

# Open files for a specific command name
lsof -c nginx
lsof -c java

# Open files by a specific user
lsof -u naveen
lsof -u ^root    # all users EXCEPT root

# Files opened in a specific directory (recursive)
lsof +D /var/log

# Specific file — who has it open?
lsof /var/log/app/app.log
lsof /dev/sda1
```

### Network Connections with lsof

```bash
# All network files (TCP + UDP sockets)
lsof -i

# Only TCP
lsof -i TCP

# Only listening ports
lsof -i -s TCP:LISTEN

# Specific port
lsof -i :8080
lsof -i :443

# Specific remote host (find all connections to an RDS endpoint)
lsof -i @my-db.cluster-xxxx.us-east-1.rds.amazonaws.com

# All connections to a specific IP
lsof -i @10.0.1.50

# Show port numbers instead of service names (-P) and IP instead of hostname (-n)
lsof -i -nP

# Connections to port 5432 (Postgres) from any process
lsof -i TCP:5432 -nP
```

### The Classic "Deleted File Still Using Disk" Pattern

```bash
# Step 1: notice disk is full but du doesn't account for it
df -h /var/log
du -sh /var/log/*   # sizes don't add up to df usage

# Step 2: find deleted files still held open
lsof | grep '(deleted)'
# or:
lsof +L1    # list files with link count < 1 (deleted from dir but inode still open)

# Sample output:
# java    12345  app    7w   REG  8,1  2147483648     0 /var/log/app/app.log (deleted)

# Step 3: recover or release
# Option A — send SIGHUP to reload (if the app supports it)
kill -HUP 12345

# Option B — if you need to recover the content before releasing
# Read from the /proc FD directly
cat /proc/12345/fd/7 > /tmp/recovered.log

# Option C — truncate in-place (frees space without restarting)
> /proc/12345/fd/7   # redirect empty string to the fd path — truncates the file

# Step 4: if truly stuck, restart the process to close all FDs
systemctl restart myapp
```

### /proc/PID Structure

```bash
# List all files in a process's /proc entry
ls -la /proc/1234/

# Key entries:
# cmdline    — null-separated command line arguments
# environ    — null-separated environment variables
# exe        — symlink to the executable binary
# cwd        — symlink to current working directory
# fd/        — directory of symlinks to open file descriptors
# fdinfo/    — detailed info about each fd (offset, flags, etc.)
# maps       — memory-mapped regions
# smaps      — detailed per-region memory stats
# status     — human-readable process status
# stat       — machine-readable stats (used by ps, top)
# net/       — network stats visible to this process's netns
# mounts     — mount points visible to this process
# limits     — current resource limits (ulimits)
```

### Reading /proc/PID Entries

```bash
PID=1234

# Full command line (readable)
tr '\0' ' ' < /proc/$PID/cmdline; echo

# Full environment (readable)
tr '\0' '\n' < /proc/$PID/environ

# Find a specific env var for a running process (without kubectl exec)
tr '\0' '\n' < /proc/$PID/environ | grep ^DATABASE_URL

# What binary is the process running?
readlink /proc/$PID/exe

# Current working directory
readlink /proc/$PID/cwd

# List all open file descriptors
ls -la /proc/$PID/fd

# What file is FD 7 pointing to?
readlink /proc/$PID/fd/7

# Memory stats — RSS and VSZ
grep -E 'VmRSS|VmSize|VmPeak' /proc/$PID/status

# Current resource limits (max open files, max memory, etc.)
cat /proc/$PID/limits

# How many threads?
grep Threads /proc/$PID/status

# CPU and memory stats (raw — used by ps)
cat /proc/$PID/stat
```

### Memory Maps

```bash
# View memory-mapped regions (libraries, heap, stack, anon mappings)
cat /proc/$PID/maps

# Sample output columns: address, perms, offset, dev, inode, pathname
# 7f8a1c000000-7f8a1c200000 r--p 00000000 08:01 1234567  /usr/lib/x86_64-linux-gnu/libc.so.6

# Detailed memory usage per mapping (USS, PSS, RSS)
cat /proc/$PID/smaps | grep -A 15 'heap'

# Check if a process has loaded a specific library
grep libssl /proc/$PID/maps

# Total memory breakdown
awk '/^Rss:/{rss+=$2} /^Pss:/{pss+=$2} END{print "RSS="rss"kB PSS="pss"kB"}' /proc/$PID/smaps
```

### /proc/PID/net — Per-Process Network Namespace View

```bash
# TCP connections visible to this process (useful for containers)
cat /proc/$PID/net/tcp    # hex-encoded local/remote addresses
cat /proc/$PID/net/tcp6   # IPv6 version
cat /proc/$PID/net/udp

# Decode hex addresses (quick helper)
python3 -c "
import socket, struct
hex_addr = '0F02000A'  # little-endian hex IP from /proc/net/tcp
ip = socket.inet_ntoa(struct.pack('<I', int(hex_addr, 16)))
print(ip)
"

# Interface stats for this process's netns
cat /proc/$PID/net/dev
```

### Practical lsof + /proc Combinations

```bash
# Find all processes with more than 1000 open FDs (FD leak candidates)
for pid in /proc/[0-9]*/fd; do
  count=$(ls "$pid" 2>/dev/null | wc -l)
  if (( count > 1000 )); then
    echo "PID ${pid##/proc/} ${pid%%/fd}: $count open FDs"
  fi
done

# Who is connecting to my Redis? Show PID + process name
lsof -i TCP:6379 -nP | awk 'NR>1 {print $1, $2, $9}'

# Find all Java processes and their log files
lsof -c java | grep '\.log'

# Track a process's open files in real time
watch -n1 'lsof -p 1234 | wc -l'

# Check ulimit for a running process (max open files)
cat /proc/1234/limits | grep 'open files'

# Count open sockets per process — find socket leak
lsof -nP | awk '$5=="IPv4" || $5=="IPv6" {count[$1" "$2]++}
END{for(k in count) if(count[k]>50) print count[k], k}' | sort -rn | head -20
```

---

## ⚠️ Gotchas & Pro Tips

- **`lsof` requires root for full visibility:** As a regular user, lsof only shows your own processes' FDs. Run with `sudo` (or as root) to see all processes — especially important when debugging system services.
- **`lsof` is slow by default:** It resolves hostnames for every network connection. Use `-n` (no hostname lookup) and `-P` (no port name lookup) to make it 10-50x faster: `lsof -nP -i`.
- **`/proc/PID/environ` captures env at start time:** It shows what environment variables were set when the process launched. Changes via `export` inside the process after launch are NOT reflected here.
- **FD symlinks in `/proc/PID/fd/` work for reading:** Even if a file is deleted, you can still read (or write to) it via `/proc/$PID/fd/<N>` — this is how you recover deleted-but-open log content.
- **`lsof +D /path` is very slow:** It does a recursive scan. For large directories, use `lsof | grep /path` instead.
- **Container processes and `/proc`:** In a multi-tenant Kubernetes node, all container processes are visible in the host's `/proc`. This is intentional and how tools like `crictl`, `cadvisor`, and `kubelet` work — but it also means host-level access to `/proc` can expose container internals.

```bash
# Quick FD leak monitor — alert if any process exceeds threshold
THRESHOLD=500
for pid in /proc/[0-9]*/; do
  pid_num=$(basename "$pid")
  fd_count=$(ls /proc/$pid_num/fd 2>/dev/null | wc -l)
  if (( fd_count > THRESHOLD )); then
    name=$(cat /proc/$pid_num/comm 2>/dev/null)
    echo "WARN: PID $pid_num ($name) has $fd_count open FDs"
  fi
done
```

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
