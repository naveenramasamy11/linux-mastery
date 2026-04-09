# 🐧 iptables & nftables — Linux Mastery

> **Every packet entering or leaving a Linux system passes through the kernel's netfilter hooks — iptables and nftables are your interface to that machinery.**

## 📖 Concept

Netfilter is the kernel subsystem that processes network packets. It defines five hooks in the packet processing pipeline: PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING. Both iptables and nftables are userspace tools that attach rules to these hooks via the kernel's netfilter API.

`iptables` has been the standard Linux firewall tool since the late 1990s. It organises rules into **tables** (filter, nat, mangle, raw, security) and **chains** within each table. When a packet arrives, the kernel walks through the applicable chains in order. The first rule that matches wins (in most cases). If no rule matches, the chain's **policy** applies (ACCEPT or DROP).

`nftables` is the modern replacement, merged into the kernel in 3.13 and now the default on Debian 10+, RHEL 8+, and Ubuntu 20.04+. It has a cleaner, more consistent syntax, better performance (single ruleset evaluation pass instead of per-table passes), native sets and maps, and atomic rule replacement. On RHEL 8/9 and Amazon Linux 2023, `firewalld` uses nftables as its backend. Understanding both is essential since you'll encounter iptables rules in legacy systems, Docker networking, Kubernetes kube-proxy, and CNI plugins.

In AWS: EC2 security groups handle external firewall at the hypervisor level, but host-level iptables/nftables are still critical for inter-pod routing in Kubernetes, NAT for private subnet instances, and service mesh traffic steering.

---

## 💡 Real-World Use Cases

- Configure SNAT/MASQUERADE on a NAT gateway EC2 instance to allow private subnet instances to reach the internet
- Block all traffic except SSH and HTTPS on a hardened bastion host
- Use iptables DNAT to port-forward traffic on a single EC2 IP to multiple backend services
- Debug Kubernetes networking issues by tracing packets through kube-proxy's iptables rules
- Migrate a server from iptables to nftables without service interruption using `iptables-translate`

---

## 🔧 Commands & Examples

### iptables Fundamentals

```bash
# Table structure:
# filter table:  INPUT, FORWARD, OUTPUT
# nat table:     PREROUTING, INPUT, OUTPUT, POSTROUTING
# mangle table:  all five chains (packet marking, TTL, TOS)
# raw table:     PREROUTING, OUTPUT (bypass conntrack)

# View all rules in filter table (default)
iptables -L -v -n

# View nat table
iptables -t nat -L -v -n

# View mangle table
iptables -t mangle -L -v -n

# View with line numbers (useful for inserting/deleting by position)
iptables -L INPUT -v -n --line-numbers

# Flush (clear) all rules in filter table
iptables -F

# Flush a specific chain
iptables -F INPUT

# Flush nat table
iptables -t nat -F

# Delete all user-defined chains
iptables -X
```

### Basic Packet Filtering Rules

```bash
# Allow established/related connections (always put this near the top)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow SSH from specific IP only
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT

# Allow HTTP/HTTPS from anywhere
iptables -A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT

# Allow ICMP (ping)
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Drop everything else (set default policy to DROP, or add explicit DROP rule)
iptables -P INPUT DROP
iptables -P FORWARD DROP

# Allow all outbound (most common setup)
iptables -P OUTPUT ACCEPT

# Rate-limit SSH connections (brute force protection)
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
  -m recent --set --name SSH
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
  -m recent --update --seconds 60 --hitcount 4 --name SSH -j DROP

# Log dropped packets (with rate limit to avoid log flooding)
iptables -A INPUT -m limit --limit 5/min -j LOG \
  --log-prefix "IPTables-Dropped: " --log-level 4
iptables -A INPUT -j DROP
```

### NAT Rules — SNAT and MASQUERADE

```bash
# Enable IP forwarding (required for NAT gateway / router)
echo 1 > /proc/sys/net/ipv4/ip_forward
# Persist:
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl -p

# MASQUERADE — dynamic SNAT (source IP = outgoing interface IP)
# Use this when your public IP can change (e.g., DHCP, Elastic IP)
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE

# SNAT — static SNAT (source IP = fixed IP)
# Slightly better performance than MASQUERADE (no dynamic IP lookup)
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j SNAT --to-source 203.0.113.1

# DNAT — port forwarding
# Forward port 8080 on public IP to 192.168.1.10:80
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.10:80

# Forward multiple ports
iptables -t nat -A PREROUTING -p tcp -m multiport --dports 8080,8443 \
  -j DNAT --to-destination 192.168.1.10

# Redirect port locally (e.g., redirect port 80 to 8080 for non-root app)
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080

# Intercept traffic to specific IP (useful for transparent proxy)
iptables -t nat -A PREROUTING -p tcp -d 169.254.169.254 --dport 80 \
  -j DNAT --to-destination 127.0.0.1:8181
```

### Saving and Restoring Rules

```bash
# Save current rules
iptables-save > /etc/iptables/rules.v4
ip6tables-save > /etc/iptables/rules.v6

# Restore from file
iptables-restore < /etc/iptables/rules.v4

# On RHEL/CentOS (use iptables-services package)
service iptables save
systemctl enable iptables

# On Debian/Ubuntu (use iptables-persistent)
apt install iptables-persistent
netfilter-persistent save
netfilter-persistent reload
```

### Kubernetes / Docker iptables Chains

```bash
# Docker creates its own chains
iptables -L DOCKER -v -n
iptables -L DOCKER-USER -v -n       # put your custom rules here — survives Docker restarts
iptables -t nat -L DOCKER -v -n

# Kubernetes kube-proxy iptables chains
iptables -t nat -L KUBE-SERVICES -v -n
iptables -t nat -L KUBE-NODEPORTS -v -n
iptables -t nat -L KUBE-POSTROUTING -v -n

# Count rules per chain — useful for K8s scale debugging
iptables -t nat -L | grep -c Chain
iptables -t nat -L | grep "^KUBE" | wc -l

# Trace a specific packet through all chains (requires iptables-extensions)
iptables -t raw -A PREROUTING -p tcp --dport 8080 -j TRACE
iptables -t raw -A OUTPUT -p tcp --dport 8080 -j TRACE
# Then watch: dmesg | grep TRACE
```

### nftables Basics

```bash
# Check if nftables is active
nft list ruleset

# List tables
nft list tables

# Create a table
nft add table inet filter

# Create base chains (inet = IPv4 + IPv6)
nft add chain inet filter input '{ type filter hook input priority 0; policy drop; }'
nft add chain inet filter forward '{ type filter hook forward priority 0; policy drop; }'
nft add chain inet filter output '{ type filter hook output priority 0; policy accept; }'

# Add rules
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input iif lo accept
nft add rule inet filter input tcp dport 22 accept
nft add rule inet filter input tcp dport { 80, 443 } accept
nft add rule inet filter input icmp type echo-request accept
nft add rule inet filter input drop

# View rules with handles (needed for deletion)
nft -a list chain inet filter input

# Delete a rule by handle
nft delete rule inet filter input handle 5
```

### nftables — Full Ruleset File Approach (Recommended)

```bash
# /etc/nftables.conf — complete ruleset
cat > /etc/nftables.conf << 'EOF'
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    set allowed_ips {
        type ipv4_addr
        elements = { 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 }
    }

    chain input {
        type filter hook input priority 0; policy drop;

        # Loopback
        iif lo accept

        # Established connections
        ct state established,related accept

        # Drop invalid
        ct state invalid drop

        # ICMP
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept

        # SSH from internal only
        tcp dport 22 ip saddr @allowed_ips accept

        # HTTP/HTTPS from anywhere
        tcp dport { 80, 443 } accept

        # Rate-limit new SSH
        tcp dport 22 ct state new limit rate 5/minute accept

        # Log and drop the rest
        log prefix "nft-drop: " flags all
        drop
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}

table ip nat {
    chain prerouting {
        type nat hook prerouting priority -100;
        tcp dport 8080 dnat to 192.168.1.10:80
    }

    chain postrouting {
        type nat hook postrouting priority 100;
        oifname "eth0" masquerade
    }
}
EOF

# Apply the ruleset atomically
nft -f /etc/nftables.conf

# Enable on boot
systemctl enable nftables
systemctl start nftables
```

### Migrating iptables Rules to nftables

```bash
# Translate individual rules
iptables-translate -A INPUT -p tcp --dport 22 -j ACCEPT
# Output: nft add rule ip filter INPUT tcp dport 22 counter accept

# Translate an entire saved ruleset
iptables-save | iptables-restore-translate -f /etc/nftables-translated.conf
cat /etc/nftables-translated.conf   # review before applying

# Or use the migration tool
dnf install iptables-nft      # RHEL 8+ — drop-in iptables wrapper using nftables backend
# After install, iptables commands write nftables rules silently
```

### Diagnostic One-Liners

```bash
# Watch packet/byte counters live
watch -n1 'iptables -L INPUT -v -n'
watch -n1 'nft list chain inet filter input'

# Count rules in iptables (useful to check for rule bloat in K8s)
iptables -L | grep -c "^[A-Z]"
iptables -t nat -L | wc -l

# Find rules that match a specific IP
iptables -L -v -n | grep 10.0.1.50

# Test if a port is blocked (from inside the machine)
nc -zv 127.0.0.1 8080
curl -v --connect-timeout 3 http://localhost:8080

# Trace packet flow with conntrack
conntrack -L
conntrack -E    # event mode — watch connections in real time
conntrack -D -s 10.0.0.5    # delete all conntrack entries for a source IP

# Check current conntrack table size and limits
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
# If count approaches max, you'll get "connection tracking table full" and dropped packets
sysctl -w net.netfilter.nf_conntrack_max=1048576
```

---

## ⚠️ Gotchas & Pro Tips

- **Rule order is critical:** iptables evaluates rules top-to-bottom and stops at the first match. Put your ESTABLISHED/RELATED rule at the very top, or every reply packet will traverse all your rules unnecessarily.
- **`-P DROP` is dangerous remotely:** Setting `iptables -P INPUT DROP` without a saved ruleset means if iptables is flushed (e.g., during a restart), you're locked out. Always ensure your SSH rule is in place before setting a DROP policy. Use fail-safe scheduled jobs: `at now + 3 min <<< 'iptables -P INPUT ACCEPT'` — cancel if everything is OK.
- **Docker overrides iptables:** Docker inserts its own rules in FORWARD and nat tables. Add your custom rules to the `DOCKER-USER` chain to ensure they survive Docker daemon restarts and apply before Docker's rules.
- **Kubernetes conntrack exhaustion:** In large K8s clusters, the conntrack table can fill up under load, causing dropped packets that look like random network failures. Monitor `nf_conntrack_count` vs `nf_conntrack_max` and tune accordingly.
- **nftables atomic reload:** `nft -f /etc/nftables.conf` with a `flush ruleset` at the top replaces the entire ruleset atomically — no window where old and new rules coexist. This is much safer than iptables where rule replacement is not atomic.
- **AWS Security Groups vs host firewall:** Security Groups are stateful and operate at the hypervisor, before traffic reaches the instance. Host-level iptables/nftables give you additional control but don't replace Security Groups — use both in defense-in-depth.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
