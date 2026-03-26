# 🐧 tcpdump & Packet Analysis — Linux Mastery

> **tcpdump is the most powerful network debugging tool available on any Linux system — no agent install, no GUI, just raw visibility into what's actually traversing the wire, available even when your application lies to you.**

## 📖 Concept

`tcpdump` is a command-line packet analyser that captures network traffic by putting a network interface into promiscuous mode and applying BPF (Berkeley Packet Filter) programs to selectively capture matching packets. It writes captures to the terminal in human-readable form or to `.pcap` files for later analysis with Wireshark.

Understanding tcpdump is fundamental for cloud engineers because AWS VPCs, security groups, NACLs, and load balancers all operate at the packet level. When an application timeout doesn't match any application-layer error, when TLS handshakes fail, when connection refused errors appear but the service is running, or when latency spikes are unexplained — tcpdump is the tool that shows ground truth. You can run it directly on EC2 instances, inside Docker containers, on K8s nodes, or capture traffic from VPC flow logs when direct access isn't possible.

BPF filters are compiled into a small bytecode program and run inside the kernel, meaning only matching packets ever reach user space. This keeps overhead minimal even on high-traffic production hosts. The filter syntax (`tcpdump -i eth0 'tcp port 443 and host 10.0.1.5'`) uses the same expressions as Wireshark display filters but with slightly different syntax — mastering it lets you zero in on exactly the traffic you care about without drowning in noise.

---

## 💡 Real-World Use Cases

- Debugging an intermittent "connection reset" error between a microservice and its database by capturing the TCP RST packet and identifying which side is resetting
- Confirming that traffic is actually reaching an EC2 instance despite security group rules appearing correct
- Capturing TLS handshake failures to determine if they're client-side or server-side certificate issues
- Analysing DNS resolution behaviour — confirming which resolver is being used and whether queries are being answered correctly
- Verifying that a load balancer is forwarding the `X-Forwarded-For` header and confirming client IPs in HTTP requests

---

## 🔧 Commands & Examples

### Basic Capture Syntax

```bash
# Capture on a specific interface (list interfaces with -D)
tcpdump -i eth0
tcpdump -D                      # list available interfaces

# Capture on all interfaces
tcpdump -i any

# Don't resolve hostnames or port names (faster, clearer in scripts)
tcpdump -n       # no hostname resolution
tcpdump -nn      # no hostname AND no port name resolution (recommended for prod)

# Verbose output
tcpdump -v       # verbose
tcpdump -vv      # more verbose (shows TCP flags, TTL, etc.)
tcpdump -vvv     # maximum verbosity

# Limit capture to N packets
tcpdump -c 100 -i eth0

# Show packet contents in hex and ASCII
tcpdump -X -i eth0
tcpdump -XX -i eth0    # includes ethernet header

# Show ASCII only (great for HTTP debugging)
tcpdump -A -i eth0
```

### Saving and Reading Captures

```bash
# Write to a .pcap file (for Wireshark analysis)
tcpdump -i eth0 -w /tmp/capture.pcap

# Write with rotation (100MB per file, keep last 10)
tcpdump -i eth0 -C 100 -W 10 -w /tmp/capture.pcap

# Read from a .pcap file
tcpdump -r /tmp/capture.pcap
tcpdump -r /tmp/capture.pcap -nn

# Capture to pcap with a filter, then read back with different filter
# (Wireshark can also read these pcap files)
tcpdump -i eth0 -w /tmp/http.pcap 'tcp port 80'
tcpdump -r /tmp/http.pcap -A | grep 'Host:'

# Timestamped captures for latency analysis
tcpdump -i eth0 -ttt -nn 'tcp port 5432'
# -ttt: time delta between packets (great for spotting slow queries)
# -tttt: absolute timestamps
```

### BPF Filter Expressions

```bash
# Filter by host (src or dst)
tcpdump -nn -i eth0 host 10.0.1.100

# Filter by source or destination specifically
tcpdump -nn -i eth0 src host 10.0.1.100
tcpdump -nn -i eth0 dst host 10.0.1.100

# Filter by port
tcpdump -nn -i eth0 port 443
tcpdump -nn -i eth0 port 80 or port 443

# Filter by port range
tcpdump -nn -i eth0 portrange 8080-8090

# Filter by protocol
tcpdump -nn -i eth0 tcp
tcpdump -nn -i eth0 udp
tcpdump -nn -i eth0 icmp

# Combine filters with and/or/not
tcpdump -nn -i eth0 'tcp and port 443 and host 10.0.1.5'
tcpdump -nn -i eth0 'not port 22'                          # exclude SSH noise
tcpdump -nn -i eth0 'tcp port 80 and not host 10.0.0.1'

# Filter by network/subnet
tcpdump -nn -i eth0 net 10.0.1.0/24
tcpdump -nn -i eth0 src net 192.168.0.0/16

# Filter by packet size
tcpdump -nn -i eth0 'len > 1400'    # jumbo frames or large payloads
tcpdump -nn -i eth0 'len < 100'     # tiny packets (possible keepalives or SYNs)

# Filter TCP flags
# SYN packets (new connections)
tcpdump -nn -i eth0 'tcp[tcpflags] & tcp-syn != 0'
# RST packets (connection resets — the interesting ones)
tcpdump -nn -i eth0 'tcp[tcpflags] & tcp-rst != 0'
# FIN packets (clean closes)
tcpdump -nn -i eth0 'tcp[tcpflags] & tcp-fin != 0'
# SYN-ACK (connection accepted)
tcpdump -nn -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-ack) == (tcp-syn|tcp-ack)'

# Capture only connection setup (SYN, SYN-ACK, ACK)
tcpdump -nn -i eth0 'tcp[13] & 0x02 != 0'    # SYN bit set
```

### Protocol-Specific Debugging

```bash
# --- HTTP Debugging ---

# Capture HTTP request headers
tcpdump -i eth0 -A -nn 'tcp port 80' | grep -A 5 'GET\|POST\|PUT\|DELETE\|Host:'

# Full HTTP body capture (be careful with sensitive data)
tcpdump -i eth0 -s 0 -A -nn 'tcp port 80'
# -s 0: capture full packet (default is 262144 bytes, old default was 68)

# Capture HTTP and HTTPS traffic on non-standard ports
tcpdump -i eth0 -nn 'tcp port 8080 or tcp port 8443'

# --- DNS Debugging ---

# Watch all DNS queries and responses
tcpdump -i eth0 -nn 'udp port 53'

# Watch DNS with full content
tcpdump -i eth0 -nn -vv 'udp port 53'

# Capture DNS queries only (not responses)
tcpdump -i eth0 -nn 'udp port 53 and udp[10] & 0x80 = 0'

# Watch for specific DNS query
tcpdump -i eth0 -nn -A 'udp port 53' | grep -A 2 'api.example.com'

# --- Database Traffic ---

# PostgreSQL connections and queries (port 5432)
tcpdump -i eth0 -nn -A 'tcp port 5432' | grep -E 'SELECT|INSERT|UPDATE|DELETE'

# MySQL (port 3306)
tcpdump -i eth0 -nn -A 'tcp port 3306'

# Redis (port 6379)
tcpdump -i eth0 -nn -A 'tcp port 6379'

# --- ICMP / Ping ---
tcpdump -i eth0 -nn icmp
tcpdump -i eth0 -nn 'icmp and host 10.0.1.5'
```

### TCP Connection Lifecycle Analysis

```bash
# Watch the full 3-way handshake and teardown for a connection
# Terminal 1: start capture
tcpdump -i eth0 -nn -S 'tcp and host 10.0.1.100 and port 5432'
# -S: show absolute sequence numbers (not relative) — easier to match packets

# Terminal 2: make a connection
curl http://10.0.1.100:5000/health

# Expected output shows:
# SYN:     client:high_port > server:5432 Flags [S]
# SYN-ACK: server:5432 > client:high_port Flags [S.]
# ACK:     client:high_port > server:5432 Flags [.]
# ... data exchange ...
# FIN:     one side sends [F.]
# FIN-ACK: other side [F.]
# ACK:     [.]  (connection closed)

# Debugging RST (connection reset)
tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-rst != 0' -v
# RST from server = server refused or closed connection
# RST from client = client gave up waiting

# TIME_WAIT analysis — connections stuck in TIME_WAIT
ss -nt state time-wait | head -20
# And see if RSTs are clearing them:
tcpdump -i eth0 -nn 'tcp[tcpflags] & tcp-rst != 0 and port 80' -c 50
```

### Capturing Inside Docker / Kubernetes

```bash
# Find the veth interface for a container
# Get container PID
CONTAINER_ID="abc123"
PID=$(docker inspect -f '{{.State.Pid}}' $CONTAINER_ID)

# Find the veth pair
ip link show | grep -A 1 "ifindex $(cat /proc/$PID/net/if_inet6 | awk '{print $7}' | head -1)"

# Capture directly on the container's network namespace
nsenter -t $PID -n tcpdump -i eth0 -nn 'tcp port 8080'

# On Kubernetes: capture traffic from a specific pod
NODE_NAME=$(kubectl get pod mypod -o jsonpath='{.spec.nodeName}')
# SSH to the node, then:
CONTAINER_ID=$(crictl ps | grep mypod | awk '{print $1}')
PID=$(crictl inspect $CONTAINER_ID | jq '.info.pid')
nsenter -t $PID -n tcpdump -i eth0 -nn -w /tmp/pod-capture.pcap

# Or use kubectl debug (ephemeral container) for newer K8s:
kubectl debug -it mypod --image=nicolaka/netshoot --target=mycontainer
# Inside the netshoot container:
tcpdump -i eth0 -nn 'tcp port 8080'

# Capture traffic on a K8s node for a specific pod IP
POD_IP=$(kubectl get pod mypod -o jsonpath='{.status.podIP}')
# On the node:
tcpdump -i any -nn "host $POD_IP" -w /tmp/pod.pcap
```

### Production-Safe Capture Patterns

```bash
# Capture 60 seconds of traffic to a file, then stop
timeout 60 tcpdump -i eth0 -nn -w /tmp/prod-capture.pcap 'tcp port 443'

# Capture with size limit (stop after 50MB)
tcpdump -i eth0 -nn -w /tmp/capture.pcap -C 50 -W 1 'tcp port 8080'

# Ring buffer capture: 10 files of 100MB each (keeps last ~1GB)
tcpdump -i eth0 -nn -w /tmp/ring-capture.pcap -C 100 -W 10

# Low-overhead capture: only capture packet headers (no payload)
tcpdump -i eth0 -nn -s 96 -w /tmp/headers-only.pcap

# Capture and compress in real-time (for long-running captures)
tcpdump -i eth0 -nn -w - 'tcp port 443' | gzip > /tmp/capture.pcap.gz

# Safe: run as unprivileged user with capabilities
# Instead of running as root, grant cap_net_raw
setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump
# Or use group dumpcap (Wireshark approach)
```

### Analysing Captured Files

```bash
# Read and filter a pcap file
tcpdump -r /tmp/capture.pcap -nn 'tcp port 443'

# Count connections by source IP
tcpdump -r /tmp/capture.pcap -nn 'tcp[tcpflags] & tcp-syn != 0' | \
  awk '{print $3}' | cut -d. -f1-4 | sort | uniq -c | sort -rn | head -20

# Extract HTTP Host headers from a capture
tcpdump -r /tmp/capture.pcap -A -nn 'tcp port 80' | grep 'Host:' | sort | uniq -c

# Find all RST events in a capture with timestamps
tcpdump -r /tmp/capture.pcap -nn -tttt 'tcp[tcpflags] & tcp-rst != 0'

# Measure inter-packet timing (latency analysis)
tcpdump -r /tmp/capture.pcap -nn -ttt 'tcp and host 10.0.1.5 and port 5432' | \
  awk '{print $1, $3, $4}' | head -50
```

---

## ⚠️ Gotchas & Pro Tips

- **Always use `-nn` in production:** Hostname and port resolution (`-n` off) causes DNS lookups for every captured packet address. On a busy server this creates DNS noise and slows down tcpdump itself. Always use `-nn` unless you specifically want resolution.

- **`-s 0` for full packet capture:** The default snap length is 262144 bytes (plenty for headers), but for application payload capture (HTTP bodies, database queries) always verify with `-s 0` or a specific snap length.

- **Encrypted traffic is mostly opaque:** TLS/SSL traffic shows the handshake (hello, certificates, key exchange) but not the application data. For HTTPS debugging you need to either: (a) capture the pre-master secret using `SSLKEYLOGFILE`, (b) use a MITM proxy, or (c) debug at the application layer with `strace` or language-specific tools.

- **Ring buffer for always-on capture:** In production, run tcpdump with `-C 100 -W 10` (rotate files at 100MB, keep 10 files) as a background service. When an incident occurs, the evidence is already captured. Without this, you're always debugging after the fact.

- **`tcpdump` on AWS EC2 won't show traffic blocked by security groups:** Security groups are enforced in the hypervisor, before the packet reaches the EC2 instance's NIC. If a packet is blocked by a security group, `tcpdump` on the EC2 instance will never see it. Use VPC Flow Logs for that level of visibility.

- **AWS VPC mirroring for production visibility:** For sensitive production traffic you can't capture directly, use VPC Traffic Mirroring to send a copy of ENI traffic to a monitoring appliance running tcpdump/Wireshark without touching the production host.

- **Combine with `tshark` for scripted analysis:** `tshark` (terminal Wireshark) uses the same capture engine but with richer output formatting. For automated pcap analysis in scripts, `tshark -T fields -e frame.time -e ip.src -e ip.dst -e tcp.flags` gives clean structured output.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
