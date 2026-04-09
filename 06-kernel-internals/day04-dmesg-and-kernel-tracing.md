# 🐧 dmesg & Kernel Tracing — Linux Mastery

> **When a system behaves strangely and no application logs explain it, dmesg and the kernel tracing infrastructure are where the truth lives.**

## 📖 Concept

The Linux kernel maintains a circular ring buffer of messages called the **kernel log buffer**. Every driver load, hardware event, OOM kill, filesystem error, network anomaly, and security event produces a message that lands here. `dmesg` is the userspace tool to read it. On modern systems with systemd, these messages are also forwarded to the journal (`journalctl -k`), but `dmesg` gives you the raw, unformatted kernel perspective.

Understanding dmesg output is a core SRE skill. The messages contain timestamps (in seconds since boot), severity levels, subsystem prefixes, and structured data. Patterns like `Out of memory: Kill process`, `EXT4-fs error`, `TCP: Possible SYN flooding`, `segfault`, `SCSI error`, or `kernel BUG at` are diagnostic signals that no application-level monitoring can replace. On EC2 instances, the console log visible in the AWS console is literally the kernel ring buffer output — the same thing dmesg shows you.

Beyond the ring buffer, the kernel has a rich **tracing infrastructure** rooted in `/sys/kernel/debug/tracing` (debugfs). This is the foundation that powers `ftrace`, `perf`, BCC/BPF tools, and `trace-cmd`. ftrace is the kernel's built-in function tracer — it can hook any kernel function (there are tens of thousands of tracepoints and kprobes available) with essentially zero overhead when disabled and very low overhead when active. Kernel tracing lets you answer questions that no profiler can: why is my kernel spending time in `ext4_journal_check_start`? what exact syscall sequence is causing this latency spike?

---

## 💡 Real-World Use Cases

- Diagnose an EC2 instance kernel panic by reading the console log for OOM kill messages and stack traces
- Identify a hardware disk error causing filesystem corruption by spotting `blk_update_request: I/O error` in dmesg
- Use ftrace to identify which kernel function is causing unexpected latency in a high-performance trading or ML workload
- Detect TCP SYN flood attacks or connection table exhaustion from kernel-level network messages
- Trace all `open()` syscalls made by a suspicious process without installing any additional tools

---

## 🔧 Commands & Examples

### dmesg Basics

```bash
# Read the kernel ring buffer
dmesg

# With human-readable timestamps (requires kernel 3.5+ for --human)
dmesg --human
dmesg -T    # same: translate timestamps to human-readable

# Show timestamps as ISO 8601
dmesg -T --time-format=iso

# Follow in real time (like tail -f)
dmesg -w
dmesg --follow

# Show only recent messages (last N lines)
dmesg | tail -50
dmesg -T | tail -100

# Filter by facility/level
dmesg --level=err,crit,alert,emerg    # only errors and above
dmesg --facility=kern                  # kernel messages only

# Color output by severity (requires modern dmesg from util-linux 2.23+)
dmesg --color=always | less -R

# Clear the ring buffer (requires root)
dmesg -C
```

### Reading dmesg Severity Levels

dmesg messages have a priority prefix like `<3>` (KERN_ERR) or `<6>` (KERN_INFO):

```bash
# Levels: 0=EMERG 1=ALERT 2=CRIT 3=ERR 4=WARN 5=NOTICE 6=INFO 7=DEBUG

# Show errors only
dmesg -l err
dmesg -l warn,err

# Show all messages at warning level and above
dmesg -l emerg,alert,crit,err,warn

# Practical filter — show errors with context
dmesg -T | grep -E "(error|fail|warn|oom|kill)" -i | tail -30
```

### Key dmesg Patterns to Know

```bash
# OOM Killer activation
dmesg | grep -i "oom\|out of memory\|killed process"
# Look for:
# Out of memory: Kill process 12345 (java) score 892 or sacrifice child
# oom_kill_process+0x... (stack trace follows)

# Disk I/O errors (EBS issue, failing disk)
dmesg | grep -i "i/o error\|blk_update_request\|SCSI error\|sense key"

# Filesystem errors
dmesg | grep -i "ext4\|xfs\|btrfs" | grep -i "error\|corrupt\|journal"

# Network issues
dmesg | grep -i "nf_conntrack: table full\|TCP: too many orphaned\|SYN flooding"

# CPU/hardware issues
dmesg | grep -i "mce\|machine check\|hardware error\|NMI"

# Kernel BUG / panic (check this first on any crashed system)
dmesg | grep -E "BUG:|WARNING:|Oops:|kernel BUG|RIP:|Call Trace"

# Driver / module issues
dmesg | grep -i "firmware\|module\|driver" | grep -i "fail\|error"

# Check time since last boot and message age
dmesg -T | head -5   # first messages give boot time context
```

### journalctl for Kernel Messages

```bash
# Kernel messages via systemd journal (equivalent to dmesg on systemd systems)
journalctl -k                          # kernel messages from current boot
journalctl -k -b -1                    # kernel messages from previous boot
journalctl -k -b -1 -p err..emerg      # errors from last boot (great for post-crash analysis)
journalctl -k --since "1 hour ago"
journalctl -k -f                       # follow in real time

# Compare: dmesg shows ring buffer (may wrap), journal persists across boots
# For crash analysis: use journalctl -k -b -1 after reboot
```

### ftrace — Kernel Function Tracing

ftrace lives in `/sys/kernel/debug/tracing`. You enable it by writing to control files.

```bash
# Mount debugfs if not already mounted
mount -t debugfs none /sys/kernel/debug

# Base path shortcut
TRACE=/sys/kernel/debug/tracing

# Check available tracers
cat $TRACE/available_tracers
# output: function function_graph blk mmiotrace nop

# Check current tracer
cat $TRACE/current_tracer

# List all available trace events (thousands of them)
ls $TRACE/events/
ls $TRACE/events/syscalls/
ls $TRACE/events/net/
ls $TRACE/events/ext4/

# Check available filter functions
wc -l $TRACE/available_filter_functions
```

### ftrace Function Tracer

```bash
TRACE=/sys/kernel/debug/tracing

# Enable function tracer
echo function > $TRACE/current_tracer

# Trace a specific function (e.g., tcp_sendmsg)
echo 'tcp_sendmsg' > $TRACE/set_ftrace_filter

# Start tracing
echo 1 > $TRACE/tracing_on

# Do your workload / trigger the event

# Stop tracing
echo 0 > $TRACE/tracing_on

# Read the trace
cat $TRACE/trace | head -50

# Clear the trace buffer
echo > $TRACE/trace

# Trace all calls to functions matching a glob
echo 'ext4_*' > $TRACE/set_ftrace_filter
echo function > $TRACE/current_tracer
echo 1 > $TRACE/tracing_on
# ... do something with ext4 ...
echo 0 > $TRACE/tracing_on
cat $TRACE/trace
```

### ftrace Function Graph — Visualise Call Chains

```bash
TRACE=/sys/kernel/debug/tracing

# Function graph tracer shows call depth and duration
echo function_graph > $TRACE/current_tracer

# Limit to a specific function entry point (trace all callees)
echo 'do_sys_open' > $TRACE/set_graph_function

echo 1 > $TRACE/tracing_on
# open a file
ls /etc
echo 0 > $TRACE/tracing_on

cat $TRACE/trace
# Output looks like:
# 0)               |  do_sys_open() {
# 0)               |    getname() {
# 0)   0.312 us    |      kmem_cache_alloc();
# 0)   1.243 us    |    } /* getname */
# 0)               |    alloc_fd() {
# ...
```

### Tracepoints — Structured Event Tracing

```bash
TRACE=/sys/kernel/debug/tracing

# Enable the sched_process_exec tracepoint (fires on execve)
echo 1 > $TRACE/events/sched/sched_process_exec/enable

# Enable all syscall enter events
echo 1 > $TRACE/events/syscalls/enable

# Enable specific syscall tracepoint
echo 1 > $TRACE/events/syscalls/sys_enter_openat/enable
echo 1 > $TRACE/events/syscalls/sys_enter_connect/enable

# Set a filter (only trace PID 1234)
echo 'common_pid == 1234' > $TRACE/events/syscalls/sys_enter_openat/filter

echo 1 > $TRACE/tracing_on
sleep 5
echo 0 > $TRACE/tracing_on
cat $TRACE/trace | grep openat | head -20

# Disable all events
echo 0 > $TRACE/events/enable
```

### trace-cmd — Easier ftrace Wrapper

`trace-cmd` is a convenient CLI wrapper around the raw ftrace interface.

```bash
# Install
yum install trace-cmd       # RHEL/Amazon Linux
apt install trace-cmd       # Debian/Ubuntu

# Record syscalls made by PID 1234 for 5 seconds
trace-cmd record -p function -P 1234 sleep 5

# Record sched events system-wide for 3 seconds
trace-cmd record -e sched sleep 3

# Record all open/connect syscalls
trace-cmd record -e syscalls:sys_enter_openat -e syscalls:sys_enter_connect sleep 10

# View the report
trace-cmd report | head -100

# Filter to a specific process
trace-cmd report | grep -A2 "myprocess"

# Live stream (like dmesg -w but for trace events)
trace-cmd stream -e sched:sched_process_fork
```

### Practical Diagnostic Workflows

```bash
# WORKFLOW 1: Post-crash OOM analysis
# After a reboot, check what got killed and why
journalctl -k -b -1 | grep -A 20 "Out of memory"
# Look for: which process, its score, system memory state at time of kill

# WORKFLOW 2: Find what's opening too many files
TRACE=/sys/kernel/debug/tracing
echo 1 > $TRACE/events/syscalls/sys_enter_openat/enable
echo 1 > $TRACE/tracing_on
sleep 10
echo 0 > $TRACE/tracing_on
# Count opens per process
awk '/openat/ {print $1}' $TRACE/trace | sort | uniq -c | sort -rn | head -10
echo 0 > $TRACE/events/enable

# WORKFLOW 3: Trace all TCP connections being established
trace-cmd record -e net:net_dev_xmit -e tcp:tcp_probe sleep 30
trace-cmd report | grep "ESTABLISHED"

# WORKFLOW 4: Kernel latency analysis with wakeup tracer
echo wakeup > $TRACE/current_tracer
echo 1 > $TRACE/tracing_on
sleep 1
echo 0 > $TRACE/tracing_on
cat $TRACE/trace | head -30
# Shows max scheduling latency — important for latency-sensitive workloads

# WORKFLOW 5: Monitor disk I/O errors in real time
dmesg -w | grep -i "i/o error\|blk_update_request" &
# or
journalctl -kf | grep -i "error"
```

### Ring Buffer Size and Performance

```bash
# Check current ring buffer size per CPU
cat /sys/kernel/debug/tracing/buffer_size_kb   # per-CPU buffer

# Increase buffer size (useful for capturing longer traces)
echo 65536 > /sys/kernel/debug/tracing/buffer_size_kb   # 64MB per CPU

# Check total buffer usage
cat /sys/kernel/debug/tracing/buffer_total_size_kb

# Check if events are being dropped (overrun counter)
cat /sys/kernel/debug/tracing/per_cpu/cpu0/stats | grep overrun
```

---

## ⚠️ Gotchas & Pro Tips

- **dmesg timestamps are since boot, not wall clock:** On a system that's been running for months, `dmesg` timestamps are large numbers. Use `dmesg -T` to see real dates/times. Note: on systems with time adjustments (NTP, VM live migration), the correlation can be slightly off.
- **Ring buffer wraps:** The kernel ring buffer has a fixed size (default ~512KB per CPU, varies by kernel config). On a system generating many kernel messages, old entries are overwritten. If you're investigating a crash that happened days ago, use `journalctl -k -b -1` — the journal persists across reboots.
- **ftrace overhead:** The function tracer adds overhead to every instrumented function call — potentially 10-30% CPU overhead when tracing broad function sets. Always be specific with `set_ftrace_filter`. For production, use tracepoints (much lower overhead) or eBPF tools.
- **Always reset ftrace after use:** It's easy to leave ftrace enabled and forget. `echo nop > /sys/kernel/debug/tracing/current_tracer` and `echo 0 > /sys/kernel/debug/tracing/tracing_on` resets it cleanly.
- **`dmesg` output in AWS console log:** The EC2 system log (visible in the AWS console or via `aws ec2 get-console-output`) is the kernel ring buffer. If an instance becomes unreachable, this is often the first place to look for panic messages.
- **OOM score:** The OOM killer selects processes by `oom_score`. You can protect critical processes: `echo -1000 > /proc/PID/oom_score_adj` (minimum — almost never killed). Use this for container runtimes, monitoring agents, and system services.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
