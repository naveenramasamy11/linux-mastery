# 🐧 iotop, iostat & I/O Performance Analysis — Linux Mastery

> **I/O bottlenecks are responsible for a disproportionate share of production performance problems — latency that looks like CPU or memory issues often traces back to a disk queue, and the tools to find it are already installed on every Linux system.**

## 📖 Concept

I/O performance problems are insidious because they manifest as high CPU wait (`%iowait` in `top`), slow application response times, or memory pressure — symptoms that initially point away from disk. Understanding the I/O stack helps: applications make syscalls (`read()`, `write()`), which pass through the VFS layer, the block layer (where I/O scheduling happens), device drivers, and finally the physical storage.

`iostat` is the primary tool for measuring I/O throughput, IOPS, and latency at the block device level. It reads from `/proc/diskstats` and presents the data in a human-readable format with configurable intervals. The key metrics are: `r/s` and `w/s` (read/write IOPS), `rkB/s` and `wkB/s` (throughput), `await` (average I/O latency in milliseconds), `%util` (percentage of time the device was busy), and `svctm` (service time — deprecated but still useful for comparison).

`iotop` shows per-process I/O usage in real time, functioning like `top` but sorted by I/O instead of CPU. It's invaluable for finding which process is hammering a disk without having to correlate process IDs with iostat data.

In AWS, these tools are essential for EBS volume tuning. Understanding whether you've hit the volume's IOPS limit, throughput limit, or latency ceiling determines whether you need to upgrade volume type (gp2 → gp3), increase provisioned IOPS, use io2 Block Express, or implement application-level caching.

---

## 💡 Real-World Use Cases

- Diagnosing why a PostgreSQL database is slow by determining if the bottleneck is I/O wait rather than CPU
- Identifying which container on an ECS host is consuming the majority of EBS I/O bandwidth
- Tuning EBS gp3 volumes after migrating from gp2 by profiling actual workload IOPS and throughput requirements
- Detecting I/O storms caused by log rotation, database vacuum, or backup jobs competing with production traffic
- Verifying that `noatime` and write scheduler changes have reduced I/O overhead after a kernel tuning exercise

---

## 🔧 Commands & Examples

### iostat Basics

```bash
# Install sysstat (provides iostat, mpstat, sar)
apt-get install sysstat      # Ubuntu/Debian
yum install sysstat          # Amazon Linux / RHEL

# Basic iostat output (snapshot since boot)
iostat

# Extended statistics (-x) with device info (-d), interval 1s, 10 samples
iostat -xd 1 10

# Human-readable units
iostat -xdh 1 5

# Monitor specific device
iostat -x /dev/nvme0n1 1 10

# Show all partitions
iostat -p ALL 1 5

# Key columns in -x (extended) output:
# r/s      — reads per second (IOPS read)
# w/s      — writes per second (IOPS write)
# rkB/s    — read bandwidth (KB/s)
# wkB/s    — write bandwidth (KB/s)
# rrqm/s   — read requests merged per second (kernel merging adjacent I/Os)
# wrqm/s   — write requests merged per second
# r_await  — average read latency (ms) — THIS IS CRITICAL
# w_await  — average write latency (ms)
# aqu-sz   — average queue depth (> 1 = device is saturated)
# %util    — % of time device was processing I/O (100% = saturated)
```

### Reading iostat Output for EBS

```bash
# Run iostat and interpret the output:
iostat -xd 1 5 /dev/nvme0n1

# Sample output:
# Device  r/s   w/s  rkB/s  wkB/s  r_await  w_await  aqu-sz  %util
# nvme0n1 150   80   6000   3200   0.5      1.2      0.3     15.0

# Interpretation:
# r/s=150: 150 read IOPS → check against EBS volume IOPS limit
# rkB/s=6000: 6 MB/s read throughput
# r_await=0.5ms: excellent read latency (NVMe SSD should be < 1ms)
# w_await=1.2ms: good write latency
# aqu-sz=0.3: low queue depth, device not saturated
# %util=15%: device not under heavy load

# What BAD looks like:
# r_await=50ms: terrible — indicates queue backup or cold HDD
# aqu-sz=8: deep queue, device overwhelmed
# %util=100%: device at capacity, all requests queued
# wrqm/s high relative to w/s: many small writes being merged (may help)

# EBS gp3 baseline:
# - 3000 IOPS (free) → can provision up to 16000 IOPS
# - 125 MB/s throughput (free) → can provision up to 1000 MB/s
# - Expected latency: sub-millisecond for NVMe-backed instances

# Check if you're hitting EBS IOPS limits
iostat -xd 1 10 | awk '/nvme0n1/ {print "IOPS:", $2+$3, "await:", $10, "util:", $NF"%"}'
```

### iotop — Per-Process I/O

```bash
# Install
apt-get install iotop
yum install iotop

# Basic iotop (requires root or CAP_NET_ADMIN)
sudo iotop

# Batch mode (non-interactive, good for logging)
sudo iotop -b -n 5    # 5 snapshots

# Show only processes with actual I/O
sudo iotop -o

# Accumulate I/O totals (like -a in top)
sudo iotop -a

# Batch mode filtered to processes doing I/O
sudo iotop -b -o -n 3 -d 2    # 3 snapshots, 2s interval, only active I/O

# Key columns:
# TID      — thread ID
# PRIO     — I/O priority (rt=realtime, be=best-effort, idle=idle)
# USER     — process owner
# DISK READ — current read bandwidth
# DISK WRITE — current write bandwidth
# SWAPIN   — % time swapping in
# IO       — % time waiting for I/O
# COMMAND  — process name

# Find which process is causing an I/O spike right now
sudo iotop -b -o -n 1 | head -20

# Log I/O usage to file for analysis
sudo iotop -b -o -n 60 -d 5 > /tmp/iotop-$(date +%Y%m%d%H%M).log 2>&1 &
```

### /proc/diskstats Deep Dive

```bash
# Raw disk statistics from the kernel
cat /proc/diskstats

# Format: major minor device_name reads_completed reads_merged sectors_read
#         ms_reading writes_completed writes_merged sectors_written ms_writing
#         ios_in_progress ms_doing_io ms_doing_io_weighted

# Parse specific fields for nvme0n1
awk '/nvme0n1 / {
  reads=$4; writes=$8; sectors_r=$6; sectors_w=$10;
  read_ms=$7; write_ms=$11;
  print "Reads:", reads, "Writes:", writes
  print "Read sectors:", sectors_r, "Write sectors:", sectors_w
  print "Read ms:", read_ms, "Write ms:", write_ms
}' /proc/diskstats

# Compute average latency manually (delta over interval)
#!/bin/bash
get_stats() {
  awk '/nvme0n1 /{print $4, $8, $7, $11}' /proc/diskstats
}
STAT1=($(get_stats)); sleep 5; STAT2=($(get_stats))

READS=$((STAT2[0] - STAT1[0]))
WRITES=$((STAT2[1] - STAT1[1]))
READ_MS=$((STAT2[2] - STAT1[2]))
WRITE_MS=$((STAT2[3] - STAT1[3]))

[ $READS -gt 0 ] && echo "Avg read latency: $(echo "scale=2; $READ_MS / $READS" | bc) ms"
[ $WRITES -gt 0 ] && echo "Avg write latency: $(echo "scale=2; $WRITE_MS / $WRITES" | bc) ms"
```

### I/O Schedulers

```bash
# View current I/O scheduler for a device
cat /sys/block/nvme0n1/queue/scheduler
# [none] mq-deadline kyber
# The [bracketed] one is active

# Available schedulers depend on kernel version:
# none       — no scheduler, used for NVMe SSDs (pass-through to device)
# mq-deadline — deadline-based fair queueing (good for mixed workloads)
# kyber      — latency-optimised for SSDs (good for latency-sensitive apps)
# bfq        — Budget Fair Queueing (good for HDDs and mixed I/O)

# Change scheduler at runtime
echo mq-deadline > /sys/block/nvme0n1/queue/scheduler
echo kyber > /sys/block/nvme0n1/queue/scheduler
echo none > /sys/block/nvme0n1/queue/scheduler

# Persist across reboots
cat > /etc/udev/rules.d/60-scheduler.rules << 'EOF'
# NVMe SSDs — no scheduler (device handles its own queue)
ACTION=="add|change", KERNEL=="nvme*", ATTR{queue/scheduler}="none"
# SATA SSDs — mq-deadline
ACTION=="add|change", KERNEL=="sd*", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"
# HDDs — bfq
ACTION=="add|change", KERNEL=="sd*", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="bfq"
EOF

udevadm control --reload-rules
udevadm trigger

# View queue depth
cat /sys/block/nvme0n1/queue/nr_requests       # current queue depth
cat /sys/block/nvme0n1/queue/max_hw_sectors_kb # max request size
```

### Read-Ahead Tuning

```bash
# View current read-ahead setting (in 512-byte sectors)
blockdev --getra /dev/nvme0n1
# Default: 256 = 128KB read-ahead

# Set read-ahead (larger for sequential workloads, smaller for random)
# 4096 = 2MB read-ahead (good for sequential media streaming, backups)
blockdev --setra 4096 /dev/nvme0n1

# 256 = 128KB (default, good for mixed OLTP workloads)
blockdev --setra 256 /dev/nvme0n1

# 0 = disable read-ahead (for pure random I/O like databases)
blockdev --setra 0 /dev/nvme0n1

# Check if read-ahead is helping or hurting
# High rrqm/s in iostat = reads are being merged = read-ahead is working
# If workload is purely random (database index lookups), read-ahead adds latency

# Persist via udev rules
echo 'ACTION=="add|change", KERNEL=="nvme0n1", RUN+="/sbin/blockdev --setra 4096 /dev/$kernel"' \
  >> /etc/udev/rules.d/60-readahead.rules
```

### Mount Options for I/O Performance

```bash
# noatime — don't update access time on reads (reduces write I/O by 30-50% on read-heavy workloads)
mount -o remount,noatime /data
# /etc/fstab:
# /dev/nvme0n1 /data ext4 defaults,noatime 0 2

# relatime — only update atime if it's older than mtime (compromise)
mount -o remount,relatime /

# data=writeback — don't journal data writes (only metadata — RISKY, may lose data on crash)
# data=ordered — default for ext4, safe and reasonably fast
# data=journal — journal everything, safest, slowest

# For databases using direct I/O (PostgreSQL, MySQL):
# The DB handles its own cache; OS page cache is wasteful and adds double-buffering
# Use O_DIRECT by configuring the database, not the filesystem

# XFS: nobarrier (only safe on battery-backed write cache or cloud block storage)
mount -o remount,nobarrier /data
# On EBS, barriers can be safely disabled since EBS is durable:
# /dev/nvme0n1 /data xfs defaults,noatime,nobarrier 0 2
```

### Profiling I/O with blktrace and fio

```bash
# blktrace: kernel-level I/O tracing
apt-get install blktrace

# Trace I/O on a device for 10 seconds
blktrace -d /dev/nvme0n1 -w 10 -o /tmp/trace

# Parse the trace
blkparse /tmp/trace.blktrace.0 | head -50
# Shows: timestamp, operation type, sector, size

# blkparse output operations:
# Q = queued (new request arrived)
# M = merged (merged with existing request)
# S = sleep (scheduler sleeping)
# I = insert (inserted into device queue)
# D = issued (sent to device driver)
# C = completed (I/O complete)
# D-C latency = actual device service time

# fio: synthetic I/O benchmarking
apt-get install fio

# Test random read IOPS (simulates database index reads)
fio --name=randread \
    --ioengine=libaio \
    --iodepth=16 \
    --rw=randread \
    --bs=4k \
    --direct=1 \
    --numjobs=4 \
    --size=4g \
    --runtime=60 \
    --filename=/dev/nvme0n1

# Test sequential throughput (simulates backup/ETL)
fio --name=seqwrite \
    --ioengine=libaio \
    --iodepth=64 \
    --rw=write \
    --bs=1M \
    --direct=1 \
    --numjobs=1 \
    --size=10g \
    --runtime=60 \
    --filename=/tmp/testfile

# Simulate mixed OLTP workload (70% read, 30% write, random 4K)
fio --name=oltp \
    --ioengine=libaio \
    --iodepth=32 \
    --rw=randrw \
    --rwmixread=70 \
    --bs=4k \
    --direct=1 \
    --numjobs=8 \
    --size=4g \
    --runtime=120 \
    --filename=/dev/nvme0n1
```

### AWS EBS-Specific Tuning

```bash
# Check volume type and limits
aws ec2 describe-volumes --volume-ids vol-xxx \
  --query 'Volumes[0].{Type:VolumeType,IOPS:Iops,Throughput:Throughput,Size:Size}'

# Check CloudWatch EBS metrics (more accurate than iostat for EBS limits)
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeReadOps \
  --dimensions Name=VolumeId,Value=vol-xxx \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Sum

# gp2 vs gp3 decision:
# gp2: IOPS = min(3 * GB, 16000), burst to 3000 for < 1TB volumes
# gp3: 3000 IOPS + 125 MB/s base FREE, provision up to 16000 IOPS / 1000 MB/s independently
# gp3 is almost always cheaper and more flexible than gp2

# Modify a running volume (no downtime on EC2)
aws ec2 modify-volume \
  --volume-id vol-xxx \
  --volume-type gp3 \
  --iops 6000 \
  --throughput 500

# Monitor the modification
aws ec2 describe-volumes-modifications --volume-ids vol-xxx

# EBS-optimised instances: ensure your instance type supports it
aws ec2 describe-instance-types \
  --instance-types r6i.2xlarge \
  --query 'InstanceTypes[0].EbsInfo'
# Check EbsOptimizedSupport and EbsOptimizedInfo.MaximumBandwidthInMbps
```

---

## ⚠️ Gotchas & Pro Tips

- **`%util=100%` doesn't always mean the device is the bottleneck:** For NVMe drives with native command queuing, `%util` measures time with *any* outstanding I/O — not actual saturation. A modern NVMe SSD can serve many parallel requests while `%util` shows 100%. Always check `aqu-sz` (queue depth) and `r_await`/`w_await` (latency) together with `%util`.

- **`await` is the most important single metric:** Average I/O latency (`await` or `r_await`/`w_await` in newer iostat) directly translates to application latency. For NVMe SSDs, this should be under 1ms. For EBS gp3, expect 0.5-1ms under normal load. Anything above 5-10ms indicates a problem.

- **`noatime` is safe and almost always worth it:** The default `atime` mount option causes a write for every file read (updating the access timestamp). On read-heavy workloads like web servers and databases, this doubles the write I/O. Use `noatime` or `relatime` — it's been safe on Linux for years.

- **EBS burst credits on gp2 silently exhaust:** gp2 volumes under 1TB use burst credits. A volume at 100GB gets baseline 300 IOPS but can burst to 3000. When credits exhaust (after sustained I/O), performance drops to 300 IOPS with no warning. Monitor `BurstBalance` in CloudWatch. Migrate to gp3 to eliminate this unpredictability.

- **Direct I/O for databases:** PostgreSQL, MySQL, and most production databases use `O_DIRECT` to bypass the kernel page cache and manage their own buffer pools. This avoids double-buffering (data in both database buffer pool and OS page cache). Don't size your EC2 instance RAM assuming the OS page cache helps database I/O — it doesn't for well-configured databases.

- **iotop in containers:** `iotop` shows host-level per-process I/O, not per-container. On a K8s node, it correctly shows I/O by PID, but you need to correlate PIDs to containers using `cat /proc/<pid>/cgroup`. The `docker stats` container I/O metric is unreliable; use `blkio.throttle.io_service_bytes` in the container's cgroup instead.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
