# 🐧 ftrace — Kernel Function Tracing — Linux Mastery

> **Trace any kernel function with zero external tools and near-zero overhead — the Linux kernel's built-in performance microscope.**

## 📖 Concept

`ftrace` (function tracer) is the Linux kernel's built-in tracing framework, accessible via the `tracefs` filesystem at `/sys/kernel/debug/tracing` (or `/sys/kernel/tracing` on newer kernels). Unlike `strace` (which traces syscalls with significant overhead via ptrace) or `perf` (which samples), ftrace can trace individual kernel functions with sub-microsecond precision and near-zero overhead when not actively tracing.

**What makes ftrace special:**
- It's built into the kernel — no additional tools or kernel modules required
- Multiple tracers: function tracer, function graph tracer, event tracer, hardware latency tracer
- Can trace specific functions, filter by PID, measure latency between events
- The `trace-cmd` and `KernelShark` tools provide a friendlier CLI/GUI on top
- Available on every Linux system that has `CONFIG_FTRACE=y` (virtually all modern distros)

**Key components:**
- **Function tracer:** Records every kernel function call (or a filtered subset)
- **Function graph tracer:** Records function calls AND returns with execution time — shows call trees
- **Event tracing:** Records specific kernel events (scheduler, block I/O, network, filesystem operations)
- **Latency tracers:** `wakeup`, `wakeup_rt`, `irqsoff` — measure scheduling and IRQ latencies

This is the tool kernel developers and SREs use to understand what the kernel is actually doing during a performance problem — before reaching for eBPF or SystemTap.

---

## 💡 Real-World Use Cases

- Trace which kernel functions are called during a slow disk write and find the bottleneck
- Measure scheduler wakeup latency for latency-sensitive applications (trading, real-time, K8s control plane)
- Understand exactly what happens in the kernel during a `fork()`, `exec()`, or `open()` syscall
- Debug kernel module behavior by tracing only functions in that module
- Find why IRQs are being disabled for too long on a production system

---

## 🔧 Commands & Examples

### Basic Setup — Accessing tracefs

```bash
# Mount debugfs if not mounted (required to access /sys/kernel/debug/tracing)
mount -t debugfs none /sys/kernel/debug

# tracefs location (two paths — same filesystem)
ls /sys/kernel/debug/tracing/
ls /sys/kernel/tracing/   # newer kernels, no debugfs required

# Key files in tracefs
ls /sys/kernel/debug/tracing/
# trace             — read the trace buffer
# trace_pipe        — stream the trace (blocks, like tail -f)
# current_tracer    — which tracer is active
# available_tracers — list of supported tracers
# tracing_on        — 1=enabled, 0=paused
# set_ftrace_filter — which functions to trace (when using function tracer)
# trace_options     — output formatting options
# events/           — available event categories

# Check available tracers
cat /sys/kernel/debug/tracing/available_tracers
# blk function_graph wakeup_dl wakeup_rt wakeup irqsoff function nop

# Check current tracer (nop = nothing active)
cat /sys/kernel/debug/tracing/current_tracer
# nop
```

### Function Tracer — Trace All Kernel Functions

```bash
# Step 1: Set the tracer
echo function > /sys/kernel/debug/tracing/current_tracer

# Step 2: Enable tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# Step 3: Capture some activity
sleep 1

# Step 4: Stop tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on

# Step 5: Read the trace
cat /sys/kernel/debug/tracing/trace | head -50

# Output format:
# TASK-PID CPU# TIMESTAMP FUNCTION
# bash-12345 [000] 1234.567890: do_sys_open <-do_sys_openat2
# bash-12345 [000] 1234.567891: filp_open <-do_sys_open

# Step 6: Clear the trace buffer before next run
echo > /sys/kernel/debug/tracing/trace

# Step 7: Reset to no tracer when done
echo nop > /sys/kernel/debug/tracing/current_tracer
```

### Filtering Functions

```bash
# List all traceable functions (can be 100,000+)
wc -l /sys/kernel/debug/tracing/available_filter_functions

# Trace only specific functions matching a pattern
echo "ext4_*" > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# Trigger some ext4 activity
touch /tmp/testfile

echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep ext4

# Trace TCP functions
echo "tcp_*" > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
curl -s https://example.com > /dev/null
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | head -30

# Trace scheduler functions
echo "schedule* wake_up*" > /sys/kernel/debug/tracing/set_ftrace_filter

# Trace a specific function
echo "do_sys_open" > /sys/kernel/debug/tracing/set_ftrace_filter

# Trace everything (clear filter)
echo > /sys/kernel/debug/tracing/set_ftrace_filter

# Filter by PID
MY_PID=12345
echo $MY_PID > /sys/kernel/debug/tracing/set_ftrace_pid

# Trace functions NOT in a filter (inverse filter)
echo "do_sys_open" > /sys/kernel/debug/tracing/set_ftrace_notrace
```

### Function Graph Tracer — See Call Trees with Timing

```bash
# Function graph shows entry AND exit, with indentation showing call depth
echo function_graph > /sys/kernel/debug/tracing/current_tracer

# Filter to specific functions for readability
echo "ext4_file_write_iter" > /sys/kernel/debug/tracing/set_graph_function

echo 1 > /sys/kernel/debug/tracing/tracing_on
echo "test content" > /tmp/testfile
echo 0 > /sys/kernel/debug/tracing/tracing_on

cat /sys/kernel/debug/tracing/trace
# Output (indented call tree with timing):
# CPU DURATION    FUNCTION CALLS
# |   |   |       |   |   |
# 0)  2.045 us    |  ext4_file_write_iter() {
# 0)  0.456 us    |    ext4_write_checks();
# 0)  1.123 us    |    generic_write_checks();
# 0)             |    ext4_journal_start_sb() {
# 0)  0.234 us    |      jbd2_journal_start();
# 0)  0.512 us    |    }

# Trace specific functions and their children (--graph-depth to limit nesting)
echo "vfs_write" > /sys/kernel/debug/tracing/set_graph_function

# Limit graph depth to avoid overwhelming output
echo 5 > /sys/kernel/debug/tracing/max_graph_depth
```

### Event Tracing — Structured Kernel Events

```bash
# List available event categories
ls /sys/kernel/debug/tracing/events/
# block  ext4  kmem  net  sched  scsi  skb  syscalls  tcp  vfs  ...

# List events within a category
ls /sys/kernel/debug/tracing/events/sched/

# Enable a specific event
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_wakeup/enable

# Enable all events in a category
echo 1 > /sys/kernel/debug/tracing/events/block/enable

# Start tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 2
echo 0 > /sys/kernel/debug/tracing/tracing_on

# Read structured events
cat /sys/kernel/debug/tracing/trace | grep "sched_switch" | head -20
# sched_switch: prev_comm=bash prev_pid=12345 prev_prio=120
#               prev_state=S ==> next_comm=kworker next_pid=45 next_prio=120

# Enable syscall events (enter and exit for every syscall)
echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_enter_openat/enable
echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_exit_openat/enable

# Disable all events
echo 0 > /sys/kernel/debug/tracing/events/enable

# Event filters — only record events matching a condition
echo 'pid == 12345' > /sys/kernel/debug/tracing/events/sched/sched_switch/filter
echo 'bytes_written > 4096' > /sys/kernel/debug/tracing/events/ext4/ext4_da_write_begin/filter
```

### Using trace-cmd (friendlier interface)

```bash
# Install trace-cmd
apt-get install trace-cmd    # Debian/Ubuntu
yum install trace-cmd        # RHEL/CentOS

# Record function graph for 5 seconds
trace-cmd record -p function_graph -g vfs_write sleep 5

# Record specific events for 3 seconds
trace-cmd record -e sched:sched_switch -e block:block_rq_issue sleep 3

# Record events for a specific PID
trace-cmd record -e all -P 12345 sleep 5

# Report the recorded trace
trace-cmd report | head -50

# Stream live events to stdout
trace-cmd stream -e sched:sched_switch

# Record and show function call latency histogram
trace-cmd record -p function -l do_sys_open sleep 2
trace-cmd report --cpu 0

# List available events
trace-cmd list -e | grep tcp

# Show function histogram (how many times each function was called)
trace-cmd record -p function sleep 1
trace-cmd report -f 2>/dev/null | awk '{print $NF}' | sort | uniq -c | sort -rn | head -20
```

### Latency Tracing — IRQ and Scheduling

```bash
# irqsoff tracer — measures max time IRQs are disabled
echo irqsoff > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 2
echo 0 > /sys/kernel/debug/tracing/tracing_on

# Shows the worst-case IRQ-disabled window with a full backtrace
cat /sys/kernel/debug/tracing/trace | head -40
# Latency: 42 us, #4/4, CPU#0 | (M:preempt VP:0, KP:0, SP:0 HP:0 #P:8)
#    -----------------
#    | task: bash-12345 (uid:0 nice:0 policy:0 rt_prio:0)
#    -----------------
#    => 42 us maximum latency

# wakeup tracer — time from wakeup to scheduled (scheduling latency)
echo wakeup > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# Run some latency-sensitive workload
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | head -20

# wakeup_rt — only measures RT (real-time) task wakeup latency
echo wakeup_rt > /sys/kernel/debug/tracing/current_tracer
```

### Practical Script — Tracing Block I/O Latency

```bash
#!/bin/bash
# Trace block I/O events for a specific operation to measure latency

set -euo pipefail

TRACE_DIR="/sys/kernel/debug/tracing"
DURATION="${1:-5}"

echo "Setting up block I/O event tracing for ${DURATION} seconds..."

# Reset
echo nop > "${TRACE_DIR}/current_tracer"
echo > "${TRACE_DIR}/trace"
echo 0 > "${TRACE_DIR}/events/enable"

# Enable block I/O events
echo 1 > "${TRACE_DIR}/events/block/block_rq_issue/enable"    # request issued
echo 1 > "${TRACE_DIR}/events/block/block_rq_complete/enable" # request completed

# Start tracing
echo 1 > "${TRACE_DIR}/tracing_on"
echo "Tracing for ${DURATION} seconds... Do your I/O-heavy operation now."
sleep "$DURATION"
echo 0 > "${TRACE_DIR}/tracing_on"

echo ""
echo "=== Block I/O Events (last 30) ==="
grep -E "block_rq_(issue|complete)" "${TRACE_DIR}/trace" | tail -30

echo ""
echo "=== Summary ==="
echo "Total I/O requests issued: $(grep -c block_rq_issue "${TRACE_DIR}/trace" 2>/dev/null || echo 0)"
echo "Total I/O requests completed: $(grep -c block_rq_complete "${TRACE_DIR}/trace" 2>/dev/null || echo 0)"

# Cleanup
echo 0 > "${TRACE_DIR}/events/block/block_rq_issue/enable"
echo 0 > "${TRACE_DIR}/events/block/block_rq_complete/enable"
echo nop > "${TRACE_DIR}/current_tracer"
```

---

## ⚠️ Gotchas & Pro Tips

- **Always reset the tracer when done:** Leave the tracer as `nop` and disable all events (`echo 0 > events/enable`) when finished. An active function tracer can add measurable overhead (1-5% CPU) and fill the trace buffer quickly.

- **The trace buffer is circular — data gets overwritten:** The default buffer size is 7MB per CPU. For long traces, either increase the buffer (`echo 65536 > buffer_size_kb` sets 64MB per CPU) or stream with `trace_pipe` instead of reading `trace` after the fact.

- **Function tracing overhead is proportional to function call rate:** Tracing `kmalloc` or `schedule` — which are called millions of times per second — will cause significant overhead. Always filter to specific functions you care about.

- **`trace_pipe` is consumed, `trace` is not:** Reading from `trace_pipe` removes events from the buffer (streaming). Reading from `trace` reads without consuming. Use `trace_pipe` for live monitoring, `trace` for post-capture analysis.

- **Requires root:** All tracefs operations need root. In containerized environments (K8s pods), you need `--privileged` and `CAP_SYS_ADMIN` to access tracefs. This limits ftrace use in containers compared to eBPF (which has safer sandboxing).

- **`trace-cmd` makes this much easier:** The raw tracefs interface is powerful but verbose. For anything beyond a quick one-off, use `trace-cmd record` + `trace-cmd report`. For visualization, `KernelShark` renders the data as a timeline GUI.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
