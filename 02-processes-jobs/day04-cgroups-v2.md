# 🐧 cgroups v2 — Linux Mastery

> **cgroups v2 is the unified resource management subsystem that underpins every container runtime, Kubernetes QoS class, and systemd slice — understanding it directly makes you better at diagnosing OOM kills, CPU throttling, and noisy-neighbour problems.**

## 📖 Concept

Control Groups (cgroups) are a Linux kernel feature that limits, accounts for, and isolates the resource usage (CPU, memory, I/O, network) of process groups. Version 2, introduced in kernel 4.5 and made default in most distributions by 2020 (RHEL 9, Ubuntu 22.04+), unifies all controllers under a single hierarchy rooted at `/sys/fs/cgroup`. This replaces the fragmented v1 system where each controller (memory, cpu, blkio, etc.) had its own separate tree.

Every process on a modern Linux system belongs to exactly one cgroup in the unified hierarchy. When systemd starts, it creates slices (`.slice` units) and scopes/services underneath, putting all system services under `system.slice` and user sessions under `user.slice`. Kubernetes uses cgroups to enforce CPU requests/limits and memory limits — the `resources.limits.memory` field in a Pod spec directly maps to the `memory.max` file in the container's cgroup directory.

In AWS, EC2 instances running EKS or ECS both rely on cgroups for workload isolation. When a container is OOM-killed, the kernel is enforcing a memory cgroup limit. When a Pod is CPU-throttled, the kernel's `cpu.max` quota mechanism is responsible. Understanding cgroups at this level lets you tune, debug, and optimise workloads far beyond what kubectl or the AWS console exposes.

---

## 💡 Real-World Use Cases

- Diagnosing why a K8s pod is CPU-throttled even when host CPU is idle (cpu quota exhausted within the period)
- Setting per-service memory limits in systemd to prevent a runaway log collector from OOM-killing the entire instance
- Limiting an ad-hoc data-processing job to use only 2 CPUs and 4 GB RAM without Docker, using raw cgroup controls
- Understanding why `docker stats` reports high memory for a container even after freeing data (page cache counted inside cgroup)

---

## 🔧 Commands & Examples

### Verifying cgroups v2

```bash
# Check if unified hierarchy is active (v2)
mount | grep cgroup
# cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate,memory_recursiveprot)
# If you see cgroup type cgroup (v1), the system uses legacy or hybrid mode

# Check kernel config
grep CONFIG_CGROUP /boot/config-$(uname -r) | head -20

# See currently enabled controllers
cat /sys/fs/cgroup/cgroup.controllers
# cpuset cpu io memory hugetlb pids rdma misc

# See which controllers are active on the root
cat /sys/fs/cgroup/cgroup.subtree_control
```

### Exploring the Hierarchy

```bash
# List top-level cgroup directories (systemd slices)
ls /sys/fs/cgroup/
# init.scope  system.slice  user.slice  ...

# See all cgroups as a tree
systemd-cgls

# See a specific service's cgroup
systemd-cgls /system.slice/nginx.service

# Find which cgroup a process belongs to
cat /proc/<PID>/cgroup
# 0::/system.slice/nginx.service

# Example: find cgroup for all nginx workers
for pid in $(pgrep nginx); do
  echo "PID $pid: $(cat /proc/$pid/cgroup)"
done
```

### Memory Controller

```bash
# View memory usage for a service
cat /sys/fs/cgroup/system.slice/nginx.service/memory.current
# 45678592  (bytes)

# View memory limit
cat /sys/fs/cgroup/system.slice/nginx.service/memory.max
# max  (no limit)

# View memory statistics breakdown
cat /sys/fs/cgroup/system.slice/nginx.service/memory.stat
# anon 40960000          — anonymous pages (heap/stack)
# file 4718592           — page cache (file-backed)
# slab 0
# ...

# View OOM events
cat /sys/fs/cgroup/system.slice/nginx.service/memory.events
# low 0
# high 0
# max 0
# oom 2          ← this service has been OOM-killed twice
# oom_kill 2

# Set a memory limit via systemd (proper way)
systemctl set-property nginx.service MemoryMax=512M
# This writes to /etc/systemd/system.control/nginx.service.d/override.conf

# Or raw kernel interface (transient, lost on reboot)
echo $((512 * 1024 * 1024)) > /sys/fs/cgroup/system.slice/nginx.service/memory.max

# Set memory high watermark (soft limit — process gets throttled before OOM)
echo $((400 * 1024 * 1024)) > /sys/fs/cgroup/system.slice/nginx.service/memory.high

# Swap limit
cat /sys/fs/cgroup/system.slice/nginx.service/memory.swap.max
```

### CPU Controller

```bash
# View CPU weight (relative scheduling priority, replaces cpu.shares)
cat /sys/fs/cgroup/system.slice/nginx.service/cpu.weight
# 100  (default)

# View CPU quota and period (hard limit)
cat /sys/fs/cgroup/system.slice/nginx.service/cpu.max
# max 100000
# Format: <quota_microseconds> <period_microseconds>
# "max" = unlimited. "50000 100000" = 50% of one CPU per 100ms period

# Set CPU limit to 1.5 CPUs (150% of one core)
echo "150000 100000" > /sys/fs/cgroup/system.slice/nginx.service/cpu.max

# Via systemd (preferred)
systemctl set-property nginx.service CPUQuota=150%

# View CPU usage statistics
cat /sys/fs/cgroup/system.slice/nginx.service/cpu.stat
# usage_usec 12345678        — total CPU time used
# user_usec  10000000
# system_usec 2345678
# nr_periods 12345           — number of enforcement periods
# nr_throttled 100           — periods where the cgroup hit the quota
# throttled_usec 500000      — total time throttled

# High nr_throttled/nr_periods ratio = your CPU limit is too tight
```

### I/O Controller

```bash
# Find block device major:minor
ls -la /dev/nvme0n1
# brw-rw---- 1 root disk 259, 0 ...  → major=259, minor=0

# View I/O stats for a cgroup
cat /sys/fs/cgroup/system.slice/nginx.service/io.stat
# 259:0 rbytes=1048576 wbytes=2097152 rios=100 wios=200 dbytes=0 dios=0

# Set I/O limit (bytes per second)
# Limit write bandwidth to 50 MB/s on /dev/nvme0n1
echo "259:0 wbps=52428800" > /sys/fs/cgroup/system.slice/nginx.service/io.max

# Limit read IOPS
echo "259:0 riops=1000" > /sys/fs/cgroup/system.slice/nginx.service/io.max

# Via systemd
systemctl set-property nginx.service IOWriteBandwidthMax="/dev/nvme0n1 50M"
systemctl set-property nginx.service IOReadIOPSMax="/dev/nvme0n1 1000"
```

### PID Controller

```bash
# View current PID count in a cgroup
cat /sys/fs/cgroup/system.slice/nginx.service/pids.current

# View PID limit
cat /sys/fs/cgroup/system.slice/nginx.service/pids.max
# max

# Set a PID limit (prevents fork bombs)
echo 200 > /sys/fs/cgroup/system.slice/nginx.service/pids.max

# Via systemd
systemctl set-property nginx.service TasksMax=200
```

### Creating Custom cgroups Manually

```bash
# Create a new cgroup for ad-hoc resource limiting
mkdir /sys/fs/cgroup/my-batch-job

# Enable controllers for this cgroup (must be in parent's subtree_control)
echo "+cpu +memory +io" > /sys/fs/cgroup/cgroup.subtree_control
echo "+cpu +memory +io" > /sys/fs/cgroup/my-batch-job/cgroup.subtree_control

# Set limits
echo $((4 * 1024 * 1024 * 1024)) > /sys/fs/cgroup/my-batch-job/memory.max  # 4GB
echo "200000 100000" > /sys/fs/cgroup/my-batch-job/cpu.max                   # 2 CPUs

# Move a process into the cgroup
echo $$ > /sys/fs/cgroup/my-batch-job/cgroup.procs     # move current shell
# Now run your job — it and all children will be in this cgroup

# Or use systemd-run for clean transient cgroup management
systemd-run --scope \
  --property MemoryMax=4G \
  --property CPUQuota=200% \
  --slice=batch.slice \
  /usr/bin/python3 /opt/jobs/heavy-etl.py
```

### Kubernetes cgroup Integration

```bash
# On a K8s node, find the cgroup for a specific pod
# First get the container ID
kubectl get pod mypod -o jsonpath='{.status.containerStatuses[0].containerID}'
# containerd://abc123...

# Find the cgroup on the node
find /sys/fs/cgroup -name "*.scope" | xargs grep -l abc123 2>/dev/null
# Or with newer nodes using cgroup v2:
ls /sys/fs/cgroup/kubepods.slice/

# Check if a pod is being CPU-throttled
CGROUP=$(find /sys/fs/cgroup/kubepods.slice -name "cpu.stat" | grep <pod-uid>)
cat $CGROUP | grep throttled

# Check OOM kills for a pod
find /sys/fs/cgroup/kubepods.slice -name "memory.events" -exec grep -l "oom_kill [^0]" {} \;

# Understand QoS classes → cgroup placement:
# Guaranteed → kubepods.slice/kubepods-pod<uid>.slice/
# Burstable   → kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod<uid>.slice/
# BestEffort  → kubepods.slice/kubepods-besteffort.slice/
```

### Monitoring with systemd

```bash
# Real-time resource usage per service (like top but per cgroup)
systemd-cgtop

# Detailed resource accounting for a service
systemctl status nginx.service  # shows memory and tasks
systemd-cgtop -n 1 -b          # single snapshot, machine-readable

# Historical resource data via journald
journalctl -u nginx.service --since "1 hour ago" | grep -i oom

# Set accounting explicitly for a service (sometimes disabled by default)
systemctl set-property nginx.service CPUAccounting=yes MemoryAccounting=yes IOAccounting=yes
```

---

## ⚠️ Gotchas & Pro Tips

- **cgroups v1 vs v2 confusion in containers:** Docker and containerd on modern systems default to cgroup v2, but older K8s node configurations or custom setups may still use v1. The file paths and filenames differ significantly (`memory.limit_in_bytes` in v1 vs `memory.max` in v2). Always check which version is active with `mount | grep cgroup`.

- **CPU throttling is not CPU starvation:** A container can be heavily throttled (hitting its quota) even when the node has idle CPU. The quota mechanism is time-period based — if your container uses its quota in the first 10ms of a 100ms period, it waits the remaining 90ms even if CPUs are free. Increasing the quota (`cpu.max`) or period helps.

- **Memory includes page cache:** The kernel counts file-backed pages (the page cache) against a cgroup's memory usage by default. A container that reads lots of files will appear to use more memory than just its heap. This is usually fine (the cache is evictable), but it can trigger `memory.high` soft limits. Use `memory.stat` to distinguish `anon` (non-evictable heap/stack) from `file` (evictable cache).

- **`oom_kill` vs process death:** When a cgroup is OOM-killed, the kernel selects a process inside the cgroup to kill based on `oom_score_adj`. Set `oom_score_adj` to `-1000` in `/proc/<PID>/oom_score_adj` to make a critical process effectively immune (systemd does this for itself).

- **Systemd property changes are persistent:** `systemctl set-property` writes override files to `/etc/systemd/system.control/`. These survive reboots. Use `systemctl revert nginx.service` to remove all property overrides.

- **`systemd-run` is your best friend for ad-hoc limits:** Instead of manually writing cgroup files, use `systemd-run --scope` or `systemd-run --service-type=exec` to launch jobs with resource limits. It handles cgroup lifecycle cleanly.

```bash
# Run a memory-intensive script with guardrails
systemd-run --scope \
  --property MemoryMax=2G \
  --property CPUQuota=100% \
  --property MemorySwapMax=0 \
  --unit=my-etl-job \
  python3 /opt/etl/process.py

# Monitor it in real time
systemd-cgtop /system.slice/my-etl-job.scope
```

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
