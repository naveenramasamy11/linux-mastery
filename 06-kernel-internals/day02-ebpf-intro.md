# 🐧 eBPF Introduction — Linux Mastery

> **Harness eBPF for real-time system observability, tracing, and security—no kernel recompile required.**

## 📖 Concept

eBPF (extended Berkeley Packet Filter) is one of the most important developments in Linux observability of the past decade. It allows arbitrary code to run safely in the kernel without rebooting or modifying kernel code. This is revolutionary: previous observability required kernel modules (brittle, dangerous, require recompile) or userspace tools (limited scope, high overhead). eBPF is safe by design—the kernel's verifier ensures programs can't crash the kernel or access unauthorized memory.

For DevOps and SRE, eBPF means real-time visibility into system behavior with minimal overhead. Instead of strace (slow, context-switching overhead) or waiting for app instrumentation, you can trace syscalls, file operations, network events, and more live. Tools like `bpftrace` and `execsnoop` let you ask ad-hoc questions: "What files is this process opening?" "How long do writes take?" "Which process is generating the most network traffic?" These are the questions that solve production incidents.

eBPF powers modern observability platforms: Cilium (service mesh for Kubernetes), Pixie (continuous profiling), Tetragon (runtime security). Understanding the basics unlocks powerful debugging capabilities and makes you dangerous in production.

---

## 💡 Real-World Use Cases

- **Syscall tracing without overhead**: Trace open, read, write, connect syscalls live to understand what a misbehaving application is doing
- **File access auditing**: See which processes access sensitive files in real-time (part of compliance and security investigations)
- **Network traffic analysis**: Monitor TCP connections, packets per second, latency without packet capture overhead
- **CPU profiling**: Sample stack traces to build flamegraphs and identify performance bottlenecks
- **Memory leak detection**: Track allocation and deallocation patterns to find leaks
- **Container security**: Detect privilege escalation attempts, unauthorized privilege changes, suspicious child process execution
- **Kubernetes cluster observability**: Run eBPF in Cilium for pod-to-pod communication visibility and network policy enforcement

---

## 🔧 Commands & Examples

### bpftrace: Ad-hoc System Tracing

```bash
# Install bpftrace (different methods per distro)
apt-get install bpftrace          # Debian/Ubuntu
yum install bpftrace              # RHEL/CentOS
brew install bpftrace             # macOS (unofficial)

# Simple one-liners:

# Trace all open() syscalls
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_open { printf("%s(%s)\n", comm, str(args->filename)); }'

# Count syscalls by process
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[comm] = count(); }' -c sleep 1

# Measure time in kernel
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_read { @start[tid] = nsecs; } tracepoint:syscalls:sys_exit_read { printf("%s: %d ns\n", comm, nsecs - @start[tid]); }'

# Watch file opens with filename
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s(%s)\n", comm, str(args->filename)); }' | head -20

# Trace malloc calls
sudo bpftrace -e 'usdt:/usr/lib/libc.so:malloc { printf("malloc(%d)\n", arg0); }'
```

### execsnoop: Process Execution Tracing

```bash
# execsnoop comes with BCC (eBPF Compiler Collection)
apt-get install bcc-tools         # Debian/Ubuntu
yum install bcc-tools             # RHEL/CentOS

# Show all process execution
sudo /usr/share/bcc/tools/execsnoop

# Output:
# PCOMM            PID     PPID    RETVAL ARGS
# bash             1234    1233    0      /bin/bash
# python           2345    1234    0      python /app/worker.py
# find             3456    2345    0      find /data -type f

# Detailed version with return status
sudo execsnoop-bpfcc

# Filter to specific process
sudo execsnoop | grep python

# In Kubernetes: exec into pod, run execsnoop
kubectl exec -it POD_NAME -- bash
apt-get update && apt-get install -y bcc-tools
sudo execsnoop
```

### opensnoop: File Access Tracing

```bash
# Show all file opens
sudo /usr/share/bcc/tools/opensnoop

# Output:
# PID    COMM             FD ERR PATH
# 1234   python           3  0   /data/config.yaml
# 1234   python           4  0   /proc/stat
# 5678   nginx            -1 2   /tmp/missing.txt

# FD: file descriptor (-1 on error)
# ERR: errno on failure (2 = ENOENT)

# Verbose: show syscall time
sudo opensnoop -v

# Filter by process
sudo opensnoop -p 1234

# Follow specific file
sudo opensnoop | grep "specific/file"

# Find what's trying to access a missing file
sudo opensnoop | grep "ENOENT"
```

### tcpconnect: Network Connection Tracing

```bash
# Show TCP connections
sudo /usr/share/bcc/tools/tcpconnect

# Output:
# PID    COMM             SADDR      SPORT DADDR      DPORT
# 1234   curl             127.0.0.1  54321 8.8.8.8    53
# 5678   python           10.0.1.5   44322 10.0.1.10  3306

# Monitor connections to specific remote host
sudo tcpconnect | grep '10.0.1.10'

# Verbose: show TCP states
sudo tcpconnect -v

# Count connections by destination
awk '{print $6, $7}' | sort | uniq -c | sort -rn
```

### biolatency: Disk I/O Latency Profiling

```bash
# Measure block I/O latency distribution
sudo /usr/share/bcc/tools/biolatency

# Shows histogram of I/O latencies (useful for slow disk issues)
# Can identify outliers and patterns (e.g., every few seconds a spike)

# Per-disk breakdown
sudo biolatency -d

# Microsecond precision
sudo biolatency -m  # milliseconds (default)

# Sample for 10 seconds
sudo biolatency 10

# During load test, capture baseline
load_test &
sleep 5
sudo biolatency 60 > baseline.txt
```

### Writing Custom eBPF Programs (bpftrace)

```bash
# Script file: trace_open.bt
tracepoint:syscalls:sys_enter_openat
{
    printf("PID %d (%s) opened %s\n", 
        pid, comm, str(args->filename));
}

# Run it
sudo bpftrace trace_open.bt

# More complex: measure latency and aggregate
# Script: measure_syscall_latency.bt
tracepoint:syscalls:sys_enter_*
{
    @start[tid] = nsecs;
}

tracepoint:syscalls:sys_exit_*
{
    if (@start[tid]) {
        $duration_ns = nsecs - @start[tid];
        @latency[execname] = hist($duration_ns / 1000);  # Convert to microseconds
        delete(@start[tid]);
    }
}

# Run
sudo bpftrace measure_syscall_latency.bt

# Press Ctrl+C to see histogram output
```

### BCC Tools for Common Tasks

```bash
# Memory allocation tracking
sudo memleak -p 1234 -a          # Track malloc/free for PID 1234
sudo memleak -c python            # Top memory-allocating commands

# Flame graph generation
sudo profile -F 99 30 > cpu.txt   # Sample CPU at 99Hz for 30 seconds
stackcollapse-perf cpu.txt > cpu.collapsed
flamegraph.pl cpu.collapsed > cpu.svg

# TCP retransmit analysis
sudo tcpretrans                   # Show TCP retransmissions
sudo tcpretrans -v                # Verbose with details

# Disk latency heatmap
sudo biolatency --heatmap 60      # Generate heatmap over 60 seconds

# What's blocking on locks?
sudo offcputime                   # Time spent sleeping (waiting for locks, I/O)
sudo offcputime -f 100            # Show stack traces with -f
```

### eBPF in Kubernetes: Cilium for Network Observability

```yaml
# Install Cilium (provides eBPF-based networking)
helm repo add cilium https://helm.cilium.io
helm install cilium cilium/cilium --namespace kube-system

# View Cilium metrics (uses eBPF internally)
kubectl get pods -n kube-system | grep cilium

# Trace pod-to-pod communication
kubectl logs -n kube-system -l app.kubernetes.io/name=cilium-agent -f
```

### eBPF Performance Profiling

```bash
# Record CPU samples with flamegraph
sudo perf record -F 99 -g -p PID sleep 30
perf script > out.perf
stackcollapse-perf out.perf > out.collapsed
flamegraph.pl out.collapsed > cpu.svg

# Alternative: async-profiler (eBPF-based for JVM)
git clone https://github.com/jvm-profiling-tools/async-profiler
cd async-profiler && make

# Start profiling
./profiler.sh -d 30 -f flamegraph.html PID
# Result: flamegraph.html ready to view

# eBPF profiler alternatives:
# - Perf (Linux kernel)
# - BPF profiler (eBPF-based)
# - Pixie (continuous profiling with eBPF)
```

### Real-World Troubleshooting Example

```bash
# Problem: Application is slow, consuming CPU, but htop shows low CPU usage
# Hypothesis: Lots of context switching due to lock contention

# Step 1: Check what syscalls it's making
sudo execsnoop | grep myapp

# Step 2: Trace specific syscalls
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_futex { printf("%s\n", comm); }'

# Step 3: Measure syscall latency
sudo biolatency               # Check I/O
sudo profile -F 99 10 | ...   # Generate flamegraph

# Step 4: If seeing many futex syscalls, app has lock contention
# Solution: Review code for inefficient locking patterns
```

### Comparing eBPF vs Traditional Tools

```bash
# Traditional approach: strace (slow, high overhead)
strace -e openat -p 1234      # Slows process significantly

# eBPF approach: opensnoop (low overhead)
opensnoop -p 1234             # Negligible performance impact

# Traditional: perf record + flamegraph
perf record -p 1234 sleep 10  # Can affect timing accuracy
perf script | stackcollapse-perf | flamegraph.pl > result.svg

# eBPF approach: async-profiler or bpf-based profiler
./profiler.sh -d 10 1234      # Always accurate, no statistical bias

# Key advantage: eBPF runs in kernel, no context switches needed
```

---

## ⚠️ Gotchas & Pro Tips

- **Kernel support required**: eBPF needs Linux 4.4+ (ideally 5.0+). Check: `cat /proc/sys/kernel/unprivileged_bpf_disabled` (0 = enabled, 1 = disabled for non-root).

- **bpftrace vs BCC**: bpftrace is simpler one-liners, BCC is more powerful but complex. Start with bpftrace for quick questions.

- **Overhead is minimal but not zero**: eBPF is fast (~few microseconds per trace point), but in very high-frequency scenarios (millions of events/sec), use sampling or aggregation.

- **Stack traces in containers**: eBPF traces work in containers, but symbol resolution may fail if binaries lack debug symbols. Include debug symbols in container images for better traces.

- **Permission requirements**: eBPF requires root or CAP_BPF + CAP_PERFMON. For Kubernetes, run eBPF tools in privileged pods or use privileged DaemonSet.

- **Memory limits on eBPF programs**: Kernel eBPF stack is limited (~500 bytes). Large data structures need BPF maps (persistent key-value stores).

- **Flamegraphs need symbols**: For accurate flamegraphs, ensure binaries have debug symbols (`-g` flag at compile time). Strip symbols in production, or use separate debug builds for profiling.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
