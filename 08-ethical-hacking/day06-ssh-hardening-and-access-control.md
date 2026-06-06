# 🐧 SSH Hardening & Access Control — Linux Mastery

> **Lock down SSH properly — the single most important service to harden on any internet-facing Linux server.**

## 📖 Concept

SSH is the front door to your Linux infrastructure. By default, it's configured to be broadly compatible rather than maximally secure. Every production server needs SSH hardened — and on cloud infrastructure (EC2, GCP VMs, Azure VMs), the default configurations are frequently not secure enough for compliance requirements.

**The attack surface:** A default SSH installation listens on port 22, accepts password authentication (enabling brute-force attacks), allows root login, accepts weak cipher algorithms, and doesn't limit authentication attempts. Public-facing servers on port 22 typically see thousands of brute-force attempts per day.

**Defense in depth for SSH:**
1. **Disable password authentication** — use keys only
2. **Disable root login** — use sudo from a regular user
3. **Limit which users can SSH** — `AllowUsers` or `AllowGroups`
4. **Restrict cipher/MAC algorithms** — remove weak algorithms (CBC mode, MD5)
5. **Enforce MFA** — combine key + TOTP or key + certificate
6. **Use SSH certificates** — scale key management beyond individual `authorized_keys`
7. **Rate-limit and monitor** — fail2ban, auditd, and CloudWatch Logs

On AWS: use SSM Session Manager as a no-SSH alternative, or EC2 Instance Connect for temporary access without persistent key management.

---

## 💡 Real-World Use Cases

- Harden SSH before passing a SOC 2 or ISO 27001 audit
- Implement certificate-based SSH authentication to replace per-user key distribution across 200+ servers
- Configure `ProxyJump` bastion host patterns for private EC2 instances in a VPC
- Set up port knocking or single-packet authorization for ultra-paranoid environments
- Detect and block SSH brute-force attacks automatically with fail2ban

---

## 🔧 Commands & Examples

### sshd_config — Core Hardening Settings

```bash
# Backup current config before making changes
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

# Harden /etc/ssh/sshd_config — full production-ready configuration

cat > /etc/ssh/sshd_config << 'EOF'
# ═══════════════════════════════════════════════════════════════
# Production SSH Server Configuration — Hardened
# ═══════════════════════════════════════════════════════════════

# Network
Port 22
ListenAddress 0.0.0.0
ListenAddress ::
AddressFamily any

# Protocol
Protocol 2

# Authentication timeouts
LoginGraceTime 30s
MaxAuthTries 3
MaxSessions 10
MaxStartups 10:30:100   # 10 unauthenticated connections max, start dropping at 30%, hard limit 100

# Disable root login (always — even on new servers)
PermitRootLogin no

# Disable password authentication — KEYS ONLY
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no

# Disable unused authentication methods
UsePAM yes
GSSAPIAuthentication no
KerberosAuthentication no
HostbasedAuthentication no
IgnoreRhosts yes
IgnoreUserKnownHosts no

# Public key authentication (enabled)
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# Limit access to specific users/groups
AllowUsers naveen deploy ansible
# AllowGroups ssh-users wheel   # alternative: allow by group

# Ciphers — only strong, modern algorithms
# Remove: aes128-cbc, aes256-cbc (CBC mode), arcfour (RC4), blowfish
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr

# MACs — only HMAC-SHA2 and ETM variants
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512,hmac-sha2-256

# Key exchange — only modern algorithms
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,diffie-hellman-group-exchange-sha256

# Host keys — prefer Ed25519 and ECDSA over RSA
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_ecdsa_key
HostKey /etc/ssh/ssh_host_rsa_key

# Timeouts and keepalives
ClientAliveInterval 300
ClientAliveCountMax 2
TCPKeepAlive no    # Use SSH-level keepalive (ClientAlive*) instead of TCP

# Logging
LogLevel VERBOSE   # logs fingerprint of connecting key (for audit trail)
SyslogFacility AUTH

# Disable dangerous features
AllowAgentForwarding no    # only enable if needed for specific users
AllowTcpForwarding no      # only enable for specific bastions
X11Forwarding no
PrintLastLog yes
PrintMotd no

# Subsystems
Subsystem sftp /usr/lib/openssh/sftp-server -l INFO

# Per-user/group overrides (at the end of file)
# Allow specific users to use TCP forwarding (for bastion hosts)
Match User bastion
    AllowTcpForwarding yes
    AllowAgentForwarding yes
    PermitOpen any
    ForceCommand echo "Bastion access only - no shell"

# Allow deployment user key-based auth only
Match User deploy
    PasswordAuthentication no
    PubkeyAuthentication yes
    AllowTcpForwarding no
EOF

# Validate configuration before restarting
sshd -t && echo "Config OK" || echo "Config ERROR — do NOT restart sshd"

# Reload sshd (keeps existing sessions alive)
systemctl reload sshd
# NOT: systemctl restart sshd (would kill existing sessions)
```

### SSH Key Management

```bash
# Generate Ed25519 key (preferred — faster, more secure than RSA-4096)
ssh-keygen -t ed25519 -C "naveen@company.com" -f ~/.ssh/id_ed25519

# Generate RSA key if Ed25519 not supported (legacy systems)
ssh-keygen -t rsa -b 4096 -C "naveen@company.com" -f ~/.ssh/id_rsa

# Set correct permissions (critical — ssh refuses to use keys with wrong permissions)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/authorized_keys

# Install public key on remote host
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@remote-host
# or manually:
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys

# Check authorized_keys on remote
ssh user@remote "cat ~/.ssh/authorized_keys"

# Rotate a key — add new key, verify login, then remove old key
# Step 1: Generate new key
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_new

# Step 2: Add new public key while old key still works
ssh user@server "echo '$(cat ~/.ssh/id_ed25519_new.pub)' >> ~/.ssh/authorized_keys"

# Step 3: Verify new key works
ssh -i ~/.ssh/id_ed25519_new user@server "echo 'New key works'"

# Step 4: Remove old key from authorized_keys
OLD_KEY=$(cat ~/.ssh/id_ed25519.pub)
ssh user@server "sed -i '/$(echo $OLD_KEY | awk '{print $2}')/d' ~/.ssh/authorized_keys"

# List all authorized keys with their fingerprints
ssh-keygen -l -f ~/.ssh/authorized_keys

# Find all authorized_keys files on a system (audit)
find / -name authorized_keys 2>/dev/null | \
    xargs -I{} bash -c 'echo "=== {} ==="; cat "{}"'
```

### Certificate-Based SSH Authentication

```bash
# SSH certificates scale key management — one CA signs all keys
# Servers trust the CA, not individual keys — add/revoke without touching every server

# ─── Setting Up the CA ────────────────────────────────────────────────────────

# Generate the CA key pair (keep private key OFFLINE or in HSM/AWS KMS)
ssh-keygen -t ed25519 -C "SSH CA Production" -f /secure/ca/ssh_ca

# Put the CA public key on all servers (via Ansible/Terraform/Packer)
# /etc/ssh/sshd_config:
# TrustedUserCAKeys /etc/ssh/ssh_ca.pub
cat /secure/ca/ssh_ca.pub >> /etc/ssh/ssh_ca.pub

# ─── Signing User Keys ────────────────────────────────────────────────────────

# Sign a user's public key (validity: 1 day)
ssh-keygen -s /secure/ca/ssh_ca \
    -I "naveen@company.com" \
    -n naveen,ubuntu,ec2-user \    # valid principals (usernames allowed to log in as)
    -V +1d \                       # valid for 1 day
    ~/.ssh/id_ed25519.pub
# Creates: ~/.ssh/id_ed25519-cert.pub

# Inspect the certificate
ssh-keygen -L -f ~/.ssh/id_ed25519-cert.pub

# Use the certificate (ssh picks it up automatically if -cert.pub file exists)
ssh naveen@server
# SSH automatically uses id_ed25519-cert.pub if present

# Sign with extensions (control capabilities)
ssh-keygen -s /secure/ca/ssh_ca \
    -I "deploy-robot" \
    -n deploy \
    -V +1h \                   # 1-hour cert for CI/CD pipeline
    -O no-port-forwarding \
    -O no-x11-forwarding \
    ~/.ssh/deploy_key.pub

# ─── Server Host Certificates ─────────────────────────────────────────────────
# Eliminates the "Are you sure you want to continue connecting?" prompt

# Sign the server's host key
ssh-keygen -s /secure/ca/ssh_ca \
    -I "prod-web-01.example.com" \
    -h \                           # -h = host certificate (not user)
    -n prod-web-01.example.com \
    -V +52w \                      # valid for 1 year
    /etc/ssh/ssh_host_ed25519_key.pub
# Creates: /etc/ssh/ssh_host_ed25519_key-cert.pub

# Configure sshd to use the host certificate
echo "HostCertificate /etc/ssh/ssh_host_ed25519_key-cert.pub" >> /etc/ssh/sshd_config

# Clients trust the CA in ~/.ssh/known_hosts:
echo "@cert-authority * $(cat /secure/ca/ssh_ca.pub)" >> ~/.ssh/known_hosts
```

### Bastion Host / ProxyJump Configuration

```bash
# ~/.ssh/config — client-side configuration for bastion hosts

cat > ~/.ssh/config << 'EOF'
# Global settings
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ConnectTimeout 10
    ControlMaster auto
    ControlPath ~/.ssh/cm_%r@%h:%p
    ControlPersist 10m    # Keep connection alive for 10 min after last use
    AddKeysToAgent yes
    IdentityFile ~/.ssh/id_ed25519

# Bastion host
Host bastion
    HostName 54.123.45.67
    User ec2-user
    IdentityFile ~/.ssh/id_ed25519
    StrictHostKeyChecking accept-new

# Jump through bastion to private EC2 instances
Host 10.0.*.*
    ProxyJump bastion
    User ec2-user
    StrictHostKeyChecking no    # because private IPs might be reused

# Production servers
Host prod-*
    User naveen
    ProxyJump bastion
    IdentityFile ~/.ssh/id_ed25519_prod
    StrictHostKeyChecking yes

# AWS SSM tunnel (when no direct SSH)
Host ssm-tunnel
    HostName i-0abc123def456789    # instance ID
    User ec2-user
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
EOF

chmod 600 ~/.ssh/config

# Test ProxyJump connection
ssh -J ec2-user@bastion-ip ec2-user@10.0.1.50

# SSH multiplexing — reuse existing connection (huge speedup for Ansible)
# First connection:
ssh bastion    # opens master connection

# Subsequent connections (instant — reuses socket):
ssh bastion    # connects via ControlMaster socket

# Check active master connections
ls ~/.ssh/cm_*
```

### Monitoring and Fail2ban

```bash
# Check SSH auth logs
journalctl -u sshd -f
grep "Failed password\|Invalid user\|Accepted" /var/log/auth.log | tail -20

# Count failed login attempts by IP
grep "Failed password" /var/log/auth.log | \
    awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -20

# Check who is currently logged in
who
w
last | head -20
lastb | head -20   # failed logins

# Fail2ban configuration for SSH
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime  = 1h
findtime  = 10m
maxretry = 5
destemail = naveenramasamy11@gmail.com
sendername = Fail2ban-$(hostname)
action = %(action_mwl)s   # ban + mail with whois + log lines

[sshd]
enabled = true
port    = ssh
filter  = sshd
backend = systemd
maxretry = 3
bantime = 24h
findtime = 10m

[sshd-aggressive]
enabled  = true
port     = ssh
filter   = sshd[mode=aggressive]
logpath  = %(sshd_log)s
backend  = %(sshd_backend)s
maxretry = 2
bantime  = 72h
EOF

systemctl enable --now fail2ban

# Check ban status
fail2ban-client status sshd

# Unban an IP (if you accidentally locked yourself out)
fail2ban-client set sshd unbanip 1.2.3.4

# Whitelist your IP (never ban this)
echo "[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 YOUR.HOME.IP.ADDRESS" >> /etc/fail2ban/jail.local
```

### AWS-Specific SSH Security

```bash
# EC2 Instance Connect — temporary key injection (no persistent keys)
aws ec2-instance-connect send-ssh-public-key \
    --instance-id i-0abc123def456789 \
    --availability-zone us-east-1a \
    --instance-os-user ec2-user \
    --ssh-public-key "$(cat ~/.ssh/id_ed25519.pub)"
# Key is valid for 60 seconds only

# Then SSH normally
ssh ec2-user@public-ip

# AWS SSM Session Manager — SSH with NO open port 22
aws ssm start-session --target i-0abc123def456789

# SSM with SSH tunneling (requires SSM plugin)
ssh -o ProxyCommand='aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters portNumber=%p' \
    ec2-user@i-0abc123def456789

# Security Group best practices
# Only allow SSH (port 22) from:
# - Your office CIDR
# - Your VPN CIDR
# - Bastion host security group (not 0.0.0.0/0!)
aws ec2 authorize-security-group-ingress \
    --group-id sg-12345678 \
    --protocol tcp \
    --port 22 \
    --cidr 10.0.0.0/8   # internal VPC only — never 0.0.0.0/0

# Check security group rules for overly permissive SSH
aws ec2 describe-security-groups \
    --query 'SecurityGroups[?IpPermissions[?ToPort==`22` && IpRanges[?CidrIp==`0.0.0.0/0`]]].[GroupId,GroupName]' \
    --output table
```

---

## ⚠️ Gotchas & Pro Tips

- **Test sshd config before reloading:** Always run `sshd -t` before `systemctl reload sshd`. A syntax error in sshd_config will prevent new connections but won't kill existing ones (since you used `reload` not `restart`). Still — always have a console/KVM/SSM backup access method before hardening.

- **Don't change port 22 for security:** Port obscurity provides minimal security benefit while causing real operational pain (automation breaks, non-standard ssh commands, firewall rules). Use fail2ban and proper authentication instead.

- **`ControlMaster` is a massive Ansible speedup:** Ansible opens one SSH connection per task by default. Add `ControlMaster auto` and `ControlPersist 60s` to `~/.ssh/config` or ansible.cfg to reuse connections — Ansible runs 5-10× faster on large playbooks.

- **Log which key was used:** `LogLevel VERBOSE` in sshd_config logs the fingerprint of the authenticated key, not just that authentication succeeded. This is essential for auditing — you need to know WHICH key was used, not just that someone logged in.

- **Certificate expiry catches people off guard:** SSH certificates with `-V +1d` (1 day) are great for security but require automation to renew. If your CI/CD pipeline uses a 24-hour cert and nobody renews it, deploys silently start failing at exactly 24:00:01. Use monitoring to alert before cert expiry.

- **`~/.ssh/known_hosts` is a security control:** `StrictHostKeyChecking no` is a man-in-the-middle vulnerability. Use `StrictHostKeyChecking accept-new` instead — it accepts new hosts automatically (useful for auto-scaling) but refuses connections to hosts whose keys have changed (catches MITM attacks).

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
