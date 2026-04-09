# 🐧 vmstat & Memory Performance Analysis — Linux Mastery

> **Memory pressure is the silent killer of Linux system performance — vmstat, free, and /proc/meminfo together paint the complete picture that CloudWatch metrics never show you.**

## 📖 Concept

Memory management in Linux is significantly more nuanced than "used vs free". The kernel aggressively uses available RAM as **page cache** (file data cached in memory), **slab cache** (kernel object caches), and **anonymous memory** (process heap/stack). A system showing "only 100MB free" but with 10GB of page cache is healthy — the page cache is immediately reclaimable. A system with 100MB free and zero cache is actually under pressure.

`vmstat` (virtual memory statistics) gives you a snapshot or continuous view of system activity: CPU states, memory statistics, swap usage, block I/O activity, interrupts, and context switches — all in a single, fast-to-read table. Unlike `top`, vmstat shows system-level aggregates rather than per-process data, making it ideal for detecting memory pressure, swap storms, high context-switch rates, and I/O bottlenecks at a glance.

The key memory performance indicators to watch: **si/so** (swap in/swap out — any non-zero value during normal operation is a warning), **buff/cache** (page cache size — healthy systems have lots), **free** (truly unused RAM — should be near zero on a healthy system, all spare RAM is cache), **wa** CPU (I/O wait — high values mean disk I/O is the bottleneck), and **cs** (context switches — very high rates can indicate scheduling overhead from too many threads).

On EC2 instances, memory is the most opaque metric — CloudWatch doesn't collect it by default (you need the CloudWatch agent). Understanding how to read kernel memory counters directly is essential for diagnosing memory-related performance issues without waiting for monitoring dashboards.

---

## 💡 Real-World Use Cases

- Diagnose an EC2 instance under JVM heap pressure: correlate high `so` (swap out) with GC pause spikes
- Detect a memory leak in a containerised service by watching RSS growth over time with vmstat and /proc/PID/status
- Identify slab cache bloat (dentries, inodes) consuming hundreds of GB on a file-heavy NFS server
- Tune `vm.swappiness` and `vm.dirty_ratio` parameters to optimise a database workload on EC2
- Set up a baseline memory performance capture script for incident post-mortems

---

## 🔧 Commands & Examples

### vmstat Basics

```bash
# One-shot snapshot (current statistics)
vmstat

# Output columns:
# procs:    r=runnable, b=blocked in uninterruptible sleep
# memory:   swpd=swap used, free=free RAM, buff=buffer cache, cache=page cache
# swap:     si=swap in (KB/s), so=swap out (KB/s)
# io:       bi=blocks in (from disk), bo=blocks out (to disk)
# system:   in=interrupts/s, cs=context switches/s
# cpu:      us=user, sy=system, id=idle, wa=I/O wait, st=steal (VM)

# Continuous output every 2 seconds
vmstat 2

# N samples every 2 seconds (then exit)
vmstat 2 10

# With timestamps
vmstat 2 -t

# Include slab memory in output
vmstat -m

# Disk statistics
vmstat -d

# Partition statistics
vmstat -p sda
vmstat -p nvme0n1p1
```

### Reading vmstat Output

```bash
# HEALTHY system example:
# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
#  1  0      0 1024000  85000 8500000   0    0     5    20  500 1200  5  2 92  1  0

# UNDER MEMORY PRESSURE example:
# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
#  3  5 2048000  12000   2000  85000  450  850  2500  1800 8000 25000 35 20 10 35  0

# Danger signs to watch:
# - swpd > 0 AND si/so > 0: active swapping — real memory pressure
# - free < 50MB AND cache < 200MB: nothing to reclaim
# - wa > 20: significant I/O wait — disk bottleneck
# - cs > 100000: excessive context switches — too many threads or lock contention
# - r > number of CPUs: processes waiting for CPU — CPU-bound or scheduling issue
# - b > 0: processes in uninterruptible sleep — usually waiting on I/O or locks
```

### free — Memory Overview

```bash
# Human-readable memory summary
free -h

# Sample output:
#               total        used        free      shared  buff/cache   available
# Mem:           15Gi       3.2Gi       1.1Gi       256Mi        11Gi        11Gi
# Swap:           2Gi       128Mi       1.9Gi

# Key column: "available" = free + reclaimable cache (what applications can actually use)
# This is more useful than "free" alone

# Show in megabytes
free -m

# Refresh every 2 seconds
free -h -s 2

# Interpreting:
# used = total - free - buff/cache
# available ≈ free + cache (cache the kernel CAN reclaim immediately)
# If available < 10% of total: investigate
# If swap used > 0: investigate what's using it
```

### /proc/meminfo — Detailed Memory Breakdown

```bash
# Full memory information
cat /proc/meminfo

# Key fields to monitor:
grep -E "MemTotal|MemFree|MemAvailable|Buffers|Cached|SwapCached|SwapTotal|SwapFree|Dirty|Writeback|AnonPages|Mapped|Shmem|Slab|SReclaimable|SUnreclaim|KernelStack|PageTables|Bounce|HugePages" /proc/meminfo

# Explanation of key fields:
# MemTotal:      Physical RAM total
# MemFree:       Completely unused RAM (low is OK if MemAvailable is high)
# MemAvailable:  Estimated available for new applications (free + reclaimable)
# Buffers:       Block device I/O buffers (block metadata cache)
# Cached:        Page cache (file content in RAM)
# SwapCached:    Pages moved back from swap but kept in swap too
# Dirty:         Pages modified but not yet written to disk — a large value means I/O lag
# Writeback:     Pages actively being written to disk
# AnonPages:     Anonymous memory (heap, stack, mmap without file backing)
# Slab:          Kernel slab cache total
# SReclaimable:  Slab cache that CAN be reclaimed (dentries, inodes)
# SUnreclaim:    Slab cache that CANNOT be reclaimed (in active use)

# Quick health check function
memory_health_check() {
  echo "=== Memory Health Check ==="
  awk '
    /MemTotal/    {total=$2}
    /MemAvailable/ {avail=$2}
    /SwapTotal/   {stotal=$2}
    /SwapFree/    {sfree=$2}
    /Dirty/       {dirty=$2}
    /SUnreclaim/  {sunreclaim=$2}
    END {
      printf "Total RAM:      %6d MB\n", total/1024
      printf "Available:      %6d MB (%.0f%%)\n", avail/1024, avail/total*100
      printf "Swap Used:      %6d MB\n", (stotal-sfree)/1024
      printf "Dirty Pages:    %6d MB\n", dirty/1024
      printf "Slab Unreclm:   %6d MB\n", sunreclaim/1024
    }' /proc/meminfo
}
memory_health_check
```

### Swap Analysis

```bash
# Check swap usage per process (who is swapped out)
# Method 1: using smaps
for pid in /proc/[0-9]*/; do
  pid_num=$(basename "$pid")
  swap=$(grep VmSwap /proc/$pid_num/status 2>/dev/null | awk '{print $2}')
  if [[ -n "$swap" ]] && (( swap > 0 )); then
    name=$(cat /proc/$pid_num/comm 2>/dev/null)
    echo "${swap}kB  PID:${pid_num}  ${name}"
  fi
done | sort -rn | head -20

# Method 2: smaps_rollup (Linux 4.14+, much faster)
for pid in /proc/[0-9]*/; do
  pid_num=$(basename "$pid")
  swap=$(awk '/^Swap:/{sum+=$2} END{print sum}' /proc/$pid_num/smaps_rollup 2>/dev/null)
  if [[ -n "$swap" ]] && (( swap > 0 )); then
    name=$(cat /proc/$pid_num/comm 2>/dev/null)
    echo "${swap}kB  PID:${pid_num}  ${name}"
  fi
done | sort -rn | head -10

# Check swap device usage
swapon -s
cat /proc/swaps

# Disable swap temporarily (for testing performance without swap)
swapoff -a

# Re-enable
swapon -a

# Create a swap file (useful on EC2 instances without swap partition)
dd if=/dev/zero of=/swapfile bs=1M count=4096   # 4GB swap file
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
# Persist: echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

### Slab Cache Analysis

```bash
# Show slab cache statistics
cat /proc/slabinfo | head -20
# or the cleaner tool:
slabtop -o     # one-shot output (no interactive mode)
slabtop        # interactive, sorted by size

# Identify top slab consumers
slabtop -s c -o | head -20   # sort by cache size

# Common large slab consumers:
# dentry        — directory entry cache (path lookup cache)
# inode_cache   — inode metadata cache
# ext4_inode_cache — ext4-specific inodes
# xfs_inode     — XFS inodes
# kmalloc-*     — generic kernel memory allocations
# sock_inode_cache — socket inodes

# Show with human-readable sizes
awk 'NR>2 {printf "%s %d MB\n", $1, ($3*$4)/1048576}' /proc/slabinfo | sort -t' ' -k2 -rn | head -10

# Force slab cache reclaim (writes dirty pages and drops caches — use carefully)
# sync                           — flush dirty pages first
# echo 1 > /proc/sys/vm/drop_caches  — drop page cache only
# echo 2 > /proc/sys/vm/drop_caches  — drop slab reclaimable
# echo 3 > /proc/sys/vm/drop_caches  — drop both
sync; echo 3 > /proc/sys/vm/drop_caches
# WARNING: causes a brief performance hit as caches rebuild — never in production under load
```

### Key Memory sysctl Tuning

```bash
# View all VM-related sysctl parameters
sysctl -a | grep vm\. | sort

# --- CRITICAL PARAMETERS ---

# vm.swappiness (0-200, default 60)
# How aggressively kernel swaps anonymous memory vs reclaiming page cache
# 0 = strongly prefer reclaiming cache before swapping
# 10 = good for database servers (PostgreSQL, MySQL)
# 60 = default (balanced)
# 100 = treat page cache and anonymous memory equally
sysctl vm.swappiness
sysctl -w vm.swappiness=10
echo 'vm.swappiness = 10' >> /etc/sysctl.d/99-memory.conf

# vm.dirty_ratio (default 20)
# Max % of RAM that can be dirty (pending write to disk) before processes BLOCK
# Lower = more frequent writes, less write stall risk
sysctl -w vm.dirty_ratio=15

# vm.dirty_background_ratio (default 10)
# % of RAM dirty before background flush starts
sysctl -w vm.dirty_background_ratio=5

# vm.dirty_expire_centisecs (default 3000 = 30 seconds)
# How old dirty data can be before it MUST be flushed
sysctl -w vm.dirty_expire_centisecs=1500

# vm.vfs_cache_pressure (default 100)
# Tendency to reclaim dentries/inodes vs anonymous memory
# Lower = keep more dentry/inode cache (good for inode-heavy workloads)
# Higher = reclaim cache more aggressively
sysctl -w vm.vfs_cache_pressure=50

# vm.overcommit_memory (default 0)
# 0 = heuristic overcommit (default)
# 1 = always allow overcommit (no OOM for malloc, but OOM killer active)
# 2 = never overcommit beyond committed_ratio
sysctl vm.overcommit_memory
# For Redis: set to 1 to avoid fork() failures during RDB saves

# Transparent Huge Pages (THP) — can help or hurt depending on workload
cat /sys/kernel/mm/transparent_hugepage/enabled
# For databases (Postgres, MongoDB, Redis): DISABLE THP to avoid latency spikes
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
# Persist via rc.local or systemd unit
```

### Memory Monitoring Scripts

```bash
#!/bin/bash
# memory-snapshot.sh — capture detailed memory state for incident post-mortems

OUTDIR="/tmp/mem-snapshot-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$OUTDIR"

echo "Capturing memory snapshot to $OUTDIR"

# vmstat history
vmstat 1 30 -t > "$OUTDIR/vmstat.txt" &

# /proc/meminfo
cp /proc/meminfo "$OUTDIR/meminfo.txt"

# Top memory consumers by RSS
ps aux --sort=-%mem | head -30 > "$OUTDIR/top-rss.txt"

# Swap users
{
  echo "PID SWAP(kB) COMMAND"
  for pid in /proc/[0-9]*/; do
    pid_num=$(basename "$pid")
    swap=$(grep VmSwap /proc/$pid_num/status 2>/dev/null | awk '{print $2}')
    [[ -n "$swap" ]] && (( swap > 0 )) && echo "$pid_num $swap $(cat /proc/$pid_num/comm 2>/dev/null)"
  done | sort -k2 -rn
} > "$OUTDIR/swap-users.txt"

# Slab top
slabtop -s c -o 2>/dev/null > "$OUTDIR/slabtop.txt"

# OOM score for top processes
ps aux --sort=-%mem | head -20 | awk '{print $2}' | tail -n+2 | while read pid; do
  oom_score=$(cat /proc/$pid/oom_score 2>/dev/null)
  name=$(cat /proc/$pid/comm 2>/dev/null)
  echo "PID=$pid comm=$name oom_score=$oom_score"
done > "$OUTDIR/oom-scores.txt"

wait   # wait for vmstat background job

echo "Done. Files saved to $OUTDIR"
ls -lh "$OUTDIR"
```

### Huge Pages Configuration

```bash
# Check current huge pages state
cat /proc/meminfo | grep Huge
# AnonHugePages:  2048 kB   (THP anonymous)
# HugePages_Total:     0
# HugePages_Free:      0
# Hugepagesize:     2048 kB

# Allocate static huge pages at boot (for Oracle DB, DPDK, etc.)
echo 512 > /proc/sys/vm/nr_hugepages    # 512 * 2MB = 1GB of hugepages
# Persist:
echo 'vm.nr_hugepages = 512' >> /etc/sysctl.d/99-hugepages.conf

# Mount hugetlbfs (required for some apps to use huge pages)
mkdir -p /mnt/hugepages
mount -t hugetlbfs hugetlbfs /mnt/hugepages
echo 'hugetlbfs /mnt/hugepages hugetlbfs defaults 0 0' >> /etc/fstab

# Use 1GB huge pages (requires CPU support and kernel boot param)
# Add to /etc/default/grub: GRUB_CMDLINE_LINUX="default_hugepagesz=1G hugepagesz=1G hugepages=16"
grep pdpe1gb /proc/cpuinfo   # check 1GB huge page CPU support

# Verify huge page allocation
cat /proc/meminfo | grep -E "HugePages|Hugepagesize"
```

---

## ⚠️ Gotchas & Pro Tips

- **"free" memory ≈ 0 is normal and healthy:** Linux uses all available RAM as page cache. The number to watch is `available` (from `free -h`), not `free`. If `available` is low, you have a real problem.
- **High `si`/`so` = emergency:** Any sustained swap I/O (`si`/`so` > 0 in vmstat) under normal load means you're running out of real memory and the system is degrading. Investigate immediately.
- **`wa` > 30% means disk is your bottleneck:** High I/O wait means processes are stalling waiting for disk. On AWS, check EBS burst credit balance and consider upgrading from gp2 to gp3 with higher provisioned IOPS.
- **Dirty page accumulation:** If `Dirty` in `/proc/meminfo` grows to hundreds of MB, you have a write buffering issue. Either the disk is too slow to keep up or `vm.dirty_ratio` is too high. Reduce `vm.dirty_background_ratio` to trigger earlier flushing.
- **`echo 3 > /proc/sys/vm/drop_caches` in production:** This is a sharp tool. Dropping caches causes an immediate performance hit as the kernel rebuilds them from disk. Use it only in controlled testing, never on a production system under load.
- **THP and databases:** Transparent Huge Pages cause random latency spikes (2ms+) in database workloads due to defragmentation. Always disable THP (`echo never > /sys/kernel/mm/transparent_hugepage/enabled`) on nodes running Postgres, MySQL, Redis, MongoDB, or Elasticsearch.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
