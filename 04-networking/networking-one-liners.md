# 🌐 Networking One-Liners & Tricks

> You don't need Wireshark open every time. These command-line tools give you everything.

---

## Port & Connection Inspection

```bash
# What's listening on what port?
ss -tlnp                        # TCP listening, show PID
ss -ulnp                        # UDP listening
ss -tlnp | grep :80
netstat -tlnp 2>/dev/null       # (older systems)

# All established connections
ss -tp state established

# Connections to a specific IP
ss -tn dst 10.0.0.5

# Count connections per IP (DDoS check)
ss -tn | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head

# Check if a port is open (no netcat needed)
timeout 3 bash -c "</dev/tcp/google.com/443" && echo "open" || echo "closed"

# nc (netcat) port scan
nc -zv 10.0.0.5 20-100 2>&1 | grep succeeded
```

---

## DNS Tricks

```bash
# Quick DNS lookup
dig +short google.com
dig +short MX gmail.com
dig +short NS example.com
dig +short TXT _dmarc.example.com

# Reverse DNS lookup
dig -x 8.8.8.8 +short
host 8.8.8.8

# Trace DNS resolution path
dig +trace google.com

# Query specific nameserver
dig @8.8.8.8 example.com

# Check propagation (compare multiple resolvers)
for ns in 8.8.8.8 1.1.1.1 9.9.9.9; do
  echo -n "$ns: "; dig +short @$ns example.com
done

# Show all DNS records
dig ANY example.com

# Flush DNS cache (systemd)
systemd-resolve --flush-caches
```

---

## curl Power Tricks

```bash
# Verbose request (see headers)
curl -v https://example.com

# Only show response headers
curl -I https://example.com

# Timing breakdown (perfect for debugging slow APIs)
curl -w "\nDNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" -o /dev/null -s https://example.com

# Follow redirects
curl -L http://example.com

# Post JSON
curl -X POST https://api.example.com/data \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"key": "value"}'

# Download with resume
curl -C - -O https://example.com/bigfile.iso

# Test with custom Host header (useful for testing behind load balancers)
curl -H "Host: myapp.example.com" http://10.0.0.5/health

# Silent, check HTTP status code only
curl -s -o /dev/null -w "%{http_code}" https://example.com

# Rate limit / bandwidth limit
curl --limit-rate 100k -O https://example.com/bigfile.iso

# Use a proxy
curl -x http://proxy:3128 https://example.com

# Check SSL certificate details
curl -vI https://example.com 2>&1 | grep -A20 "Server certificate"
```

---

## tcpdump — Network Traffic Analysis

```bash
# Capture on interface eth0
tcpdump -i eth0

# Capture HTTP traffic
tcpdump -i eth0 port 80 -A      # -A = ASCII output

# Capture traffic to/from specific host
tcpdump -i eth0 host 10.0.0.5

# Capture DNS queries
tcpdump -i eth0 port 53

# Capture HTTPS traffic (see encrypted packets)
tcpdump -i eth0 port 443 -nn

# Write to pcap file (open in Wireshark)
tcpdump -i eth0 -w /tmp/capture.pcap

# Read pcap file
tcpdump -r /tmp/capture.pcap

# Capture SYN packets only (new connections)
tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'

# Exclude SSH traffic (port 22)
tcpdump -i eth0 not port 22

# Most verbose capture
tcpdump -i any -nn -vvv -XX -s 0 port 8080
```

---

## iptables — Firewall Rules

```bash
# View rules
iptables -L -n -v             # All chains, numeric, verbose
iptables -L INPUT -n -v       # Only INPUT chain
iptables -t nat -L -n -v      # NAT table

# Allow / block
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -s 10.0.0.5 -j DROP

# Block all outbound except established
iptables -P OUTPUT DROP
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -p tcp --dport 443 -j ACCEPT

# Port forwarding / redirect
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080

# Save rules
iptables-save > /etc/iptables/rules.v4
iptables-restore < /etc/iptables/rules.v4

# Delete rule by number
iptables -L INPUT --line-numbers
iptables -D INPUT 3

# Check connection tracking
conntrack -L | grep ESTABLISHED | wc -l
```

---

## nmap — Network Discovery

```bash
# Scan a host
nmap 10.0.0.5

# Scan open ports quickly
nmap -F 10.0.0.5              # Fast (top 100 ports)
nmap -p- 10.0.0.5             # All 65535 ports

# OS detection + service version
nmap -A 10.0.0.5

# Scan a subnet
nmap -sn 10.0.0.0/24          # Ping scan (alive hosts)
nmap -p 22,80,443 10.0.0.0/24

# UDP scan (often forgotten!)
nmap -sU -p 53,161 10.0.0.5

# Script scan (vulnerability check)
nmap --script vuln 10.0.0.5
nmap --script ssl-cert 10.0.0.5:443
```

---

## Network Interface & Routing

```bash
# Show interfaces
ip addr show
ip -4 a                        # IPv4 only
ip -6 a                        # IPv6 only

# Show routing table
ip route show
ip route get 8.8.8.8           # Which route to use for this IP?

# Add/remove route
ip route add 10.0.0.0/8 via 192.168.1.1 dev eth0
ip route del 10.0.0.0/8

# Monitor network events in real time
ip monitor all

# Network interface stats
ip -s link show eth0
cat /proc/net/dev

# Check MTU / change it
ip link show eth0 | grep mtu
ip link set eth0 mtu 9000      # Jumbo frames

# Capture bandwidth usage (no extra tools)
cat /proc/net/dev | awk 'NR>2 {print $1, $2, $10}'
```

---

## SSH Power Tricks

```bash
# Jump through bastion host
ssh -J bastion@10.0.0.1 user@internal-server

# SOCKS proxy (tunnel all traffic through SSH)
ssh -D 1080 user@remote-server
# Then set browser proxy to SOCKS5 localhost:1080

# Port forward — local to remote (access remote port locally)
ssh -L 5432:db-server:5432 user@bastion
# Now: psql -h localhost -p 5432

# Reverse tunnel — expose local port on remote
ssh -R 8080:localhost:8080 user@remote

# Persistent tunnel with autossh
autossh -M 0 -N -L 5432:db:5432 user@bastion &

# Copy SSH key (avoid password prompts)
ssh-copy-id user@server
ssh-copy-id -i ~/.ssh/custom_key.pub user@server

# Run remote command without login shell
ssh user@server "sudo systemctl restart nginx"

# Mount remote filesystem
sshfs user@server:/remote/path /local/mountpoint

# X11 forwarding (run GUI apps remotely)
ssh -X user@server xclock
```

---

> **Next:** [Kernel Internals](../06-kernel-internals/)
