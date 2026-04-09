# 🐧 fail2ban & Network Hardening — Linux Mastery

> **Automated brute-force protection and systematic network hardening are table stakes for any internet-facing Linux system — fail2ban is the first line of defence after your firewall.**

## 📖 Concept

`fail2ban` is an intrusion prevention daemon that monitors log files for patterns indicating malicious behaviour (failed logins, brute-force attempts, exploit scans) and automatically blocks offending IP addresses using iptables, nftables, or other configurable actions. It operates via **jails** — each jail monitors a specific service log, applies a regex filter to detect failures, and triggers ban actions when a threshold is crossed within a time window.

The system is both reactive (banning after N failures) and temporary (bans expire after a configurable `bantime`). For persistent threats you can use `bantime.increment` with multipliers, or maintain permanent blocklists. fail2ban is available as a package on all major distros and is especially important on bastion hosts, SSH jump servers, and any EC2 instance with a public IP.

Beyond fail2ban, network hardening means systematically reducing your attack surface: closing unnecessary listening services, configuring TCP wrappers, hardening `/etc/ssh/sshd_config`, setting kernel-level network security parameters via sysctl, and scanning your own system to verify the posture. The principle is defence in depth — AWS Security Groups are your outer wall, but a compromised or misconfigured system inside that wall needs its own hardening.

---

## 💡 Real-World Use Cases

- Automatically ban IPs that fail SSH authentication 5 times within 10 minutes on a public-facing bastion
- Protect a web application from credential stuffing attacks by monitoring nginx 401/403 responses
- Harden an EC2 instance to pass a CIS benchmark assessment by combining sshd hardening, sysctl settings, and fail2ban
- Scan your own EC2 instance with nmap to verify your security posture from an attacker's perspective
- Detect and respond to port scan attempts in real time using portscan jail

---

## 🔧 Commands & Examples

### Installing and Starting fail2ban

```bash
# Install
yum install fail2ban fail2ban-systemd   # RHEL/CentOS/Amazon Linux
apt install fail2ban                     # Debian/Ubuntu

# Enable and start
systemctl enable fail2ban
systemctl start fail2ban

# Check status
systemctl status fail2ban
fail2ban-client status

# fail2ban config hierarchy:
# /etc/fail2ban/fail2ban.conf        — global settings (don't edit directly)
# /etc/fail2ban/jail.conf            — default jail definitions (don't edit)
# /etc/fail2ban/fail2ban.local       — your global overrides
# /etc/fail2ban/jail.local           — your jail overrides (EDIT THIS)
# /etc/fail2ban/filter.d/            — regex filter definitions
# /etc/fail2ban/action.d/            — ban action definitions
```

### Essential jail.local Configuration

```bash
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
# Whitelist your own IPs (comma-separated)
ignoreip = 127.0.0.1/8 ::1 10.0.0.0/8 172.16.0.0/12

# Ban duration (seconds): -1 = permanent, 86400 = 24 hours
bantime = 3600

# Time window for finding failures (seconds)
findtime = 600

# Number of failures before ban
maxretry = 5

# Backend for reading logs (auto detects systemd/polling)
backend = auto

# Email notifications (configure sender/recipient)
# destemail = admin@example.com
# sender = fail2ban@example.com
# action = %(action_mwl)s   # ban + email with log lines

# Use nftables instead of iptables (for RHEL 8+ / Ubuntu 20.04+)
# banaction = nftables-multiport
# banaction_allports = nftables-allports

# Incremental ban times (multiplies bantime on repeated offenses)
bantime.increment = true
bantime.factor = 2
bantime.maxtime = 604800   # max 7 days

# SSH jail
[sshd]
enabled = true
port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
maxretry = 3
bantime  = 86400

# SSHD aggressive: shorter window, ban for longer
[sshd-aggressive]
enabled  = false
port     = ssh
filter   = sshd
logpath  = %(sshd_log)s
maxretry = 2
findtime = 120
bantime  = 604800    # 7 days

# nginx authentication failures
[nginx-http-auth]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/error.log

# nginx bad bots / 4xx flood
[nginx-limit-req]
enabled  = true
port     = http,https
logpath  = /var/log/nginx/error.log
maxretry = 10

# nginx status 400 (bad request) — catches malformed requests / scanners
[nginx-botsearch]
enabled  = true
port     = http,https
filter   = nginx-botsearch
logpath  = /var/log/nginx/access.log
maxretry = 2

# Postfix mail server
[postfix]
enabled  = false
port     = smtp,465,submission
logpath  = %(postfix_log)s

EOF

# Reload fail2ban to apply
fail2ban-client reload
```

### Managing Bans

```bash
# List all active jails
fail2ban-client status

# Status of a specific jail (shows banned IPs and stats)
fail2ban-client status sshd

# Sample output:
# Status for the jail: sshd
# |- Filter
# |  |- Currently failed: 2
# |  |- Total failed: 127
# |  `- File list: /var/log/secure
# `- Actions
#    |- Currently banned: 3
#    |- Total banned: 15
#    `- Banned IP list: 192.168.1.100 10.0.0.50 203.0.113.5

# Manually ban an IP
fail2ban-client set sshd banip 203.0.113.99

# Unban an IP (undo accidental self-ban)
fail2ban-client set sshd unbanip 10.0.0.50

# Unban across ALL jails (if you're locked out)
fail2ban-client unban 10.0.0.50

# Flush all bans in a jail
fail2ban-client set sshd unbanip --all

# View the actual iptables rules fail2ban created
iptables -L f2b-sshd -v -n
iptables -L fail2ban-sshd -v -n

# View banned IPs in the database
fail2ban-client banned
sqlite3 /var/lib/fail2ban/fail2ban.sqlite3 "SELECT ip,timeofban,bantime,jail FROM bans ORDER BY timeofban DESC LIMIT 20;"
```

### Writing Custom Filters

```bash
# Test an existing filter against a log
fail2ban-regex /var/log/nginx/error.log /etc/fail2ban/filter.d/nginx-http-auth.conf

# Test with a custom regex string
fail2ban-regex /var/log/app/app.log "Failed login attempt for user .* from <HOST>"

# Create a custom filter for an application
cat > /etc/fail2ban/filter.d/myapp-auth.conf << 'EOF'
[Definition]
# Match lines like: "2026-04-09 10:23:45 ERROR: Authentication failed from 203.0.113.5"
failregex = ^.*Authentication failed from <HOST>$
            ^.*Invalid token from <HOST>$
            ^.*Too many requests from <HOST>$

# Lines to ignore
ignoreregex = ^.*Test authentication.*$

EOF

# Add the jail for it
cat >> /etc/fail2ban/jail.local << 'EOF'

[myapp-auth]
enabled  = true
port     = 8080,443
filter   = myapp-auth
logpath  = /var/log/app/app.log
maxretry = 10
findtime = 300
bantime  = 3600

EOF

fail2ban-client reload
fail2ban-client status myapp-auth
```

### SSH Hardening (/etc/ssh/sshd_config)

```bash
# Backup first
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# Apply hardening settings
cat > /etc/ssh/sshd_config.d/99-hardening.conf << 'EOF'
# Disable password authentication (require key-based auth)
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM yes

# Disable root login
PermitRootLogin no

# Disable empty passwords
PermitEmptyPasswords no

# Limit authentication attempts per connection
MaxAuthTries 3

# Limit concurrent unauthenticated sessions
MaxStartups 10:30:60

# Disconnect idle sessions after 15 minutes
ClientAliveInterval 300
ClientAliveCountMax 3

# Use SSH protocol 2 only (default in modern OpenSSH but explicit is better)
Protocol 2

# Restrict to specific users or groups
AllowGroups sshusers admin
# Or: AllowUsers naveen deploy

# Disable X11 forwarding unless needed
X11Forwarding no

# Disable TCP forwarding if not needed (prevents SSH tunnels to bypass firewalls)
# AllowTcpForwarding no   # uncomment if tunneling is not required

# Use strong ciphers and MACs
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512

# Log level (VERBOSE logs key fingerprints)
LogLevel VERBOSE

# Disable legacy features
IgnoreRhosts yes
HostbasedAuthentication no
EOF

# Test config before reloading
sshd -t
# If OK:
systemctl reload sshd
```

### Network Hardening via sysctl

```bash
cat > /etc/sysctl.d/99-network-hardening.conf << 'EOF'
# Disable IP forwarding (unless this is a router/NAT gateway)
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# Prevent IP spoofing — reverse path filtering
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Ignore ICMP broadcast requests (Smurf attack prevention)
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Ignore bogus ICMP error responses
net.ipv4.icmp_ignore_bogus_error_responses = 1

# SYN flood protection
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 5

# Disable source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# Disable ICMP redirects (prevent MITM routing attacks)
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

# Don't send ICMP redirects
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Log martian packets (packets with impossible source addresses)
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# Increase ephemeral port range
net.ipv4.ip_local_port_range = 1024 65535

# Protect against TIME_WAIT assassination
net.ipv4.tcp_rfc1337 = 1

EOF

# Apply immediately
sysctl -p /etc/sysctl.d/99-network-hardening.conf

# Verify
sysctl net.ipv4.tcp_syncookies
sysctl net.ipv4.conf.all.rp_filter
```

### Scanning Your Own System

```bash
# Scan yourself — see what an attacker sees
# From the instance itself (loopback)
nmap -sS -O -p- 127.0.0.1

# From another instance in the same VPC
nmap -sS -sV -O -p 1-65535 10.0.1.50

# Quick scan of common ports
nmap -F 10.0.1.50

# Detect running service versions
nmap -sV --version-intensity 5 10.0.1.50

# Check for open UDP ports (requires root)
nmap -sU -p 53,67,68,69,123,161,500 10.0.1.50

# Check for default credentials on running services (careful — can cause auth lockouts)
nmap -sV --script=auth 10.0.1.50

# Verify firewall rules are working as expected
nmap -sA -p 22,80,443 10.0.1.50   # ACK scan — reveals firewall filtering

# Show listening ports on the local system
ss -tlnp                     # TCP listening
ss -ulnp                     # UDP listening
ss -tlnp | awk '{print $4}'  # extract addresses
```

### Automated Hardening Check Script

```bash
#!/bin/bash
# quick-hardening-check.sh — verify key security settings

PASS=0; FAIL=0

check() {
  local desc="$1"; local cmd="$2"; local expected="$3"
  actual=$(eval "$cmd" 2>/dev/null)
  if [[ "$actual" == "$expected" ]]; then
    echo "PASS: $desc"
    (( PASS++ ))
  else
    echo "FAIL: $desc (expected='$expected' got='$actual')"
    (( FAIL++ ))
  fi
}

# SSH checks
check "PermitRootLogin disabled" \
  "sshd -T | grep -i 'permitrootlogin' | awk '{print \$2}'" "no"
check "PasswordAuthentication disabled" \
  "sshd -T | grep 'passwordauthentication' | awk '{print \$2}'" "no"
check "MaxAuthTries ≤ 4" \
  "sshd -T | grep 'maxauthtries' | awk '\$2<=4{print \"yes\"}'" "yes"

# sysctl checks
check "TCP syncookies enabled" \
  "sysctl -n net.ipv4.tcp_syncookies" "1"
check "IP forwarding disabled" \
  "sysctl -n net.ipv4.ip_forward" "0"
check "RP filter enabled" \
  "sysctl -n net.ipv4.conf.all.rp_filter" "1"
check "ICMP redirects disabled" \
  "sysctl -n net.ipv4.conf.all.accept_redirects" "0"

# fail2ban check
check "fail2ban running" \
  "systemctl is-active fail2ban" "active"
check "sshd jail enabled" \
  "fail2ban-client status sshd 2>/dev/null | grep -c 'Filter'" "1"

echo ""
echo "Results: $PASS passed, $FAIL failed"
```

---

## ⚠️ Gotchas & Pro Tips

- **Always whitelist your own IPs first:** Set `ignoreip` in `jail.local` before enabling fail2ban. A single typo in your SSH command can get your current IP banned. On EC2, include the VPC CIDR and your bastion's IP.
- **fail2ban reads log files, not sockets:** It needs the log to exist and be writable. If you switch sshd to use systemd journald without a file backend, set `backend = systemd` and `logpath = %(journald_log)s` in the sshd jail.
- **Test with `fail2ban-regex` before deploying custom filters:** A broken regex that matches nothing will silently not protect you. Always verify with `fail2ban-regex /path/to/log /path/to/filter`.
- **bantime of -1 means permanent ban:** Great for known bad actors, but you'll need to manually unban legitimate IPs. Use `bantime.increment` for a smarter approach that escalates bans for repeat offenders.
- **sshd -T** prints the effective configuration (after includes and defaults are resolved). Always use it to verify hardening took effect: `sshd -T | grep -E 'permit|password|auth'`.
- **nmap scans trigger fail2ban:** If you run aggressive nmap scans against your own system (`-sS`, `-sV`, etc.), fail2ban may ban your scanner's IP. Whitelist it first or use `fail2ban-client unban` afterward.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
