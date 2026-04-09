# 🐧 /proc & /sys Deep Dive for Performance Analysis — Linux Mastery

> **`/proc` and `/sys` are the kernel's live dashboard — mastering them lets you diagnose CPU stalls, memory pressure, network saturation, and disk bottlenecks without installing a single monitoring agent.**

## 📖 Concept

**`/proc`** is a virtual filesystem (procfs) that exposes kernel data structures as files. It was originally designed for per-process information (hence the name) but evolved into the primary interface for global kernel state. Nothing in `/proc` is stored on disk — every read triggers a kernel function that generates the content on-the-fly. This means reads are always consistent and current, but also that some files (like `/proc/net/tcp`) can be expensive to read under high connection counts.

**`/sys`** (sysfs) is the newer, more structured kernel interface, introduced in Linux 2.6. While `/proc` grew organically and contains a mix of formats, `/sys` has a strict one-value-per-file philosophy. It's organized around kernel object model: devices, drivers, buses, block devices, and their attributes all appear as directories and files. This is where you tune I/O schedulers, CPU frequency governors, network device parameters, and kernel subsystem behavior at runtime.

The key insight for production use: these filesystems are your **zero-overhead telemetry**. No sampling, no agents, no overhead beyond what the kernel already tracks internally. When your monitoring stack is down and an incident is in progress, `/proc` and `/sys` are always there.

---

## 💡 Real-World Use Cases

- **CPU performance investigation:** `/proc/schedstat` and `/proc/stat` reveal scheduler queue depths and CPU steal time (critical for EC2 noisy-neighbor diagnosis).
- **Memory pressure analysis:** `/proc/meminfo` + `/proc/vmstat` reveal swap pressure, page fault rates, and OOM conditions before `free` or `top` can show it.
- **Network buffer saturation:** `/proc/net/softnet_stat` shows dropped packets due to softirq budget exhaustion — invisible in `netstat` or `ss`.
- **Disk I/O attribution:** `/proc/diskstats` → per-device queue depth and latency without `iostat`.
- **Container resource tuning:** `/sys/fs/cgroup/` exposes live cgroup accounting — the ground truth for what K8s resource limits are actually enforcing.

---

## 🔧 Commands & Examples

### /proc/cpuinfo — CPU Deep Dive

```bash
# Count physical CPUs (sockets)
grep "physical id" /proc/cpuinfo | sort -u | wc -l

# Cores per socket
grep "cpu cores" /proc/cpuinfo | head -1 | awk '{print $NF}'

# Logical CPUs (threads)
nproc
grep -c "^processor" /proc/cpuinfo

# CPU flags — check for virtualization, AES-NI, AVX etc
grep "^flags" /proc/cpuinfo | head -1 | tr ' ' '\n' | grep -E "vmx|svm|aes|avx|sse"
# vmx = Intel VT-x (hardware virtualization)
# aes = AES-NI hardware acceleration
# avx2 = AVX2 SIMD (important for ML workloads)

# CPU model and frequency
grep "model name\|cpu MHz\|cache size" /proc/cpuinfo | sort -u

# NUMA topology
cat /proc/cpuinfo | grep "physical id\|core id" | paste - -
```

### /proc/meminfo — Memory Analysis

```bash
# Full memory breakdown
cat /proc/meminfo

# Key fields explained:
# MemTotal      — Total physical RAM
# MemFree       — Completely unused RAM
# MemAvailable  — RAM available without swapping (use THIS, not MemFree)
# Buffers       — Kernel buffer cache (filesystem metadata)
# Cached        — Page cache (file contents)
# SwapCached    — Pages in both swap and RAM (being swapped back in)
# Dirty         — Pages modified but not yet written to disk
# Writeback     — Pages currently being written to disk
# AnonPages     — Anonymous (non-file-backed) pages (heap, stack)
# Mapped        — Files mapped into memory (shared libraries, mmap)
# Shmem         — Shared memory (includes tmpfs, /dev/shm)
# Slab          — Kernel slab allocator usage
# SReclaimable  — Slab pages that can be reclaimed under pressure
# SUnreclaim    — Slab pages that cannot be reclaimed
# HugePages_Total/Free/Rsvd — Huge pages for databases (Oracle, Postgres)
# DirectMap     — TLB coverage (important for performance)

# Quick memory health check
awk '/MemTotal|MemAvailable|SwapTotal|SwapFree|Dirty|Writeback/{print}' /proc/meminfo

# Check for memory pressure (available < 10% of total)
awk '/MemTotal/{total=$2} /MemAvailable/{avail=$2} END{
    pct=avail/total*100;
    printf "Memory Available: %.1f%%\n", pct;
    if(pct < 10) print "WARNING: Memory pressure!"
}' /proc/meminfo

# Monitor dirty pages (high dirty = write stall risk)
watch -n1 'grep -E "^Dirty|^Writeback" /proc/meminfo'
```

### /proc/vmstat — Virtual Memory Statistics

```bash
# Show all vmstat counters (cumulative since boot)
cat /proc/vmstat

# Key counters to watch:
# pgfault         — minor page faults (page already in memory)
# pgmajfault      — MAJOR page faults (had to read from disk!) — this is your I/O miss rate
# pgpgin/pgpgout  — pages paged in/out
# pswpin/pswpout  — swap pages in/out (non-zero pswpin = you're in trouble)
# oom_kill        — OOM killer invocations (alert on any non-zero delta!)
# numa_hit/miss   — NUMA allocation hit rate (miss = cross-NUMA latency)
# nr_dirty        — current dirty pages
# nr_writeback    — pages currently being written

# Watch for swap activity (rate of change)
watch -n2 'awk "/pswpin|pswpout|pgmajfault|oom_kill/{print}" /proc/vmstat'

# One-liner: check if OOM killer fired since boot
awk '/oom_kill/{if($2>0) print "OOM kills:", $2, "- INVESTIGATE!"; else print "No OOM kills"}' /proc/vmstat
```

### /proc/net/ — Network Internals

```bash
# /proc/net/softnet_stat — per-CPU network receive stats
# Columns: total, dropped, time_squeeze, [0 0 0], cpu_collision, received_rps
cat /proc/net/softnet_stat
# Each line = one CPU
# Column 2 (dropped) > 0 = packets dropped because backlog full
# Column 3 (time_squeeze) = softirq budget exceeded (increase net.core.netdev_budget)

# Parse softnet_stat
awk 'BEGIN{cpu=0} {
    dropped=strtonum("0x"$2);
    squeeze=strtonum("0x"$3);
    if(dropped>0 || squeeze>0)
        printf "CPU%d: dropped=%d squeeze=%d\n", cpu, dropped, squeeze;
    cpu++
}' /proc/net/softnet_stat

# /proc/net/dev — per-interface traffic counters
cat /proc/net/dev
# More readable:
awk 'NR>2{printf "%-10s RX:%-12s TX:%-12s\n", $1, $2, $10}' /proc/net/dev

# /proc/net/sockstat — socket summary
cat /proc/net/sockstat
# TCP: inuse 234 orphan 5 tw 12 alloc 280 mem 45
# inuse = established+listen, tw = TIME_WAIT, orphan = FIN-WAIT (watch for leaks)

# /proc/net/tcp — all TCP connections (hex format)
# Column 4: state (01=ESTAB, 0A=LISTEN, 06=TIME_WAIT...)
cat /proc/net/tcp | awk '{print $4}' | sort | uniq -c | sort -rn
```

### /proc/diskstats — Disk I/O Deep Dive

```bash
# Format: major minor device reads_c reads_m sectors_r ms_r writes_c writes_m sectors_w ms_w io_progress ms_io ms_io_w
cat /proc/diskstats | grep -v "loop\|ram"

# Calculate disk utilization % (requires two samples)
disk_util() {
    local dev=${1:-sda}
    local prev_ms=$(awk "/$dev /{print \$13}" /proc/diskstats)
    sleep 1
    local curr_ms=$(awk "/$dev /{print \$13}" /proc/diskstats)
    echo "Disk $dev utilization: $((curr_ms - prev_ms))%"
}
disk_util xvda

# Average I/O latency
# Formula: ms_io / (reads + writes) = avg latency per op
awk '/xvda /{
    reads=$4; writes=$8; ms=$13;
    if((reads+writes)>0)
        printf "Avg I/O latency: %.2f ms\n", ms/(reads+writes)
}' /proc/diskstats
```

### /sys — Runtime Tuning

```bash
# I/O Scheduler per device (critical for SSDs/NVMe)
cat /sys/block/xvda/queue/scheduler
# [none] mq-deadline kyber bfq
# none = best for NVMe, mq-deadline = best for HDDs

# Change scheduler at runtime
echo mq-deadline > /sys/block/xvda/queue/scheduler

# Read-ahead tuning (reduce for random I/O, increase for sequential)
cat /sys/block/xvda/queue/read_ahead_kb   # default 128
echo 0 > /sys/block/xvda/queue/read_ahead_kb   # Disable for DB random I/O
echo 2048 > /sys/block/xvda/queue/read_ahead_kb  # Increase for streaming

# NVMe queue depth
cat /sys/block/nvme0n1/queue/nr_requests   # default 1023
echo 2048 > /sys/block/nvme0n1/queue/nr_requests

# CPU frequency governor (important for latency-sensitive workloads)
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# powersave  ← bad for latency-critical apps!
echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Set all CPUs to performance governor
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance > "$cpu"
done

# Network device transmit queue length
cat /sys/class/net/eth0/tx_queue_len
echo 10000 > /sys/class/net/eth0/tx_queue_len   # Default 1000

# Transparent Huge Pages (THP) — disable for databases
cat /sys/kernel/mm/transparent_hugepage/enabled
# always [madvise] never
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

### /proc/PID/ — Per-Process Analysis

```bash
# Given PID of a suspect process
PID=12345

# Memory map with sizes
cat /proc/$PID/smaps | awk '/^Size:/{sum+=$2} END{print "Total mapped:", sum, "kB"}'

# File descriptors
ls -la /proc/$PID/fd | wc -l   # Total open FDs
ls -la /proc/$PID/fd           # What they point to

# Network connections for this process
cat /proc/$PID/net/tcp | wc -l

# CPU and memory stats
cat /proc/$PID/stat | awk '{
    print "PID:", $1, "State:", $3, "User CPU ticks:", $14, "Sys CPU ticks:", $15
}'

# Current syscall (what is it doing RIGHT NOW?)
cat /proc/$PID/syscall
# 4 0x3 0x7f1234 0x100 0 0 0 0x7ffe1234 0x7f5678
# First column is syscall number: 4 = write

# Memory maps
cat /proc/$PID/maps | head -20

# Environment variables
strings /proc/$PID/environ | sort
# Useful for: confirming env vars in containers, debugging config injection

# Limits (ulimits)
cat /proc/$PID/limits
```

---

## ⚠️ Gotchas & Pro Tips

- **`/proc/net/tcp` is expensive at scale.** Reading it acquires a kernel lock and iterates all sockets. On a server with 100,000+ connections, this can take seconds and cause latency spikes. Use `ss --no-header -t state established | wc -l` instead (uses netlink, much faster).
- **`/proc/meminfo`'s `MemFree` is misleading.** The kernel aggressively uses RAM for caches. Always use `MemAvailable` to know how much RAM is actually free for new applications.
- **`/sys` writes are NOT persistent.** Changes survive until reboot. Persist via `sysctl` entries in `/etc/sysctl.d/` or `udev` rules for block device settings.
- **`/proc/vmstat` counters are cumulative.** To get rates, sample twice and subtract. `vmstat 1` does this automatically and is the easier tool for real-time monitoring.
- **Pro tip — one-liner OOM hunt:**

```bash
# Find processes closest to OOM kill threshold
for pid in /proc/[0-9]*/oom_score; do
    score=$(cat $pid 2>/dev/null)
    proc=$(cat ${pid%oom_score}comm 2>/dev/null)
    echo "$score $proc"
done | sort -rn | head -10
```

- **`/sys/kernel/debug/` (debugfs) goes even deeper** — tracing, scheduler stats, power management. Mount with `mount -t debugfs none /sys/kernel/debug`. Used extensively by `perf`, `ftrace`, and `bpftrace`.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
