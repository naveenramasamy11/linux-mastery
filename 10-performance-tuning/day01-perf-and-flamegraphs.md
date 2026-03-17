# 🐧 perf and Flamegraphs — Linux Mastery

> **Master CPU profiling with perf and flamegraph visualization—the tools that turn "why is this slow?" into actionable insights.**

## 📖 Concept

Performance tuning without profiling is guessing. The `perf` tool (Linux performance events subsystem) provides statistical sampling of CPU activity, hardware events, and call stacks. Instead of measuring wall time, perf samples the instruction pointer and call stack thousands of times per second, revealing where CPU time actually goes. Flamegraphs visualize this data: the wider the bar, the more CPU time spent. The height represents call stack depth.

For DevOps and SRE, perf is the difference between a vague "application is slow" and "85% of CPU is in memcpy during JSON serialization—let's batch operations." It works on production systems with negligible overhead (typical 1-3% impact). Combined with flamegraphs, it's the industry-standard approach for finding performance bottlenecks.

The workflow is simple: `perf record` collects samples, `perf report` shows results interactively, and flamegraph.pl converts output to interactive SVGs. For CPU-bound performance issues, this is your go-to toolset. For I/O and latency issues, other tools (biolatency, tcpstat) are better, but perf handles everything.

---

## 💡 Real-World Use Cases

- **Production slow-down diagnosis**: Sample CPU on live service, identify function causing issues, fix without guessing
- **Before/after optimization comparison**: Compare flamegraphs before and after code changes to verify improvement
- **Microservice bottleneck analysis**: In distributed system, identify which service's CPU is the constraint
- **Container CPU throttling**: Detect if containers are CPU-limited by examining where CPU time goes
- **Lock contention detection**: Identify if application is spending time in synchronization primitives (mutex, spinlock)
- **GC pause analysis**: For JVM/Go applications, see if garbage collection dominates CPU usage
- **Hardware counter analysis**: Monitor cache misses, branch misses, CPU stalls to find low-level bottlenecks

---

## 🔧 Commands & Examples

### perf stat: Hardware Event Counting

```bash
# Count hardware events during execution
perf stat /path/to/command arg1 arg2

# Example: count events for a web request
perf stat curl -s http://localhost:8080/api

# Output shows:
# Performance counter stats for 'curl -s http://localhost:8080/api':
#
#   1234.567123 task-clock:u (msec)   # CPU time in milliseconds
#     0          context-switches:u     # Voluntary context switches
#     0          cpu-migrations:u       # Migrations to different CPU
#   5467        page-faults:u           # Memory page faults
# 2341230123    cycles:u                # CPU cycles
# 1234567890    instructions:u          # Instructions executed
#        0.45   insn per cycle          # Instruction efficiency
# 1234567890    cache-references:u      # L1/L2/L3 cache accesses
#   12345678    cache-misses:u (2.31%)  # Cache misses
#

# Key metrics:
# - insn per cycle: CPU efficiency (ideal = 2-4 on modern CPUs)
# - cache-misses: High % = memory bottleneck
# - context-switches: High = lock contention or I/O
# - page-faults: High = memory allocation stress
```

### perf record: Sampling Call Stacks

```bash
# Record CPU samples with call stacks
perf record -g /path/to/command arg1 arg2
# Creates perf.data file with samples

# Options:
# -g: record call graph (stack traces)
# -F 99: sample at 99 Hz (once per 10ms, avoid multiples of 100Hz for bias)
# -p PID: sample running process
# -a: sample all CPUs system-wide
# -c N: sample every Nth event (reduce overhead)

# Example: sample running web server
perf record -g -p $(pgrep -f 'python.*app.py') -a sleep 30
# Collects samples for 30 seconds while app runs

# Example: sample with frequency
perf record -F 99 -g ./myapp arg1 arg2
```

### perf report: Interactive Analysis

```bash
# Analyze recorded samples interactively
perf report

# Interface shows:
# - Percentage of samples in each function
# - Call stacks (expand with Enter)
# - Source/assembly view (if symbols available)

# Example:
# Samples: 12K of event 'cpu-clock:ppp'
# Event count (approx.): 1200000000
#
# Overhead  Command  Shared Object        Symbol
# ========  =======  ===================  =======================
#   42.31%  python   libm-2.31.so        __libc_sin
#   18.50%  python   python               _json_encode
#   14.20%  python   python               [JIT] loop_iteration
#    9.10%  python   libpthread-2.31.so  pthread_mutex_lock

# Top function (__libc_sin) uses 42% of CPU
# This is your optimization target

# Navigation in perf report:
# Enter: expand/collapse function
# s: sort by different columns
# q: quit
# /: search

# Export for analysis
perf report -H --stdio > report.txt
```

### Flamegraph Generation

```bash
# Install flamegraph tools
git clone https://github.com/brendangregg/FlameGraph
cd FlameGraph

# Generate flamegraph from perf.data
perf script | ./stackcollapse-perf.pl > out.collapsed
./flamegraph.pl out.collapsed > cpu.svg

# Open in browser (interactive SVG)
firefox cpu.svg

# Flamegraph interpretation:
# - x-axis: time (or sample count, no time dimension)
# - y-axis: call stack depth
# - width: CPU time (wider = more time)
# - color: function (random for readability)
# - click to zoom
# - search function with Ctrl+F

# Example workflow:
perf record -F 99 -g -p PID -a sleep 60
perf script > /tmp/perf.txt
/path/to/FlameGraph/stackcollapse-perf.pl /tmp/perf.txt | \
    /path/to/FlameGraph/flamegraph.pl > cpu.svg
# Result: cpu.svg ready to open
```

### Diff Flamegraphs: Before/After Optimization

```bash
# Collect baseline
perf record -F 99 -g ./app --config=baseline sleep 30
perf script > baseline.txt
./stackcollapse-perf.pl baseline.txt > baseline.collapsed

# Collect after optimization
perf record -F 99 -g ./app --config=optimized sleep 30
perf script > optimized.txt
./stackcollapse-perf.pl optimized.txt > optimized.collapsed

# Generate diff flamegraph
./flamegraph.pl --color=diff baseline.collapsed optimized.collapsed > diff.svg

# Result: red bars = slower, blue bars = faster
# Quickly see which functions improved/regressed
```

### Advanced: CPU Profiling Options

```bash
# Profile all events (CPU, cache, branch)
perf stat -e cycles,instructions,cache-misses,branch-misses ./app

# Count specific syscalls
perf stat -e syscalls:sys_enter_* ./app

# Memory allocation tracking (requires USDT probes)
perf record -e malloc:malloc ./app

# Java/JVM profiling (requires JVM symbols)
perf record -g -p $(pgrep java) sleep 30
# Note: JVM may need special configuration for symbols
# Use async-profiler instead for better Java support

# Kernel tracing (advanced)
perf record -e sched:sched_switch -g ./app sleep 30
# Shows context switches and which functions caused them
```

### Real-World Example: Web Server Profiling

```bash
# Slow endpoint: GET /api/users takes 2 seconds
# Hypothesis: database query too slow, or JSON encoding bottleneck?

# Step 1: Collect profile during load
ab -n 1000 -c 50 http://localhost:8080/api/users &
sleep 1
perf record -F 99 -g -p $(pgrep python) sleep 10

# Step 2: Analyze
perf report
# Output shows 45% in __libc_sin (JSON encoding using math)
# 35% in libpq (database client library)
# 20% in application code

# Diagnosis:
# - JSON encoding is biggest bottleneck (45%)
# - Use faster JSON library: ujson instead of json
# - Reduce database queries: batch them together
# - Cache database results: add Redis

# Step 3: Verify improvement
# Implement changes
perf record -F 99 -g -p $(pgrep python) sleep 10
perf report
# Now: 15% JSON encoding, 25% database, 60% waiting on I/O (good)

# Step 4: Diff shows improvement
perf script | stackcollapse-perf | \
    flamegraph.pl --color=diff > improvement.svg
# Blue bars = improved (smaller)
```

### Container and Kubernetes Profiling

```bash
# Profile process inside container
docker run -d --name myapp myimage:latest
PID=$(docker inspect -f '{{.State.Pid}}' myapp)
perf record -F 99 -g -p $PID -a sleep 30

# Analyze as normal
perf report

# In Kubernetes: profile pod on node
kubectl get pod myapp -o wide  # Find node
ssh NODE
docker ps | grep myapp         # Get container ID
PID=$(docker inspect -f '{{.State.Pid}}' CONTAINER_ID)
perf record -F 99 -g -p $PID sleep 30
perf script | flamegraph.pl > cpu.svg
# Download svg to view

# Important: Kernel symbols may not resolve in containers
# Ensure container image includes debug symbols
# Or run perf on host pointing to container symbols
```

### Flamegraph Best Practices

```bash
# Symbol resolution: ensure binaries aren't stripped
objdump -t /path/to/binary | grep -c symtab
# High count = good (symbols available)

# For Go binaries
perf record -g ./go_binary sleep 30
# Go uses DWARF debug info, usually available

# For Python
python -m pdb my_script.py
# Or use py-spy (Python-specific profiler)

# For C/C++
gcc -g -O2 myapp.c  # -g keeps symbols
perf record -g ./a.out

# Remove noise: filter common functions
perf script | grep -v '\[kernel\]' | stackcollapse-perf | \
    flamegraph.pl > cpu_userspace_only.svg

# Zoom on specific function
perf report -g graph
# Then drill down on interesting function
```

### Performance Profiling Workflow

```bash
#!/bin/bash
# Standard perf profiling workflow

APP_NAME="myapp"
SAMPLE_TIME=60
FREQUENCY=99

echo "Starting profiling of $APP_NAME..."

# Collect samples
perf record \
    -F $FREQUENCY \
    -g \
    -o perf.data \
    -p $(pgrep -f "$APP_NAME") \
    -a \
    sleep $SAMPLE_TIME

echo "Collecting samples... done"

# Generate flamegraph
perf script > /tmp/perf.txt
stackcollapse-perf.pl /tmp/perf.txt > /tmp/perf.collapsed
flamegraph.pl /tmp/perf.collapsed > cpu-profile-$(date +%Y%m%d-%H%M%S).svg

echo "Flamegraph generated: cpu-profile-*.svg"

# Also generate text report
perf report -H --stdio > report-$(date +%Y%m%d-%H%M%S).txt
echo "Report generated: report-*.txt"
```

---

## ⚠️ Gotchas & Pro Tips

- **Sampling frequency matters**: 99 Hz is typical (avoids biasing multiples of 100Hz timer). Higher (999 Hz) = more accurate but higher overhead. Lower (19 Hz) = less overhead but noisier data.

- **Symbols required for readable output**: If binary is stripped, perf shows addresses instead of function names. Compile with `-g` or install debug packages (linux-image-*-dbgsym).

- **Flamegraph width is NOT time**: x-axis shows sample count, not wall time. A wider function didn't take longer; it was on the call stack in more samples.

- **Kernel overhead in flamegraph**: If sampling kernel code (syscalls, context switches), it appears in flamegraph. This is real overhead, not a profiler artifact.

- **perf needs permissions**: Non-root profiling may fail. Either run as root, set `perf_event_paranoid`, or use capabilities: `setcap cap_perfmon,cap_bpf+ep /usr/bin/perf`.

- **Function inlining hides detail**: Compiler inlining moves work into parent function. Recompile with `-fno-inline` for profiling to see actual function names.

- **JVM profiling is special**: Java's JIT compilation means symbols aren't always available. Use async-profiler or jfr (Java Flight Recorder) for best results.

- **Container profiling requires host symbols**: perf in container may not resolve symbols from host binaries. Run perf on the host pointing to container PID instead.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
