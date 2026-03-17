# 🐧 Socket Diagnostics with ss — Linux Mastery

> **Master ss for production socket diagnostics—the modern replacement for netstat that every SRE needs at 3 AM.**

## 📖 Concept

`ss` (socket statistics) is the modern replacement for `netstat`. It's faster, provides more detailed information, and is essential for production troubleshooting. When a microservice is slow, connections are timing out, or you suspect network saturation, `ss` is your first diagnostic tool. It hooks directly into the kernel's socket layer, avoiding userspace overhead.

Understanding socket states is foundational. Every TCP connection goes through predictable states: LISTEN, SYN_SENT, ESTABLISHED, TIME_WAIT, CLOSE_WAIT, and others. Each state tells a story: TIME_WAIT accumulation suggests clients closing connections properly but not reopening (maybe a connection pool leak); CLOSE_WAIT means the server closed but the client hasn't (likely a resource leak in client code). In production, socket state anomalies often surface problems before they become outages.

For DevOps and SRE engineers, `ss` answers critical questions: Which process owns port 8080? Why are there thousands of TIME_WAIT connections? Are we hitting the file descriptor limit? Is the kernel dropping packets? These answers come from mastering `ss` and understanding the data it exposes.

---

## 💡 Real-World Use Cases

- **Port ownership during troubleshooting**: Find which process is listening on port 443 or 8080, especially when it's not what you expected
- **Connection leak diagnosis**: Detect accumulation of CLOSE_WAIT or TIME_WAIT connections indicating resource leaks in clients or servers
- **Network saturation analysis**: Identify established connections, pending connections, and dropped packets
- **Kubernetes pod networking**: Debug network policies, service mesh connectivity, and inter-pod communication
- **Microservice discovery**: Map which services are connecting to each other and on which ports
- **Load testing and performance**: Monitor connection counts during test runs to validate scaling behavior
- **Security incident response**: Identify unexpected outbound connections or listening ports suggesting compromise

---

## 🔧 Commands & Examples

### Basic ss Commands

```bash
# Show all sockets (TCP, UDP, UNIX)
ss -a

# Show only listening sockets (common starting point)
ss -l

# Show only TCP sockets
ss -t

# Show only UDP sockets
ss -u

# Show only UNIX domain sockets
ss -x

# Show listening and established TCP connections
ss -tn

# Show connections with process information (-p requires root)
sudo ss -tlnp

# Show extended information (buffer sizes, RTT, etc.)
ss -e

# Combine flags: listening TCP with process info and extended details
sudo ss -tlnep
```

### Understanding ss Output

```bash
# Example output:
# sudo ss -tlnp
# State  Recv-Q Send-Q  Local:Port  Peer:Port  Process
# LISTEN 0      128     0.0.0.0:22  0.0.0.0:*  users:(("sshd",pid=1234))

# State: Connection state (LISTEN, ESTABLISHED, CLOSE_WAIT, TIME_WAIT, etc.)
# Recv-Q: Data queued for the application (should be 0 for LISTEN)
# Send-Q: Data in flight, not yet acked (should be 0 for LISTEN)
# Local:Port: What address/port we're listening on or connecting from
# Peer:Port: Remote address/port (0.0.0.0:* for listening sockets)
# Process: Which process owns this socket

# For established connections:
# ss -tn | grep ESTABLISHED
# ESTAB  0  0    10.0.1.5:42856  10.0.1.10:3306
# Recv-Q=0, Send-Q=0: connection is healthy, data flowing smoothly
```

### Filtering Connections by State

```bash
# Count connections by state
ss -tan | tail -n +2 | awk '{print $1}' | sort | uniq -c

# Show all TIME_WAIT connections (clients that closed but waiting)
ss -tan 'state TIME-WAIT'

# Show all CLOSE_WAIT connections (server closed, client hasn't)
ss -tan 'state CLOSE-WAIT'

# Show established connections only
ss -tan 'state ESTABLISHED'

# Show connections by destination port
ss -tan 'dport = :3306'    # MySQL connections
ss -tan 'dport = :5432'    # PostgreSQL connections
ss -tan 'dport = :6379'    # Redis connections

# Show connections from specific source IP
ss -tan 'src 10.0.0.5'

# Show connections on specific local port
ss -tan 'sport = :8080'
```

### Finding Which Process Owns a Port

```bash
# Critical: find the PID listening on port 8080
sudo ss -tlnp 'sport = :8080'

# Output shows the pid and command that owns it
# LISTEN 0 128 0.0.0.0:8080 0.0.0.0:* users:(("java",pid=5678))

# Without sudo, -p shows limited info, but combined with grep:
ss -tlnp 2>/dev/null | grep :8080

# Kill the process
sudo kill -9 5678

# Or use pkill/killall directly
sudo pkill -f 'java.*8080'
```

### Advanced Filtering: TCP Socket States Explained

```bash
# TCP state machine:
# LISTEN:      Waiting for incoming connections
# SYN_SENT:    Sent SYN, waiting for SYN-ACK
# SYN_RECV:    Received SYN, sent SYN-ACK
# ESTABLISHED: Connection open, data flowing
# FIN_WAIT1:   Sent FIN, waiting for ACK
# FIN_WAIT2:   Received ACK of FIN, waiting for peer's FIN
# TIME_WAIT:   Closed, waiting for any delayed packets
# CLOSE_WAIT:  Received FIN, waiting for app to close
# LAST_ACK:    Sent FIN, waiting for ACK

# Check for problematic states:
ss -tan 'state CLOSE-WAIT' | wc -l     # High count = server not closing connections
ss -tan 'state TIME-WAIT' | wc -l       # High count = normal, but > 100k is a problem

# SYN flood attack detection
ss -tan 'state SYN-RECV' | wc -l        # Should be < 100 normally
ss -tan 'state SYN-SENT' | wc -l        # Usually 0 unless making outbound connections
```

### Monitoring Connection Queues

```bash
# Recv-Q and Send-Q tell the whole story
ss -e | head -20

# High Recv-Q = application isn't reading data fast enough
#   (application bottleneck, may need more processing power)
# High Send-Q = kernel can't send data fast enough
#   (network saturation or socket buffer limits)

# Example: stuck connection
# ESTAB  100 0  10.0.1.5:8080  10.0.1.10:40000
#        ^^^ This means 100 bytes sitting unread
#            Application is slow or blocked

# Monitor in real-time
watch -n 1 'ss -e | grep -E "Send-Q|Recv-Q" | head'

# Alert on high queue depths
ss -e | awk '{if ($2 > 50 || $3 > 50) print "HIGH QUEUE: " $0}'
```

### Troubleshooting Network Problems

```bash
# Is the service listening?
ss -tlnp 'sport = :3306'    # MySQL

# How many established connections does it have?
ss -tn 'dport = :3306' | grep ESTABLISHED | wc -l

# Are connections stuck waiting for data?
sudo ss -tep | grep mysql

# TCP options and RTT (round-trip time)
sudo ss -tein 'dport = :8080'   # e=extended, i=TCP info
# Shows RTT estimates, window sizes, etc.

# Is kernel dropping packets?
ss -s      # Summary statistics
# Outputs: TCP: 1234 established, 567 closed, 100 orphaned
# If "TCP bad segments" is high, network is unreliable or under attack

# Connection counts by port
ss -tn | awk '{print $4}' | cut -d: -f2 | sort | uniq -c | sort -rn
```

### Comparing ss vs lsof vs netstat

```bash
# ss: Fast, kernel-level, detailed
sudo ss -tlnp

# lsof: Userspace, more information, slower
sudo lsof -i -n -P

# netstat: Deprecated, slow, avoid in production
sudo netstat -tlnp    # Don't use this

# For SRE: always use ss

# ss vs lsof for specific use case:
# "Which process is listening on port 443?"
sudo ss -tlnp 'sport = :443'     # Instant
sudo lsof -i :443                # Works but slower

# "What files did my process open?"
sudo lsof -p 5678     # lsof wins here (ss doesn't show files)
```

### Production Monitoring Script

```bash
#!/bin/bash
# Monitor socket health every minute

LOG_FILE="/var/log/socket-monitor.log"

{
    echo "=== $(date) ==="
    
    # Connection summary
    echo "Connection counts:"
    ss -tan | tail -n +2 | awk '{print $1}' | sort | uniq -c
    
    # High queue depths
    echo "High Recv-Q or Send-Q:"
    ss -tne | awk '$2 > 100 || $3 > 100 { print $0 }'
    
    # CLOSE_WAIT accumulation
    echo "CLOSE_WAIT connections:"
    ss -tan 'state CLOSE-WAIT' | wc -l
    
    # TIME_WAIT warning
    TIME_WAIT_COUNT=$(ss -tan 'state TIME-WAIT' | wc -l)
    if [ $TIME_WAIT_COUNT -gt 10000 ]; then
        echo "WARNING: High TIME_WAIT count: $TIME_WAIT_COUNT"
    fi
    
    # UDP statistics
    echo "UDP sockets:"
    ss -uan | wc -l
    
} >> "$LOG_FILE"
```

### Finding and Killing Stuck Connections

```bash
# Kill all TIME_WAIT connections (safe, kernel will handle)
# Note: These are already closed; killing just frees resources
ss -K state TIME-WAIT    # Requires ss with TCP_DIAG

# Or manually:
ss -tan 'state TIME-WAIT' | awk 'NR>1 {print $4}' | while read addr; do
    port=${addr##*:}
    echo "Would kill connections on port $port"
done

# Kill connections to specific remote host
ss -K 'dst 10.0.0.10'

# Kill all connections on port 3306 (MySQL)
sudo pkill -f 'port 3306'    # Less precise
sudo ss -K 'sport = :3306'    # More precise with ss
```

### Kubernetes Network Debugging

```bash
# Check pod network connectivity from node
kubectl debug node/NODE_NAME -it --image=ubuntu:22.04

# Inside the debug pod, check what's connecting to service
ss -tne | grep SERVICE_CLUSTER_IP

# Check if NodePort is receiving connections
sudo ss -tne 'sport = :30001'    # NodePort 30001

# Monitor pod-to-pod communication
sudo ss -tne | grep POD_IP | wc -l

# Verify service mesh sidecar connections
sudo ss -tne | grep ':15000'     # Envoy admin port
```

---

## ⚠️ Gotchas & Pro Tips

- **Recv-Q on LISTEN sockets**: Should always be 0. If > 0, the application isn't accepting connections fast enough. Increase `listen()` backlog or add more worker processes.

- **TIME_WAIT is not a problem**: Many engineers panic about thousands of TIME_WAIT connections. This is normal and expected. The kernel reuses TIME_WAIT slots when needed. Only worry if you're hitting system limits (65535 port combinations).

- **CLOSE_WAIT is a real problem**: If you see many CLOSE_WAIT connections, your application isn't properly closing connections. Usually a resource leak in client code (not calling `close()` or `connection.close()`).

- **ss requires root for -p flag**: Without root, `-p` shows limited information. If you need process info without root, use `lsof` instead (slower but works unprivileged).

- **ss vs ss -t**: `ss -t` shows all TCP sockets (including LISTEN). `ss -ta` or `ss -tn` shows only established. `ss -tl` shows only LISTEN.

- **Orphaned sockets**: `ss -s` shows "TCP orphaned", which are sockets whose processes crashed without cleaning up. These are automatically reaped by the kernel but consuming memory. Not urgent but worth investigating.

- **Send-Q backlog**: If Send-Q is consistently high, you're sending data faster than the network can transmit. Check bandwidth, network policies (iptables), or socket buffer settings.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
