# 🐧 Kernel Modules & sysctl Production Tuning — Linux Mastery

> **Kernel modules let you load driver code at runtime without rebooting; sysctl lets you tune kernel behaviour in milliseconds — together they give you control over the lowest layer of your infrastructure stack without ever taking an instance offline.**

## 📖 Concept

### Kernel Modules

Linux kernel modules (`.ko` files) are compiled pieces of kernel code that can be loaded and unloaded from a running kernel without rebooting. This is how device drivers, filesystem drivers, network protocol implementations, and security frameworks like SELinux modules are managed. Every `docker run`, VPN connection, NFS mount, and `iptables` rule potentially loads a kernel module.

Understanding module management is critical in cloud environments: when you `modprobe overlay` to enable Docker's overlay filesystem, `modprobe br_netfilter` to enable Kubernetes networking, or `modprobe nbd` to mount EBS snapshots — you're dynamically loading kernel code. On Amazon Linux 2/2023 and Ubuntu AMIs, many modules are pre-loaded; on hardened or custom AMIs they may not be, causing silent failures when containerised workloads start.

### sysctl

`sysctl` provides a runtime interface to the kernel's parameter tree, exposed as files under `/proc/sys/`. Every parameter has a name (e.g., `net.ipv4.tcp_tw_reuse`), a current value, and semantics that affect kernel behaviour immediately. Unlike kernel module options, sysctl parameters can be changed without loading/unloading code — they're read by the running kernel on every relevant code path.

For AWS production environments, sysctl tuning is often the difference between a system that handles 50k concurrent connections and one that falls over at 1k. The default kernel parameters are tuned for safety and compatibility, not for high-throughput production workloads. Network buffer sizes, TCP stack behaviour, VM memory overcommit settings, and file descriptor limits are all controlled here.

---

## 💡 Real-World Use Cases

- Loading `br_netfilter` and `overlay` modules in a K8s node bootstrap script before running `kubeadm join`
- Tuning `net.core.somaxconn` and TCP buffer sizes on an EC2 instance running a high-traffic API gateway
- Debugging why Docker can't start by checking if the `overlay2` storage driver's required modules are loaded
- Preventing "CLOSE_WAIT connection accumulation" by enabling `net.ipv4.tcp_tw_reuse` on services making many outbound connections
- Hardening an EC2 instance by setting `kernel.dmesg_restrict=1` and `net.ipv4.conf.all.rp_filter=1`

---

## 🔧 Commands & Examples

### Kernel Module Management

```bash
# List all currently loaded modules
lsmod
# Output: Module name | Size | Use count | Used by

# Get detailed info about a module
modinfo overlay
modinfo br_netfilter
# Shows: filename, description, author, license, depends, vermagic, parameters

# Load a module
modprobe overlay
modprobe nf_conntrack
modprobe ip_vs           # needed for kube-proxy IPVS mode

# Load with parameters
modprobe nf_conntrack hashsize=65536

# Remove a module (if use count is 0)
modprobe -r dummy
rmmod dummy

# Force remove (dangerous — can crash if module is in use)
modprobe -rf nf_conntrack   # DON'T do this in production

# Load a module from a non-standard path
insmod /lib/modules/$(uname -r)/extra/mydriver.ko

# Check if a module is loaded
lsmod | grep overlay
# Or using /proc/modules:
cat /proc/modules | grep overlay

# List module dependencies
modprobe --show-depends overlay
# modprobe: br_netfilter requires nf_defrag_ipv6, nf_defrag_ipv4
```

### Module Persistence Across Reboots

```bash
# Make modules load at boot — add to /etc/modules-load.d/
cat > /etc/modules-load.d/k8s.conf << 'EOF'
overlay
br_netfilter
nf_conntrack
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
EOF

# Verify on next boot: systemd-modules-load.service processes this file
# Check if the service ran successfully
systemctl status systemd-modules-load.service
journalctl -u systemd-modules-load.service

# Pass parameters to modules at load time
cat > /etc/modprobe.d/conntrack.conf << 'EOF'
options nf_conntrack hashsize=131072
EOF

# Blacklist a module (prevent auto-loading)
cat >> /etc/modprobe.d/blacklist.conf << 'EOF'
blacklist nouveau
blacklist pcspkr
EOF

# Force blacklisted module to not even be loaded if required by other modules
echo "install nouveau /bin/false" >> /etc/modprobe.d/blacklist.conf

# List all available modules for current kernel
find /lib/modules/$(uname -r) -name "*.ko*" | wc -l
ls /lib/modules/$(uname -r)/kernel/
```

### Debugging Module Issues

```bash
# Check kernel ring buffer for module-related messages
dmesg | grep -i "module\|driver\|error" | tail -20

# Why won't a module load? Check dependencies
modprobe --verbose overlay
modprobe -v br_netfilter

# Check if a module failed with an error
dmesg | grep -E "FATAL|taint|oops" | tail -20

# See which modules are tainting the kernel
cat /proc/sys/kernel/tainted
# 0 = clean, non-zero = some issue (bit flags)
# https://www.kernel.org/doc/html/latest/admin-guide/tainted-kernels.html

# Check module parameters at runtime
cat /sys/module/nf_conntrack/parameters/hashsize
cat /sys/module/overlay/parameters/redirect_dir

# List all parameters for a module
ls /sys/module/nf_conntrack/parameters/
```

### sysctl Basics

```bash
# View all current sysctl settings
sysctl -a
sysctl -a | grep net.ipv4.tcp | head -20

# View a specific setting
sysctl net.core.somaxconn
sysctl vm.swappiness

# Set a value immediately (runtime, not persistent)
sysctl -w net.core.somaxconn=65536
sysctl -w vm.swappiness=10

# Equivalent direct file access
cat /proc/sys/net/core/somaxconn
echo 65536 > /proc/sys/net/core/somaxconn

# Make persistent: write to /etc/sysctl.d/
cat > /etc/sysctl.d/99-production.conf << 'EOF'
# Applied after all other sysctl.d files (99 = high priority)
net.core.somaxconn = 65536
net.ipv4.tcp_tw_reuse = 1
vm.swappiness = 10
EOF

# Apply all sysctl.d files immediately
sysctl --system

# Apply a specific file
sysctl -p /etc/sysctl.d/99-production.conf
```

### Network Stack Tuning (Production-Critical)

```bash
# --- TCP Connection Queue ---
# Max connections waiting to be accepted (listen backlog)
sysctl -w net.core.somaxconn=65536
# SYN backlog (half-open connections during 3-way handshake)
sysctl -w net.ipv4.tcp_max_syn_backlog=65536

# --- TCP Buffer Sizes ---
# Increase TCP receive/send buffer (allow 256MB buffers)
sysctl -w net.core.rmem_max=268435456
sysctl -w net.core.wmem_max=268435456
# TCP auto-tuning min/default/max (bytes)
sysctl -w net.ipv4.tcp_rmem="4096 87380 268435456"
sysctl -w net.ipv4.tcp_wmem="4096 65536 268435456"

# --- TIME_WAIT and Connection Reuse ---
# Allow reusing TIME_WAIT sockets for new connections
sysctl -w net.ipv4.tcp_tw_reuse=1
# Max number of connections in TIME_WAIT
sysctl -w net.ipv4.tcp_max_tw_buckets=2000000

# --- File Descriptor Limits ---
# System-wide max open files
sysctl -w fs.file-max=2097152
# Per-process limits (also edit /etc/security/limits.conf)
sysctl -w fs.nr_open=2097152

# --- Network Interface Queuing ---
# Increase receive queue size per CPU
sysctl -w net.core.netdev_max_backlog=65536

# --- Ephemeral Port Range ---
# Increase available ports for outgoing connections
sysctl -w net.ipv4.ip_local_port_range="1024 65535"

# --- TCP Keepalive ---
# Detect dead connections faster (good for load balancers)
sysctl -w net.ipv4.tcp_keepalive_time=60
sysctl -w net.ipv4.tcp_keepalive_intvl=10
sysctl -w net.ipv4.tcp_keepalive_probes=5

# --- Conntrack (for firewalls, NAT, K8s) ---
sysctl -w net.netfilter.nf_conntrack_max=1048576
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=86400
```

### VM / Memory Tuning

```bash
# --- Swappiness ---
# 0 = don't swap unless absolutely necessary (good for databases, K8s)
# 10 = very reluctant to swap (good for most production workloads)
# 60 = default
sysctl -w vm.swappiness=10

# --- OOM Killer ---
# Panic instead of killing processes on OOM (for critical systems)
sysctl -w vm.panic_on_oom=1
sysctl -w kernel.panic=30    # reboot after 30 seconds
# OR: just kill aggressively (for stateless workloads)
sysctl -w vm.overcommit_memory=1     # always allow allocation
sysctl -w vm.overcommit_ratio=50     # commit up to 50% beyond RAM

# --- Dirty Page Writeback (I/O tuning) ---
# When to start flushing dirty pages
sysctl -w vm.dirty_ratio=15          # % of RAM (hard threshold)
sysctl -w vm.dirty_background_ratio=5 # % of RAM (background flush starts)
# For databases with large write workloads, lower these:
sysctl -w vm.dirty_ratio=10
sysctl -w vm.dirty_background_ratio=3

# --- Transparent Huge Pages (THP) ---
# THP can cause latency spikes in databases (Redis, MongoDB, Postgres)
# Check current setting:
cat /sys/kernel/mm/transparent_hugepage/enabled
# Disable THP:
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
# Make persistent via rc.local or a systemd service

# --- NUMA ---
cat /proc/sys/kernel/numa_balancing
sysctl -w kernel.numa_balancing=0    # disable for latency-sensitive workloads
```

### Security Hardening via sysctl

```bash
cat > /etc/sysctl.d/99-security-hardening.conf << 'EOF'
# Restrict dmesg access to root
kernel.dmesg_restrict = 1

# Restrict kernel pointer leaks in /proc
kernel.kptr_restrict = 2

# Restrict ptrace to parent process only (prevents process injection)
kernel.yama.ptrace_scope = 1

# Disable SUID binaries executing with elevated ASLR randomization
kernel.randomize_va_space = 2

# --- Network Security ---
# Reverse path filtering (block spoofed packets)
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Ignore ICMP broadcasts (Smurf attack mitigation)
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Ignore bogus ICMP error responses
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Disable IP source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Disable ICMP redirects (MITM prevention)
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# Enable SYN cookies (SYN flood protection)
net.ipv4.tcp_syncookies = 1

# Log martian packets (packets with impossible source addresses)
net.ipv4.conf.all.log_martians = 1

# --- IPv6 security ---
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
EOF

sysctl --system
```

### Complete Production Config for K8s Nodes

```bash
cat > /etc/sysctl.d/99-k8s-node.conf << 'EOF'
# Required by K8s networking
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# Connection tracking for kube-proxy
net.netfilter.nf_conntrack_max = 1048576

# High-throughput networking
net.core.somaxconn = 65536
net.ipv4.tcp_max_syn_backlog = 65536
net.core.netdev_max_backlog = 65536
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_tw_reuse = 1

# Large buffer sizes for inter-pod communication
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728

# Inotify for Kubernetes watch mechanisms
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512

# File descriptors for pods
fs.file-max = 2097152
fs.nr_open = 2097152

# Disable swap (K8s requires this)
vm.swappiness = 0
EOF

sysctl --system

# Also load required modules before applying sysctl
modprobe br_netfilter
modprobe overlay
modprobe nf_conntrack
```

---

## ⚠️ Gotchas & Pro Tips

- **Module changes take effect immediately but sysctl.d needs `sysctl --system`:** Writing to `/etc/sysctl.d/` doesn't auto-apply. You must run `sysctl --system` (or `sysctl -p /path/to/file`) to load the new values. Include this in your bootstrap scripts after writing the config.

- **`br_netfilter` must be loaded BEFORE applying K8s sysctl settings:** The `net.bridge.bridge-nf-call-iptables` parameter only exists in `/proc/sys/` when the `br_netfilter` module is loaded. If you apply the sysctl before the module loads, it silently fails. Always `modprobe br_netfilter` first.

- **Transparent Huge Pages and database performance:** THP can cause 10-50x latency spikes in Redis, MongoDB, and Cassandra workloads. These databases explicitly document requiring THP to be disabled. In AWS, even on memory-optimised instances, always disable THP for database workloads.

- **`vm.swappiness=0` doesn't disable swap, it just makes it very unlikely:** The kernel will still swap under extreme memory pressure. To truly disable swap: `swapoff -a` and comment out swap entries in `/etc/fstab`. Kubernetes nodes require swap to be disabled (K8s will fail admission if swap is active).

- **sysctl settings have no effect inside containers:** Container namespaces don't get their own network sysctl values by default. K8s security contexts can grant `CAP_NET_ADMIN` to allow a container to set certain sysctls, but most settings must be set on the host node to affect the container's networking.

- **Check kernel version support:** Not all sysctl parameters exist in all kernel versions. Before deploying a sysctl config to a fleet, test it: `sysctl -p /etc/sysctl.d/99-myconfig.conf` will error on unknown parameters. Use `sysctl -e` (ignore errors) if you need a config that works across mixed kernel versions.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
