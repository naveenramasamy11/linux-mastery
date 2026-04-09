# 🐧 TCP & Network Performance Tuning — Linux Mastery

> **Default Linux TCP settings are tuned for 1990s hardware — on modern 10/25/100GbE NICs with millisecond RTT to your load balancer or RDS instance, a few sysctl changes can double your throughput and cut tail latency by 50%.**

## 📖 Concept

TCP performance is governed by a feedback loop: the sender maintains a **congestion window (cwnd)** and a **receive window (rwnd)**, both measured in bytes. The effective throughput is `min(cwnd, rwnd) / RTT`. If your RTT to an RDS instance is 1ms and your receive buffer is only 128KB, your maximum throughput is `128KB / 0.001s = 128MB/s` regardless of NIC speed. Larger buffers allow larger windows, which enables higher throughput on high-latency paths.

The Linux kernel manages TCP buffers at three levels: **socket send buffer** (`/proc/sys/net/ipv4/tcp_wmem`), **socket receive buffer** (`/proc/sys/net/ipv4/tcp_rmem`), and the **global socket memory limit** (`/proc/sys/net/core/rmem_max`). The `tcp_mem` sysctl controls when the kernel starts applying memory pressure globally.

**TCP congestion control algorithm** choice is critical. The default `cubic` algorithm is designed for high-bandwidth, high-latency paths (cross-continental). For intra-datacenter and cloud traffic (low RTT, predictable loss), **BBR** (Bottleneck Bandwidth and RTT, from Google) dramatically improves throughput by modeling bandwidth and RTT instead of reacting to packet loss. BBR is the right choice for most AWS/GCP/Azure workloads.

**`tcp_fastopen`** eliminates the TCP handshake latency for repeat connections by sending data in the SYN packet. **`tcp_tw_reuse`** allows reuse of TIME_WAIT sockets, preventing port exhaustion on busy API servers. **`SO_REUSEPORT`** enables multiple processes/threads to bind to the same port for kernel-level load balancing.

---

## 💡 Real-World Use Cases

- **EC2 instance to RDS:** Tuning `tcp_rmem`/`tcp_wmem` and enabling BBR on a `c5.4xlarge` running a connection-heavy application reduced p99 query latency by 35% by eliminating buffer stalls.
- **NLB/ALB high-connection-rate:** `net.ipv4.tcp_tw_reuse=1` + `ip_local_port_range` expansion prevented port exhaustion on an API gateway handling 100k req/s.
- **Inter-AZ data replication:** Enabling BBR for cross-AZ replication streams between Kafka brokers increased throughput 40% by better utilizing available bandwidth during transient congestion.
- **K8s pod-to-pod on 25GbE:** Default qdisc (`pfifo_fast`) replaced with `fq` (Fair Queue) + BBR for better per-flow fairness and reduced bufferbloat.
- **EFA (Elastic Fabric Adapter) tuning:** Custom `tcp_mem` and interrupt coalescing settings for HPC workloads on `hpc6a` instances.

---

## 🔧 Commands & Examples

### Check Current TCP Settings

```bash
# Current TCP buffer settings
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem net.core.rmem_max net.core.wmem_max
# net.ipv4.tcp_rmem = 4096	87380	6291456    (min default max in bytes)
# net.ipv4.tcp_wmem = 4096	16384	4194304
# net.core.rmem_max = 212992
# net.core.wmem_max = 212992

# Current congestion control
sysctl net.ipv4.tcp_congestion_control
# net.ipv4.tcp_congestion_control = cubic

# Available congestion control algorithms
sysctl net.ipv4.tcp_available_congestion_control
# net.ipv4.tcp_available_congestion_control = reno cubic bbr

# Current connection tracking (NAT/conntrack)
sysctl net.netfilter.nf_conntrack_max
sysctl net.netfilter.nf_conntrack_count   # Current usage

# Socket backlog
sysctl net.core.somaxconn        # Max listen() backlog
sysctl net.ipv4.tcp_max_syn_backlog  # SYN queue depth

# TIME_WAIT socket count
ss -s | grep "TIME-WAIT"
ss --tcp state time-wait | wc -l
```

### TCP Buffer Tuning

```bash
# Formula: optimal buffer = bandwidth (bytes/sec) × RTT (sec)
# Example: 10Gbps link, 1ms RTT to RDS in same VPC:
# 10Gbps = 1,250,000,000 bytes/sec × 0.001 = 1,250,000 bytes ≈ 1.2MB per connection
# For 1,000 concurrent connections: 1.2GB total socket buffer budget

# Production TCP buffer settings for AWS (tune per workload)
cat >> /etc/sysctl.d/99-tcp-tuning.conf << 'EOF'
# TCP receive buffer: min=4K, default=256K, max=16M
net.ipv4.tcp_rmem = 4096 262144 16777216

# TCP send buffer: min=4K, default=256K, max=16M
net.ipv4.tcp_wmem = 4096 262144 16777216

# Allow kernel to auto-tune receive buffers
net.ipv4.tcp_moderate_rcvbuf = 1

# Global max socket buffer size (must be >= tcp_rmem/wmem max)
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.rmem_default = 262144
net.core.wmem_default = 262144

# Total TCP memory (in pages): min pressure max
# Default is auto-calculated based on RAM
# For 16GB RAM: ~1.5M pages budget
net.ipv4.tcp_mem = 196608 524288 1572864

# Increase network device backlog queue
net.core.netdev_max_backlog = 16384

# Socket listen backlog (for high-connection-rate servers)
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
EOF

# Apply immediately
sysctl -p /etc/sysctl.d/99-tcp-tuning.conf

# Verify
sysctl net.ipv4.tcp_rmem net.core.rmem_max
```

### BBR Congestion Control

```bash
# Check if BBR module is available
modinfo tcp_bbr 2>/dev/null && echo "BBR available" || echo "BBR not available"

# Load BBR module
modprobe tcp_bbr
echo "tcp_bbr" >> /etc/modules-load.d/bbr.conf  # Load at boot

# Enable BBR + FQ qdisc (required for BBR to work correctly)
cat >> /etc/sysctl.d/99-tcp-tuning.conf << 'EOF'
net.ipv4.tcp_congestion_control = bbr
net.core.default_qdisc = fq
EOF

sysctl -p /etc/sysctl.d/99-tcp-tuning.conf

# Verify BBR is active
sysctl net.ipv4.tcp_congestion_control
# net.ipv4.tcp_congestion_control = bbr

# Check qdisc on each interface
tc qdisc show dev eth0
# qdisc fq 8001: root refcnt 2 limit 10000p flow_limit 100p ...

# Set FQ on all interfaces
for iface in $(ip -o link show | awk -F': ' '{print $2}' | grep -v lo); do
    tc qdisc replace dev $iface root fq
done

# Verify existing connections are using BBR
ss -tin | grep bbr
# cubic is shown per-socket; new connections will use bbr
```

### Port Exhaustion and TIME_WAIT Tuning

```bash
# Check ephemeral port range
sysctl net.ipv4.ip_local_port_range
# net.ipv4.ip_local_port_range = 32768	60999 ← only ~28K ports!

# Expand ephemeral port range (max usable: 1024-65535)
cat >> /etc/sysctl.d/99-tcp-tuning.conf << 'EOF'
# Expand ephemeral port range from ~28K to ~64K ports
net.ipv4.ip_local_port_range = 1024 65535

# Allow TIME_WAIT socket reuse for outbound connections
# Safe for client-side (outbound to backends, not server listen sockets)
net.ipv4.tcp_tw_reuse = 1

# Reduce TIME_WAIT duration (default 60s — this halves FIN-WAIT-2)
# WARNING: Don't lower tcp_fin_timeout below 15s in production
net.ipv4.tcp_fin_timeout = 15

# Max TIME_WAIT sockets before kernel starts killing them
net.ipv4.tcp_max_tw_buckets = 1440000
EOF

sysctl -p /etc/sysctl.d/99-tcp-tuning.conf

# Monitor TIME_WAIT count in real-time
watch -n2 'ss -s | grep TIME-WAIT; ss --tcp state time-wait | wc -l'
```

### TCP Keepalive Tuning

```bash
# Default keepalive: 2h idle, 75s interval, 9 probes = 2h + 11min before detecting dead
# For cloud environments, connections to NAT gateways/ELBs time out after 350s idle

cat >> /etc/sysctl.d/99-tcp-tuning.conf << 'EOF'
# Start keepalive probes after 60s idle (protects against NAT timeout)
net.ipv4.tcp_keepalive_time = 60

# Send probe every 10s
net.ipv4.tcp_keepalive_intvl = 10

# After 6 failed probes (60s total), declare connection dead
net.ipv4.tcp_keepalive_probes = 6
EOF

sysctl -p /etc/sysctl.d/99-tcp-tuning.conf

# Per-socket keepalive (Python example for application-level control)
# In Python:
# import socket
# sock.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
# sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, 60)
# sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPINTVL, 10)
# sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPCNT, 6)
```

### NIC Interrupt Coalescing and Ring Buffer Tuning

```bash
# Check current ring buffer size
ethtool -g eth0
# Ring parameters for eth0:
# Pre-set maximums:
# RX:             4096
# TX:             4096
# Current hardware settings:
# RX:             256      ← too small for 10GbE!
# TX:             256

# Increase ring buffers (reduces dropped packets under burst traffic)
ethtool -G eth0 rx 4096 tx 4096

# Persist ring buffer changes via NetworkManager dispatcher or udev rule
cat > /etc/udev/rules.d/99-nic-ring.rules << 'EOF'
ACTION=="add", SUBSYSTEM=="net", KERNEL=="eth*", \
    RUN+="/usr/sbin/ethtool -G %k rx 4096 tx 4096"
EOF

# Check interrupt coalescing (balance between latency and CPU overhead)
ethtool -c eth0
# Coalesce parameters for eth0:
# rx-usecs: 3         ← interrupt after 3µs of inactivity (low latency mode)
# rx-frames: 0
# tx-usecs: 3
# tx-frames: 0

# For throughput-optimized (high-bandwidth batch workloads)
ethtool -C eth0 rx-usecs 50 tx-usecs 50 rx-frames 64 tx-frames 64

# For latency-optimized (real-time, low-latency APIs)
ethtool -C eth0 rx-usecs 0 tx-usecs 0   # Adaptive disabled, interrupt on every packet

# Multi-queue NIC: spread IRQs across CPUs
# Check current IRQ affinity
cat /proc/interrupts | grep eth0

# Automatic IRQ balancing
systemctl enable --now irqbalance

# Manual IRQ pinning (for NUMA-aware tuning)
# Pin eth0 queue 0 IRQ to CPU 0
echo 1 > /proc/irq/$(grep eth0-TxRx-0 /proc/interrupts | awk -F: '{print $1}' | tr -d ' ')/smp_affinity
```

### Benchmarking Network Performance

```bash
# iperf3 — bandwidth test (run server on target, client on source)
# Server:
iperf3 -s -p 5201

# Client: single stream
iperf3 -c 10.0.1.100 -p 5201 -t 30

# Client: parallel streams (better for multi-queue NICs)
iperf3 -c 10.0.1.100 -P 8 -t 30

# UDP test (test packet loss and jitter)
iperf3 -c 10.0.1.100 -u -b 1G -t 30

# nuttcp — alternative with more options
# netperf — latency-focused benchmarks (RR test = request/response latency)
netperf -H 10.0.1.100 -t TCP_RR -l 30 -- -r 64,64
# Measures transactions per second (inverse of RTT)

# ss — real-time socket stats during benchmark
watch -n1 'ss -tin dst 10.0.1.100 | grep -A1 10.0.1.100 | grep cwnd'
# tcpi_snd_cwnd:87 rcv_space:14480 ...
# cwnd growing = BBR/cubic ramping up as expected

# Verify BBR is working during a transfer
ss -tni | grep -E "bbr|cubic|reno"
# Should show bbr for new connections
```

### conntrack Tuning (for NAT/iptables heavy systems)

```bash
# Check conntrack usage
sysctl net.netfilter.nf_conntrack_max
sysctl net.netfilter.nf_conntrack_count  # Current connections tracked

# Check for conntrack table full (this drops packets silently!)
dmesg | grep "nf_conntrack: table full"
journalctl -k | grep "nf_conntrack: table full"

# Increase conntrack table size
cat >> /etc/sysctl.d/99-tcp-tuning.conf << 'EOF'
# conntrack table: memory = nf_conntrack_max × ~320 bytes
# 2M entries ≈ 640MB RAM — tune based on available memory
net.netfilter.nf_conntrack_max = 2097152
net.netfilter.nf_conntrack_tcp_timeout_established = 300   # Default 432000s = 5 days!
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 15
EOF

# Increase conntrack hash table size (set at module load time)
echo "options nf_conntrack hashsize=524288" > /etc/modprobe.d/nf_conntrack.conf
# Takes effect after module reload

# Monitor conntrack in real-time
watch -n2 'cat /proc/sys/net/netfilter/nf_conntrack_count; \
           conntrack -L 2>/dev/null | wc -l'
```

---

## ⚠️ Gotchas & Pro Tips

- **`tcp_tw_recycle` was REMOVED in Linux 4.12.** Many old tuning guides still recommend it. It caused connection drops behind NAT and was dangerous. Use `tcp_tw_reuse` instead (safe for outbound connections only).
- **BBR requires `fq` qdisc to work correctly.** Without Fair Queue (`net.core.default_qdisc = fq`), BBR may underperform. The FQ qdisc provides per-flow pacing that BBR relies on.
- **`rmem_max` must be ≥ `tcp_rmem` max.** Setting `tcp_rmem = 4096 262144 16777216` without also setting `net.core.rmem_max = 16777216` limits actual buffer allocation. Both must be set.
- **conntrack `nf_conntrack_tcp_timeout_established` default is 5 days.** On busy API servers or K8s nodes, this causes the conntrack table to fill with long-dead connections. Drop it to 300-3600s based on your application's connection lifecycle.
- **Ring buffer increases trade memory for throughput.** Each entry in the RX ring buffer holds a full-size frame (1500+ bytes). 4096 × 1500B = ~6MB per NIC per direction. On servers with many NICs, budget accordingly.
- **Pro tip — verify actual throughput with BPF:**

```bash
# Use bpftrace to measure TCP retransmit rate in real-time
bpftrace -e 'kprobe:tcp_retransmit_skb { @[comm] = count(); } interval:s:5 { print(@); clear(@); }'

# Measure actual TCP receive buffer usage
bpftrace -e 'kprobe:tcp_rcv_established { @bytes = hist(((struct sock*)arg0)->sk_rcvbuf); } interval:s:10 { print(@); exit(); }'
```

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
