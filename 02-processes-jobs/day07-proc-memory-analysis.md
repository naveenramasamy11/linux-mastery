# 🐧 /proc Memory Analysis — Linux Mastery

> **When a process is leaking, crashing, or OOM-killing, /proc is your ground truth — learn to read it and you'll never be guessing about memory again.**

## 📖 Concept

The Linux kernel exposes real-time process memory information through the `/proc` virtual filesystem. Unlike tools like `top` or `free`, which aggregate and estimate, `/proc/<pid>/` gives you raw kernel data — exactly what the kernel sees for each process. For production debugging on AWS EC2 instances, ECS tasks, or Kubernetes pods, this knowledge is indispensable.

Every running process has a directory at `/proc/<pid>/`. The most important memory-related files are `status`, `smaps`, `maps`, `statm`, and `mem`. Understanding the difference between **VSZ (Virtual Size)**, **RSS (Resident Set Size)**, **PSS (Proportional Set Size)**, and **USS (Unique Set Size)** is foundational — these numbers mean completely different things, and misreading them is a common source of incorrect capacity planning.

**VSZ** is the total virtual address space — it includes all mapped regions, even if pages haven't been faulted in yet. It's almost always dramatically higher than actual memory usage. **RSS** is the amount of physical RAM currently used, but it counts shared libraries multiple times — once per process that maps them. This makes RSS misleading for multi-process servers. **PSS** divides shared pages proportionally among all processes that map them, giving a more accurate per-process cost. **USS** only counts pages private and unique to that process — the memory that would actually be freed if the process died.

For OOM (Out-of-Memory) debugging and memory leak detection, you want PSS or USS. `/proc/<pid>/smaps_rollup` gives you a one-shot summary. `/proc/<pid>/smaps` gives you full per-mapping detail for deep dives.

On AWS, understanding `/proc` memory helps you right-size EC2 instances and ECS task memory limits. A common mistake is setting container `memory` limits based on RSS and then getting OOM-killed because the limit was too low.

---

## 💡 Real-World Use Cases

- **Container OOM debugging**: A Kubernetes pod keeps being OOM-killed even though `top` shows "only" 200MB RSS — using `smaps_rollup` reveals the actual USS is 400MB, explaining the OOM kills
- **Memory leak detection**: Tracking PSS growth over time per PID to identify which service is leaking
- **EC2 right-sizing**: Using `smaps` to measure actual memory usage of a daemon before choosing an instance type
- **Java heap analysis**: Reading `/proc/<pid>/status` to see `VmRSS` vs `VmHWM` (high water mark) for JVM tuning
- **Detecting shared memory bloat**: Using `maps` to identify anonymous mappings vs file-backed shared segments
- **Pre-OOM alerting**: Monitoring `/proc/meminfo` fields like `MemAvailable` in custom CloudWatch metrics

---

## 🔧 Commands & Examples

### /proc/meminfo — System-Wide Memory Overview

```bash
# Full system memory snapshot
cat /proc/meminfo

# Key fields explained:
# MemTotal:      Total physical RAM
# MemFree:       Completely unused RAM (not cache)
# MemAvailable:  RAM available for new processes (incl. reclaimable cache) — use THIS for pressure monitoring
# Buffers:       Raw disk block cache
# Cached:        Page cache (file data)
# SwapCached:    Swap that has been read back into RAM
# Dirty:         Pages waiting to be written to disk
# Writeback:     Pages currently being written
# AnonPages:     Non-file anonymous memory (heap, stack)
# Mapped:        Files mapped into memory (mmap)
# Shmem:         Shared memory (tmpfs, SysV shm)
# Slab:          Kernel slab allocator memory
# KReclaimable:  Slab memory the kernel can reclaim under pressure
# CommitLimit:   Maximum RAM+swap the kernel will commit
# Committed_AS:  Total VSZ committed across all processes

# Watch for memory pressure in real time
watch -n 1 'grep -E "MemAvailable|Dirty|AnonPages|Committed_AS" /proc/meminfo'

# One-liner: show free memory as percentage
python3 -c "
data = dict(line.split()[:2] for line in open('/proc/meminfo') if ':' in line)
total = int(data['MemTotal:'])
avail = int(data['MemAvailable:'])
print(f'Available: {avail}kB / {total}kB ({avail*100//total}%)')
"

# Parse MemAvailable for a CloudWatch custom metric
MEM_AVAIL=$(grep MemAvailable /proc/meminfo | awk '{print $2}')
aws cloudwatch put-metric-data \
    --namespace "Custom/Memory" \
    --metric-name "MemAvailableKB" \
    --value "$MEM_AVAIL" \
    --region us-east-1
```

### /proc/<pid>/status — Per-Process Memory Summary

```bash
# Get memory info for a process by name
PID=$(pgrep -f "my-app" | head -1)
cat /proc/$PID/status

# Key fields:
# VmPeak:   Peak virtual memory usage (high-water mark)
# VmSize:   Current virtual memory (VSZ)
# VmLck:    Locked (mlock) memory
# VmPin:    Pinned pages
# VmHWM:    Peak RSS (Resident Set Size high-water mark) — great for JVM tuning
# VmRSS:    Current RSS (physical RAM used — includes shared libs)
# RssAnon:  Anonymous RSS (heap, stack — this is what matters for memory pressure)
# RssFile:  File-backed RSS (shared libs, mmap files)
# RssShmem: Shared memory RSS
# VmData:   Data segment size (heap)
# VmStk:    Stack size
# VmExe:    Text (code) size
# VmLib:    Shared library size
# VmSwap:   Amount swapped out

# Extract specific fields
grep -E "VmRSS|VmHWM|RssAnon|VmSwap" /proc/$PID/status

# Monitor RSS growth over time for leak detection
while true; do
    RSS=$(grep VmRSS /proc/$PID/status | awk '{print $2}')
    ANON=$(grep RssAnon /proc/$PID/status | awk '{print $2}')
    echo "$(date +%T) VmRSS=${RSS}kB RssAnon=${ANON}kB"
    sleep 5
done

# Alert if RSS exceeds 2GB
RSS=$(grep VmRSS /proc/$PID/status | awk '{print $2}')
LIMIT=$((2 * 1024 * 1024))  # 2GB in kB
if [ "$RSS" -gt "$LIMIT" ]; then
    echo "ALERT: PID $PID RSS is ${RSS}kB — exceeds 2GB threshold"
fi
```

### /proc/<pid>/statm — Compact Memory Stats

```bash
# statm format: size  resident  shared  text  lib  data  dt
# All values in PAGES (usually 4096 bytes each)
cat /proc/$PID/statm

# Parse it properly
PAGESIZE=$(getconf PAGESIZE)
read VSZ RSS SHARED TEXT LIB DATA DT < /proc/$PID/statm
echo "VSZ:    $((VSZ * PAGESIZE / 1024 / 1024)) MB"
echo "RSS:    $((RSS * PAGESIZE / 1024 / 1024)) MB"
echo "Shared: $((SHARED * PAGESIZE / 1024 / 1024)) MB"
echo "Data:   $((DATA * PAGESIZE / 1024 / 1024)) MB"

# Quick script to show memory for all processes
#!/bin/bash
PAGESIZE=$(getconf PAGESIZE)
for pid in /proc/[0-9]*/statm; do
    p=${pid%/statm}; p=${p#/proc/}
    comm=$(cat /proc/$p/comm 2>/dev/null) || continue
    read vsz rss _ < $pid 2>/dev/null || continue
    echo "$p $comm $((rss * PAGESIZE / 1024))kB"
done | sort -k3 -rn | head -20
```

### /proc/<pid>/smaps_rollup — Best Single-File Memory View

```bash
# smaps_rollup = smaps aggregated into one block (Linux 4.14+)
# This is the BEST way to get accurate per-process memory usage
cat /proc/$PID/smaps_rollup

# Example output:
# Rss:              245760 kB   <- RSS (includes shared pages)
# Pss:              198432 kB   <- PSS (shared divided proportionally) — best for per-proc cost
# Pss_Anon:         181248 kB   <- Anonymous PSS
# Pss_File:          17184 kB   <- File-backed PSS
# Pss_Shmem:             0 kB   <- Shmem PSS
# Shared_Clean:      28672 kB   <- Shared pages not modified
# Shared_Dirty:          0 kB   <- Shared pages modified
# Private_Clean:     35840 kB   <- Private unmodified pages
# Private_Dirty:    181248 kB   <- Private modified pages (heap/stack) = USS roughly
# Referenced:       245760 kB
# Anonymous:        181248 kB
# Swap:                  0 kB

# Extract USS (unique private memory — freed if process dies)
USS=$(grep "Private_" /proc/$PID/smaps_rollup | awk '{sum += $2} END {print sum}')
PSS=$(grep "^Pss:" /proc/$PID/smaps_rollup | awk '{print $2}')
echo "PSS: ${PSS}kB  USS: ${USS}kB"

# Compare USS vs RSS to understand sharing impact
RSS=$(grep "^Rss:" /proc/$PID/smaps_rollup | awk '{print $2}')
echo "RSS=${RSS}kB  PSS=${PSS}kB  USS=${USS}kB"
echo "Shared overhead: $((RSS - PSS))kB"
```

### /proc/<pid>/smaps — Detailed Per-Mapping Breakdown

```bash
# smaps shows every memory mapping with full detail
cat /proc/$PID/smaps | head -100

# Format per mapping:
# address           perms  offset   dev   inode   path
# Size:             128 kB       <- Virtual size
# KernelPageSize:     4 kB
# MMUPageSize:        4 kB
# Rss:              128 kB       <- Physical pages present
# Pss:               64 kB       <- Proportional (if shared with 2 processes)
# Shared_Clean:     128 kB
# Private_Dirty:      0 kB
# ...

# Find the largest anonymous (heap) mapping
grep -A 20 "^[0-9a-f].*anon" /proc/$PID/smaps | grep -E "Size|Pss:" | \
    paste - - | sort -k2 -rn | head -10

# Show all file-backed mappings and their PSS
awk '/^[0-9a-f]/ {file=$NF} /^Pss:/ && file != "" && file != "[heap]" && file != "[stack]" \
    {print $2, file}' /proc/$PID/smaps | sort -rn | head -20

# Find total heap size (anonymous mappings)
grep -A5 "\[heap\]" /proc/$PID/smaps | grep "^Size:" | awk '{sum+=$2} END {print sum"kB"}'

# Detect large anonymous mappings (potential memory leaks)
awk '/^[0-9a-f]/ {anon=($NF == "")} /^Size:/ && anon {if($2 > 102400) print $2"kB anon mapping"}' \
    /proc/$PID/smaps | sort -rn | head -10
```

### /proc/<pid>/maps — Virtual Address Space Layout

```bash
# maps shows all virtual memory regions (no size/PSS details, just layout)
cat /proc/$PID/maps

# Format:
# start-end  perms  offset  dev  inode  pathname
# 55a3b2c00000-55a3b2c01000 r--p 00000000 08:01 1234 /usr/bin/myapp
# 7f8a1c000000-7f8a1c200000 rw-p 00000000 00:00 0    [heap]
# 7ffcf0000000-7ffcf2000000 rw-p 00000000 00:00 0    [stack]

# Count memory region types
awk '{print $NF}' /proc/$PID/maps | sort | uniq -c | sort -rn

# Find which shared libraries are loaded
grep "\.so" /proc/$PID/maps | awk '{print $NF}' | sort -u

# Identify deleted files still mapped (potential descriptor leak)
grep "(deleted)" /proc/$PID/maps

# Check for anonymous executable mappings (possible code injection)
grep " r-xp " /proc/$PID/maps | grep "^[0-9a-f].*00:00"
```

### OOM Killer Analysis

```bash
# Check OOM score (0-1000: higher = more likely to be killed)
cat /proc/$PID/oom_score

# Check OOM score adjustment (-1000 to +1000)
cat /proc/$PID/oom_score_adj

# Make a process less likely to be OOM-killed (protect critical services)
echo -500 > /proc/$PID/oom_score_adj   # Requires root

# Make a process more killable (sacrifice less important processes)
echo 500 > /proc/$PID/oom_score_adj

# View OOM kill log from kernel
dmesg | grep -A 20 "Out of memory"
journalctl -k | grep -A 20 "OOM killer"

# See what would be killed if OOM happened now (the process with highest badness)
for pid in /proc/[0-9]*/oom_score; do
    p=${pid%/oom_score}; p=${p#/proc/}
    score=$(cat $pid 2>/dev/null)
    comm=$(cat /proc/$p/comm 2>/dev/null)
    echo "$score $p $comm"
done | sort -rn | head -10

# In Kubernetes — OOM-killed container detection
kubectl describe pod <pod-name> | grep -A5 "OOMKilled\|Exit Code"
kubectl get events --field-selector reason=OOMKilling -A
```

### Memory Leak Detection Workflow

```bash
#!/bin/bash
# Track PSS/USS growth every 30 seconds for a suspect process

PID=$(pgrep -f "suspicious-app")
LOG=/tmp/mem-track-$PID.csv
echo "timestamp,pss_kb,uss_kb,rss_kb" > $LOG

while kill -0 $PID 2>/dev/null; do
    TS=$(date +%s)
    ROLLUP=/proc/$PID/smaps_rollup

    PSS=$(grep "^Pss:" $ROLLUP 2>/dev/null | awk '{print $2}')
    RSS=$(grep "^Rss:" $ROLLUP 2>/dev/null | awk '{print $2}')
    PRIV_CLEAN=$(grep "Private_Clean:" $ROLLUP 2>/dev/null | awk '{print $2}')
    PRIV_DIRTY=$(grep "Private_Dirty:" $ROLLUP 2>/dev/null | awk '{print $2}')
    USS=$((PRIV_CLEAN + PRIV_DIRTY))

    echo "$TS,$PSS,$USS,$RSS" >> $LOG
    echo "$(date +%T) PSS=${PSS}kB USS=${USS}kB RSS=${RSS}kB"
    sleep 30
done

echo "Process ended. Log: $LOG"
# Analyze with: awk -F, 'NR>1 {print $1, $2, $3}' $LOG | gnuplot ...
```

---

## ⚠️ Gotchas & Pro Tips

- **RSS is almost always misleading**: A Node.js process showing 500MB RSS might only have 200MB USS. RSS includes shared libc, libssl, and other libraries counted once per process. Always use PSS or USS for capacity planning and container memory limit-setting.

- **smaps_rollup requires Linux 4.14+**: On older kernels (some legacy EC2 AMIs), it won't exist. Fall back to parsing `/proc/<pid>/smaps` manually. Check: `uname -r`.

- **Reading smaps is not free**: `cat /proc/$PID/smaps` acquires a lock on the process's memory descriptor. On processes with thousands of mappings (Java, Node.js), this can take 10–50ms and cause brief latency spikes. Use `smaps_rollup` instead for monitoring; it's faster.

- **Kernel memory (slab, dentry cache) doesn't show in /proc/<pid>**: If `free` shows low memory but all process RSS sums are modest, the memory is in kernel slab cache. Check `slabtop` and `/proc/slabinfo`. On heavily-loaded web servers, dentry and inode caches can consume GBs.

- **MemAvailable vs MemFree**: Never alert on `MemFree` — it will always be low because Linux uses all free RAM for page cache. Alert on `MemAvailable`, which includes reclaimable cache and is the true measure of memory pressure.

- **Container OOM uses cgroup limit, not /proc/meminfo**: In a Kubernetes pod or Docker container, the OOM killer fires when RSS+swap exceeds the cgroup's `memory.limit_in_bytes`, not the host's total RAM. Check `cat /sys/fs/cgroup/memory/memory.limit_in_bytes` inside the container.

- **Transparent Huge Pages bloat RSS**: If THP is enabled and the process uses large anonymous mappings, the kernel may allocate 2MB pages instead of 4KB pages. This can cause RSS to jump in 2MB increments and appear to spike suddenly. Check: `cat /proc/$PID/smaps | grep THPeligible`.

- **Zombie processes still hold /proc entries**: A zombie (Z state) process still has a `/proc/<pid>/` entry but has released all memory. Its status file will show 0 for all VmXxx fields. The process slot is held until the parent calls `wait()`.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
