# 🔬 Day 01 — Kernel Internals: /proc, /sys, strace & sysctl

> The kernel exposes everything about itself through virtual filesystems. Most people never look.

---

## /proc — The Live Kernel Window

`/proc` is a virtual filesystem — nothing on disk. It's the kernel talking to you in real time.

```bash
# System-wide info
cat /proc/cpuinfo           # CPU details (cores, flags, MHz)
cat /proc/meminfo           # Memory breakdown (MemFree, Cached, Buffers...)
cat /proc/loadavg           # Load averages + running/total processes
cat /proc/uptime            # Uptime in seconds (with idle time)
cat /proc/version           # Kernel version string
cat /proc/cmdline           # Kernel boot parameters!
cat /proc/filesystems       # Supported filesystems
cat /proc/mounts            # Currently mounted filesystems
cat /proc/swaps             # Swap partitions/files
cat /proc/interrupts        # Hardware interrupts per CPU
cat /proc/net/dev           # Network I/O stats per interface
cat /proc/net/tcp           # TCP connections (hex format)
cat /proc/diskstats         # Disk I/O statistics
```

### Decode /proc/net/tcp

```bash
# Column 3 = local address (hex), Column 4 = remote address (hex)
# Convert hex port: 0050 → 80, 01BB → 443

# Pretty-print active TCP connections
awk 'NR>1 {split($2,l,":"); split($3,r,":");
  printf "Local: %d.%d.%d.%d:%d  Remote: %d.%d.%d.%d:%d\n",
  strtonum("0x"substr(l[1],7,2)), strtonum("0x"substr(l[1],5,2)),
  strtonum("0x"substr(l[1],3,2)), strtonum("0x"substr(l[1],1,2)),
  strtonum("0x"l[2]),
  strtonum("0x"substr(r[1],7,2)), strtonum("0x"substr(r[1],5,2)),
  strtonum("0x"substr(r[1],3,2)), strtonum("0x"substr(r[1],1,2)),
  strtonum("0x"r[2])}' /proc/net/tcp
```

---

## /proc/PID — Per-Process Kernel Data

```bash
PID=1234

cat /proc/$PID/status        # State, memory (VmRSS, VmPeak), threads
cat /proc/$PID/cmdline | xargs -0 echo   # Full command line
cat /proc/$PID/environ | tr '\0' '\n'    # Environment variables
cat /proc/$PID/maps          # Virtual memory map (libs, heap, stack)
cat /proc/$PID/smaps         # Detailed memory breakdown per mapping
cat /proc/$PID/limits        # ulimit settings for the process
cat /proc/$PID/io            # I/O counters (read_bytes, write_bytes)
cat /proc/$PID/net/tcp       # Network connections scoped to process (namespaces)
ls -la /proc/$PID/fd/        # Open file descriptors → see open files/sockets
ls -la /proc/$PID/exe        # Symlink to the actual binary
ls -la /proc/$PID/cwd        # Current working directory

# Memory usage breakdown
grep VmRSS /proc/$PID/status   # Resident Set Size (actual RAM used)
grep VmSwap /proc/$PID/status  # Swapped memory
grep Threads /proc/$PID/status # Thread count
```

---

## /sys — Hardware & Kernel Tuning Interface

```bash
# CPU information
ls /sys/devices/system/cpu/
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor   # CPU frequency policy
echo "performance" > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Disable/enable a CPU (for testing NUMA, etc.)
echo 0 > /sys/devices/system/cpu/cpu3/online    # Offline cpu3
echo 1 > /sys/devices/system/cpu/cpu3/online    # Bring it back

# Network tuning via /sys
cat /sys/class/net/eth0/speed           # Interface speed (Mbps)
cat /sys/class/net/eth0/operstate       # "up" or "down"
cat /sys/class/net/eth0/statistics/rx_bytes  # Received bytes

# Block device scheduler
cat /sys/block/sda/queue/scheduler
echo "deadline" > /sys/block/sda/queue/scheduler  # Change I/O scheduler

# VM (virtual memory) settings
cat /sys/kernel/mm/transparent_hugepage/enabled
echo "never" > /sys/kernel/mm/transparent_hugepage/enabled  # Disable THP (for databases!)
```

---

## sysctl — Tune the Running Kernel

```bash
# View all kernel parameters
sysctl -a

# View specific parameters
sysctl net.ipv4.ip_forward
sysctl vm.swappiness
sysctl kernel.pid_max

# Set a parameter (immediate, not persistent)
sysctl -w net.ipv4.ip_forward=1
sysctl -w vm.swappiness=10

# Persistent changes
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
echo "vm.swappiness = 10" >> /etc/sysctl.conf
sysctl -p    # Apply from file

# IMPORTANT SETTINGS TO KNOW
# ─────────────────────────────────────────────────────
# vm.swappiness=10          → Use swap only as last resort (default: 60)
# vm.dirty_ratio=10         → % of RAM before sync dirty pages
# kernel.shmmax             → Max shared memory segment (databases!)
# net.ipv4.tcp_syncookies=1 → Protect against SYN flood
# net.ipv4.ip_forward=1     → Enable routing (required for containers!)
# net.core.somaxconn=65535  → Max TCP connections (high-traffic servers)
# fs.file-max=2097152       → Max open files system-wide
# kernel.pid_max=4194304    → Max PIDs
# net.ipv4.tcp_tw_reuse=1   → Reuse TIME_WAIT sockets (high connection rate)
```

---

## strace — Spy on System Calls

`strace` traces system calls made by a process — invaluable for debugging.

```bash
# Trace a new process
strace ls /tmp

# Trace a running process
strace -p $PID

# Summary mode (count syscalls)
strace -c ls /tmp
strace -c -p $PID

# Trace only specific syscalls
strace -e trace=open,read,write ls
strace -e trace=network curl https://example.com
strace -e trace=file python3 app.py    # All file-related syscalls
strace -e trace=process bash script.sh # Process creation

# Show timestamps
strace -t ls /tmp       # Absolute time
strace -T ls /tmp       # Time spent in each syscall
strace -r ls /tmp       # Relative timestamps

# Follow forks/threads
strace -f ./myapp       # Follow child processes
strace -ff -o /tmp/trace ./myapp   # Separate file per PID

# Debugging a crashed app
strace -e signal program 2>&1 | grep -i signal
```

---

## ltrace — Library Call Tracer

```bash
# Trace library calls (glibc, etc.)
ltrace ./binary
ltrace -p $PID

# Trace only specific functions
ltrace -e malloc,free,strcpy ./binary

# Show timestamps and statistics
ltrace -c ./binary
```

---

## Kernel Modules

```bash
# List loaded modules
lsmod

# Module info
modinfo ext4
modinfo tcp_bbr

# Load a module
modprobe tcp_bbr
modprobe overlay             # Required for Docker!

# Load with parameters
modprobe usbcore usbfs_memory_mb=200

# Unload
modprobe -r tcp_vegas

# Prevent a module from loading (blacklist)
echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf

# Enable TCP BBR (better congestion control — great for cloud!)
modprobe tcp_bbr
echo "tcp_bbr" >> /etc/modules-load.d/modules.conf
sysctl -w net.ipv4.tcp_congestion_control=bbr
sysctl -w net.core.default_qdisc=fq
```

---

## dmesg — Kernel Ring Buffer

```bash
# View recent kernel messages
dmesg
dmesg -T             # With human-readable timestamps
dmesg | tail -50
dmesg -w             # Follow mode (like tail -f)

# Filter by level
dmesg --level err    # Errors only
dmesg --level warn,err

# Filter by facility
dmesg --facility kern    # Kernel messages

# Common things to look for
dmesg | grep -i "oom"       # Out of Memory killer activity!
dmesg | grep -i "error"     # Disk/hardware errors
dmesg | grep -i "usb"       # USB device events
dmesg | grep -i "dropped"   # Network packet drops
dmesg | grep -i "segfault"  # Application crashes
dmesg | grep -i "call trace" # Kernel oops/panics
```

---

## eBPF — Modern Kernel Observability

```bash
# Install BCC tools
apt install bpfcc-tools

# What system calls are being made?
execsnoop-bpfcc          # Trace exec() calls
opensnoop-bpfcc          # Trace open() calls
tcpconnect-bpfcc         # Trace new TCP connections
tcpaccept-bpfcc          # Trace accepted TCP connections
biolatency-bpfcc         # Block I/O latency histogram
profile-bpfcc -F 99 10   # CPU profiler, 99Hz, 10 seconds

# Install bpftrace for custom tracing
apt install bpftrace

# Count syscalls per second
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# Trace file opens by process name
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'

# DNS query tracer
bpftrace -e 'kprobe:udp_sendmsg { printf("%s\n", comm); }'
```

---

> **Next:** [Day 02 — Performance Tuning](../10-performance-tuning/)
