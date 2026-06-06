# 🐧 CPU Performance Tuning & Huge Pages — Linux Mastery

> **Extract every CPU cycle from your Linux system — from NUMA topology to huge pages to CPU pinning for latency-critical workloads.**

## 📖 Concept

Linux's default CPU and memory configuration is tuned for general workloads. For high-performance databases, message brokers (Kafka), in-memory caches (Redis), real-time trading systems, and K8s control plane components, the defaults leave significant performance on the table. This guide covers the tuning knobs that matter most in production.

**CPU frequency scaling:** Modern CPUs dynamically adjust their frequency based on load. The governor controls this behavior. `powersave` reduces frequency to minimize power; `performance` keeps the CPU at max frequency; `schedutil` is the modern adaptive governor. For latency-sensitive workloads (database query execution, network packet processing), `performance` governor eliminates frequency-scaling induced jitter.

**NUMA (Non-Uniform Memory Access):** Servers with multiple CPU sockets have non-uniform memory latency. Each socket has local RAM (fast, ~100ns) and remote RAM (2-3× slower). NUMA-unaware applications on multi-socket servers silently use remote memory. `numactl` pins processes to a NUMA node to ensure local memory access.

**Huge Pages:** The default Linux page size is 4KB. TLB (Translation Lookaside Buffer) caches virtual-to-physical address mappings. With 4KB pages, a 1GB dataset needs 262,144 TLB entries — far more than the TLB can hold, causing frequent TLB misses. Huge pages (2MB or 1GB) drastically reduce TLB pressure — 512 huge pages cover what 262,144 small pages did. Databases (PostgreSQL, Oracle), Java JVMs, Redis, and Elasticsearch all benefit significantly.

---

## 💡 Real-World Use Cases

- Configure huge pages for a PostgreSQL server to reduce TLB miss rate by 60-80%
- Pin Kafka broker to a specific NUMA node to eliminate cross-socket memory latency
- Set CPU governor to `performance` on EC2 bare metal instances for consistent query latency
- Disable NUMA balancing on a Redis server to prevent automatic memory page migration that causes latency spikes
- Use `taskset` and `cgroups cpuset` to isolate a critical process from noisy neighbors on a K8s node

---

## 🔧 Commands & Examples

### CPU Topology and Frequency

```bash
# Understand your CPU topology
lscpu
# Architecture:            x86_64
# CPU(s):                  96
# Thread(s) per core:      2    (= hyperthreading)
# Core(s) per socket:      24
# Socket(s):               2
# NUMA node(s):            2
# NUMA node0 CPU(s):       0-23,48-71
# NUMA node1 CPU(s):       24-47,72-95

# More detailed topology
lscpu --extended   # shows each CPU with socket, core, NUMA affinity

# CPU cache information
lscpu | grep -i cache
# L1d cache:  32K per core
# L2 cache:   256K per core
# L3 cache:   36MB shared per socket

# Check CPU frequency and governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq

# List all available governors
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
# conservative ondemand userspace powersave performance schedutil

# Set governor for ALL CPUs
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance > "$cpu"
done

# Verify
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor | sort -u
# performance

# Set governor persistently with cpupower
cpupower frequency-set -g performance
cpupower frequency-info

# On AWS EC2 — check if turbo boost is enabled
cat /sys/devices/system/cpu/intel_pstate/no_turbo
# 0 = turbo enabled (good for batch), 1 = turbo disabled

# Disable turbo boost for consistent (not peak) latency
echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo
```

### NUMA Awareness and Tuning

```bash
# Check NUMA topology
numactl --hardware
# available: 2 nodes (0-1)
# node 0 cpus: 0 1 2 3 ... 23 48 49 ... 71
# node 0 size: 96473 MB
# node 0 free: 45123 MB
# node 1 cpus: 24 25 26 27 ... 47 72 73 ... 95
# node 1 size: 96495 MB
# node 1 free: 50234 MB
# node distances:
# node   0   1
#   0:  10  21
#   1:  21  10
# (10 = local access latency units, 21 = remote — 2.1x slower)

# Check NUMA memory usage per node
numastat
numastat -m   # per-process NUMA memory stats

# Check NUMA hit/miss rate (numa_miss = cross-socket accesses — want this LOW)
numastat -z | grep -E "numa_hit|numa_miss|local_node|other_node"

# Run a process bound to a specific NUMA node (both CPU and memory)
numactl --cpunodebind=0 --membind=0 /usr/bin/postgres

# Run Redis on NUMA node 0 exclusively
numactl --cpunodebind=0 --membind=0 redis-server /etc/redis/redis.conf

# Run a Java process (Kafka) on NUMA node 1
numactl --cpunodebind=1 --membind=1 java -jar kafka.jar

# Check NUMA policy of a running process
numactl --show   # current process
cat /proc/$(pgrep postgres | head -1)/numa_maps | head -10

# Disable automatic NUMA balancing (can cause latency spikes from page migration)
# Default: enabled (kernel automatically migrates pages to local node)
# For databases that manage their own memory (PostgreSQL, Oracle): DISABLE
echo 0 > /proc/sys/kernel/numa_balancing

# Make persistent in /etc/sysctl.conf
echo "kernel.numa_balancing = 0" >> /etc/sysctl.conf

# For K8s nodes — check if NUMA-aware scheduling is configured
kubectl describe node <node> | grep -i numa
# kubelet can enforce NUMA alignment with --topology-manager-policy=single-numa-node
```

### Huge Pages Configuration

```bash
# Check current huge page configuration
cat /proc/meminfo | grep -i huge
# AnonHugePages:    2097152 kB    (transparent huge pages in use)
# HugePages_Total:       0        (static huge pages allocated)
# HugePages_Free:        0
# HugePages_Rsvd:        0
# Hugepagesize:       2048 kB     (2MB pages)

# ─── Static Huge Pages (2MB) ─────────────────────────────────────────────────
# Reserve 1000 × 2MB = 2GB of huge pages for databases

# Allocate immediately (from free memory — allocate early at boot)
echo 1000 > /proc/sys/vm/nr_hugepages
# or:
sysctl -w vm.nr_hugepages=1000

# Verify allocation
grep HugePages /proc/meminfo
# HugePages_Total:    1000
# HugePages_Free:     1000   ← must be same as total until process uses them

# Make persistent
echo "vm.nr_hugepages = 1000" >> /etc/sysctl.conf

# Allocate huge pages per NUMA node (better for NUMA systems)
echo 500 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
echo 500 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages

# ─── Configure PostgreSQL to Use Huge Pages ───────────────────────────────────
# /etc/postgresql/14/main/postgresql.conf
echo "huge_pages = on" >> /etc/postgresql/14/main/postgresql.conf

# Calculate how many huge pages PostgreSQL needs
# shared_buffers / hugepage_size = number of pages needed
# e.g., 16GB shared_buffers / 2MB hugepages = 8192 pages
sysctl -w vm.nr_hugepages=8192

# Verify PostgreSQL is using huge pages
grep ^VmFlags /proc/$(pgrep -f "postgres: checkpointer" | head -1)/smaps | \
    grep -c "ht"   # ht = huge page in VmFlags

# Or:
grep AnonHugePages /proc/$(pgrep -f "postgres: checkpointer" | head -1)/smaps | \
    awk '{sum+=$2} END {print sum/1024 " MB using AnonHugePages"}'

# ─── 1GB Huge Pages ──────────────────────────────────────────────────────────
# Even larger pages for very large in-memory workloads

# Must be configured at boot time via kernel cmdline
# Add to /etc/default/grub:
# GRUB_CMDLINE_LINUX="hugepagesz=1G hugepages=4 default_hugepagesz=1G"
grep 1G /proc/meminfo
# Hugepages_Total (for 1G pages)

# ─── Transparent Huge Pages (THP) ────────────────────────────────────────────
# THP: kernel automatically uses 2MB pages when possible
# Good for some workloads, BAD for databases (causes latency spikes during compaction)

cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never
# always = THP enabled for all memory
# madvise = THP only for memory explicitly requesting it
# never = THP disabled

# For databases (PostgreSQL, MySQL, Oracle, MongoDB) — DISABLE THP
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Check if THP defrag is running (causes stop-the-world pauses)
cat /sys/kernel/mm/transparent_hugepage/defrag
# [always] defer defer+madvise madvise never

# For databases: never. For JVM: madvise (JVM manages its own huge pages)
echo defer+madvise > /sys/kernel/mm/transparent_hugepage/defrag

# Make persistent via rc.local or systemd service
cat > /etc/systemd/system/disable-thp.service << 'EOF'
[Unit]
Description=Disable Transparent Huge Pages
After=sysinit.target local-fs.target
Before=mongod.service postgresql.service redis.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled && echo never > /sys/kernel/mm/transparent_hugepage/defrag'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
systemctl enable --now disable-thp.service
```

### CPU Pinning and Isolation

```bash
# taskset — pin a process to specific CPUs

# Run a process on CPUs 0 and 1 only
taskset -c 0,1 /usr/bin/myapp

# Pin by CPU mask (bitmask — CPU 0 = bit 0, CPU 1 = bit 1, etc.)
taskset 0x3 /usr/bin/myapp   # CPUs 0 and 1 (binary: 11)

# Change CPU affinity of a running process
taskset -cp 0,1 12345   # PID 12345 — pin to CPUs 0 and 1

# Check current affinity of a process
taskset -p $(pgrep postgres | head -1)

# Pin IRQ handlers to specific CPUs (isolate app CPUs from interrupt handling)
# Find IRQ for network interface
grep eth0 /proc/interrupts

# Set IRQ affinity to CPU 0 only (so CPUs 1-7 are free for application)
echo 1 > /proc/irq/42/smp_affinity   # hex bitmask: CPU 0 only

# ─── isolcpus — Kernel-Level CPU Isolation ────────────────────────────────────
# Removes CPUs from the scheduler — NOTHING runs on them unless explicitly assigned
# Add to /etc/default/grub:
# GRUB_CMDLINE_LINUX="isolcpus=2,3,4,5 nohz_full=2,3,4,5 rcu_nocbs=2,3,4,5"
# update-grub && reboot

# After boot, only explicitly taskseted processes run on CPUs 2-5
taskset -c 2-5 /usr/bin/latency_critical_app

# Check isolated CPUs
cat /sys/devices/system/cpu/isolated

# ─── cgroups cpuset — Container-Level CPU Pinning ─────────────────────────────
# Pin a K8s pod to specific CPUs via resource limits + kubelet topology manager
# In pod spec:
# resources:
#   requests:
#     cpu: "2"       # exact CPU count (integer) enables exclusive allocation
#     memory: "4Gi"
#   limits:
#     cpu: "2"       # requests == limits = Guaranteed QoS = eligible for CPU pinning
#     memory: "4Gi"
# topologyManager: single-numa-node (kubelet config)
```

### Production Tuning Checklist Script

```bash
#!/bin/bash
# CPU and memory performance audit script

set -uo pipefail

echo "=== CPU Performance Audit ==="
echo ""

# CPU governor
echo "1. CPU Governor:"
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor 2>/dev/null || echo "N/A (no cpufreq)"
echo ""

# Turbo boost
echo "2. Intel Turbo Boost:"
cat /sys/devices/system/cpu/intel_pstate/no_turbo 2>/dev/null && \
    echo "(0=enabled, 1=disabled)" || echo "N/A"
echo ""

# NUMA topology
echo "3. NUMA Nodes:"
numactl --hardware 2>/dev/null | grep -E "^node [0-9]+ (cpus|size)"
echo ""

# NUMA balancing
echo "4. NUMA Balancing:"
cat /proc/sys/kernel/numa_balancing
echo "(0=disabled recommended for databases)"
echo ""

# Huge pages
echo "5. Huge Pages (2MB):"
grep HugePages /proc/meminfo | grep -v Anon
echo ""

# THP
echo "6. Transparent Huge Pages:"
cat /sys/kernel/mm/transparent_hugepage/enabled
echo "(recommend: never or madvise for databases)"
echo ""

# THP defrag
echo "7. THP Defrag:"
cat /sys/kernel/mm/transparent_hugepage/defrag
echo "(recommend: never or defer+madvise for databases)"
echo ""

# Recommendations
echo "=== Recommendations for Database/High-Performance Workloads ==="
GOVERNOR=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor 2>/dev/null)
THP=$(cat /sys/kernel/mm/transparent_hugepage/enabled | grep -o '\[.*\]' | tr -d '[]')
NUMA_BAL=$(cat /proc/sys/kernel/numa_balancing)
HP_TOTAL=$(grep HugePages_Total /proc/meminfo | awk '{print $2}')

[ "$GOVERNOR" != "performance" ] && echo "⚠ Set CPU governor to 'performance'"
[ "$THP" == "always" ] && echo "⚠ Disable THP (echo never > /sys/kernel/mm/transparent_hugepage/enabled)"
[ "$NUMA_BAL" == "1" ] && echo "⚠ Disable NUMA balancing (sysctl -w kernel.numa_balancing=0)"
[ "$HP_TOTAL" == "0" ] && echo "⚠ No huge pages allocated — configure if running databases"

echo ""
echo "=== Done ==="
```

---

## ⚠️ Gotchas & Pro Tips

- **`performance` governor + EBS on EC2:** On EC2, the CPU frequency governor doesn't directly map to physical frequencies (that's managed by the hypervisor). However, setting `performance` on instance types that support it (metal, compute-optimized) still reduces frequency-transition overhead. On T3/T4 burstable instances, it has no effect.

- **Huge page pre-allocation must happen early:** The kernel allocates huge pages by finding contiguous 2MB physical memory regions. On a system that's been running for hours with fragmented memory, `echo 1000 > /proc/sys/vm/nr_hugepages` may only partially succeed. Always allocate at boot time. Check `HugePages_Total` vs what you requested.

- **THP causes Redis latency spikes:** Redis 6.0+ documentation explicitly recommends disabling THP. The issue: THP defrag causes stop-the-world pauses of 200ms-2s while the kernel tries to compact memory into 2MB regions. This shows up as P99 latency spikes in monitoring.

- **NUMA and JVM:** Java's garbage collector is NUMA-unaware by default. Use `-XX:+UseNUMA` JVM flag to enable NUMA-aware heap allocation. This alone can improve throughput 20-30% on multi-socket servers for Kafka, Elasticsearch, and Cassandra.

- **`isolcpus` is permanent until reboot:** Once you isolate CPUs in the kernel cmdline, they're excluded from the scheduler completely. This is powerful but means system daemons also don't run there — including monitoring agents. Make sure your monitoring can tolerate this.

- **CPU pinning and K8s:** Kubernetes CPU Manager (Guaranteed QoS class + integer CPU requests) provides CPU pinning, but requires `kubelet --cpu-manager-policy=static`. Without this, your pod can migrate between CPUs, causing cache misses and NUMA effects even if your pods are perfectly written.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
