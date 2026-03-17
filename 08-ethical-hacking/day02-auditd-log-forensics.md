# 🐧 auditd & Forensics — Linux Mastery

> **Master Linux audit framework for compliance, incident response, and forensic investigation.**

## 📖 Concept

The Linux Audit Framework (auditd) is your system's forensic recorder. It captures syscalls, file access, privilege changes, and user logins at the kernel level—before applications can hide or manipulate logs. This is critical for compliance (HIPAA, PCI-DSS, SOC 2), incident response (who accessed sensitive files?), and security investigations (did this user escalate privileges?).

Unlike syslog-based logging (which apps can clear), auditd operates at the kernel level. A compromised application cannot erase audit logs without root access, and even then, audit records can be forwarded to immutable syslog servers. For DevOps and SRE, auditd answers post-incident questions: "When did the breach occur?" "What files did the attacker access?" "Which privilege escalations happened?"

The audit framework isn't enabled by default on most systems. Setting up audit rules requires planning (too much logging creates noise, misses real incidents; too little provides incomplete evidence). Understanding common audit rules (watching SUID binaries, tracking sudo use, monitoring privileged processes) is foundational for production security.

---

## 💡 Real-World Use Cases

- **Compliance auditing**: Prove that only authorized users accessed sensitive data (PCI-DSS requires audit logs for card data access)
- **Incident investigation**: Reconstruct attacker actions: what files were touched, what commands ran, what privilege escalations occurred
- **Detecting privilege escalation**: Alert when processes gain elevated privileges via sudo, su, or SUID binaries
- **File integrity monitoring**: Detect when critical system files change and by whom (configuration tampering)
- **User activity tracking**: Know which user ran which commands and when (accountability for production changes)
- **Container security**: Audit syscalls to detect container escape attempts or malicious container behavior
- **Compliance evidence**: Export audit logs as evidence for auditors (finance, security, regulatory)

---

## 🔧 Commands & Examples

### Installing and Enabling auditd

```bash
# Install auditd
apt-get install auditd              # Debian/Ubuntu
yum install audit                   # RHEL/CentOS
systemctl enable auditd
systemctl start auditd

# Verify it's running
auditctl -l        # List current audit rules
systemctl status auditd

# Check kernel version (audit needs modern kernel)
uname -r           # Should be 2.6.32+
```

### Basic audit.rules: File Monitoring

```bash
# Edit /etc/audit/audit.rules

# Watch /etc/passwd for modifications
-w /etc/passwd -p wa -k passwd-changes
# -w: watch
# -p wa: permissions (w=write, a=attribute change)
# -k: key (for searching logs later)

# Watch sensitive files
-w /etc/shadow -p wa -k shadow-changes
-w /etc/sudoers -p wa -k sudoers-changes
-w /root/.ssh -p wa -k ssh-key-changes

# Watch for new SUID files (potential privilege escalation)
-a always,exit -F dir=/usr/local/bin/ -F perm=u+s -F auid>1000 -F auid!=4294967295 -k new-suid

# Reload rules
auditctl -R /etc/audit/audit.rules
```

### Tracking User Privilege Escalation

```bash
# Watch sudo usage
-w /etc/sudoers -p wa -k sudoers-changes
-w /etc/sudoers.d -p wa -k sudoers-changes

# Track sudo command execution
-a always,exit -F arch=b64 -F exe=/usr/bin/sudo -F key=sudo-commands

# Monitor su (switch user)
-a always,exit -F arch=b64 -F exe=/usr/bin/su -F key=su-commands

# Detect unauthorized privilege escalation attempts
-a always,exit -F arch=b64 -F syscall=execve -F exe=/usr/bin/sudo -F uid>1000 -k escalation-attempt

# View sudo events
sudo ausearch -k sudoers-changes
sudo ausearch -k sudo-commands | head -20
```

### Monitoring System Calls

```bash
# Watch for all syscalls (will be VERY noisy, use sparingly)
-a always,exit -F arch=b64 -S adjtimex,settimeofday -k time-changes
# -S: filter by syscall

# Monitor open syscalls (see what processes open)
-a always,exit -F arch=b64 -S open,openat -F dir=/sensitive/dir -k sensitive-file-access

# Monitor delete/unlink syscalls (detect log deletion attempts)
-a always,exit -F arch=b64 -S unlink,unlinkat -F dir=/var/log -k log-deletion

# Monitor connect syscalls (network connections)
-a always,exit -F arch=b64 -S socket,connect,sendto,recvfrom -k network-connections
# Will show all network activity (noisy in production)
```

### Common Audit Rule Set for Production

```bash
# /etc/audit/audit.rules - Production-ready ruleset

# Remove any existing rules
-D

# Buffer Size
-b 8192

# Failure handling (0=silent, 1=printk, 2=panic)
-f 2

# File integrity monitoring
-w /etc/passwd -p wa -k passwd-changes
-w /etc/shadow -p wa -k shadow-changes
-w /etc/sudoers -p wa -k sudoers-changes
-w /etc/sudoers.d/ -p wa -k sudoers-changes

# Monitor authentication
-w /var/log/auth.log -p wa -k auth-log-changes
-w /var/log/faillog -p wa -k faillog-changes

# Monitor system configuration
-w /etc/sysctl.conf -p wa -k sysctl-changes
-w /etc/security/ -p wa -k security-config-changes

# Track admin action attempts
-a always,exit -F arch=b64 -F exe=/usr/bin/sudo -k sudo-commands
-a always,exit -F arch=b64 -F exe=/usr/bin/su -k su-commands

# Load the rules
# Run: auditctl -R /etc/audit/audit.rules
```

### Searching Audit Logs with ausearch

```bash
# List all audit log files
auditctl -l

# Search by key
ausearch -k passwd-changes         # Show all passwd changes
ausearch -k sudoers-changes        # Show sudoers modifications

# Search by user
ausearch -u 1000                   # Show all events by UID 1000
ausearch -u root                   # Show all events by username 'root'

# Search by process ID
ausearch -p 1234                   # Show events from PID 1234

# Search by time
ausearch --start today             # Events from today
ausearch --start 3/17/2026 --end 3/18/2026
ausearch --start 10:00:00 --end 12:00:00

# Search by syscall
ausearch -m syscall -S open        # All 'open' syscalls

# Search for specific file access
ausearch -f /etc/passwd            # All access to /etc/passwd

# Complex search: failed file access
ausearch -f /sensitive/file -m open -k file-access 2>/dev/null
```

### aureport: Summarizing Audit Logs

```bash
# Generate summary reports

# All audit events summary
aureport

# User account modifications summary
aureport --account

# Authentication attempts summary
aureport --auth

# File modifications summary
aureport --file

# Process execution summary
aureport --process

# Network connections summary
aureport --net

# Export report to CSV
aureport --csv --file

# Detailed report with more info
aureport --user-summary

# Focus on failed events only
aureport --failed
```

### Detecting Privilege Escalation Attempts

```bash
# Monitor SUID binary execution
-a always,exit -F arch=b64 -F exe=/usr/bin/sudo -F auid>1000 -F auid!=4294967295 -k sudo-exec

# Track capability changes
-a always,exit -F arch=b64 -S capset -k capability-changes

# Monitor failed privilege escalation
-a always,exit -F arch=b64 -S execve -F uid=0 -F auid>1000 -F auid!=4294967295 -k suid-execution

# Alert on setuid/setgid changes
-a always,exit -F arch=b64 -S chmod,fchmod,fchmodat -F amode=u+s -k suid-changes

# Query: show all privilege escalation
ausearch -m EXECVE -F uid>1000 | ausearch-interpret
```

### Real-World: Detecting SSH Key Theft

```bash
# Setup audit rules to catch SSH key access
-w /root/.ssh -p wa -k ssh-key-access
-w /home -p wa -k home-access

# After suspected breach, search for unauthorized key access
ausearch -k ssh-key-access

# Interpret results
ausearch -k ssh-key-access | ausearch-interpret

# Export for forensic investigation
ausearch -k ssh-key-access --format csv > ssh_key_access.csv

# Check if keys were copied or deleted
ausearch -k ssh-key-access -m EXECVE | grep -E "cp|scp|nc|ssh-copy"
```

### Audit Log Forwarding for Immutability

```bash
# Forward logs to syslog server (prevents local deletion)
# Edit /etc/audit/audisp-syslog.conf

active = yes
direction = out
path = builtin_syslog
type = builtin_syslog
format = string

# Redirect to remote syslog
-a always,exit -F arch=b64 -S open,openat,unlink,unlinkat -F dir=/var/log -k log-integrity

# Restart auditd
systemctl restart auditd

# Verify logs are being sent
tail -f /var/log/syslog | grep audit
```

### Container Security: Auditd for Docker

```bash
# In container, audit syscalls that indicate escape attempts
-a always,exit -F arch=b64 -S mount -k container-mount-attempt
-a always,exit -F arch=b64 -S ptrace -k process-trace-attempt
-a always,exit -F arch=b64 -S open,openat -F dir=/proc -k proc-access

# Host: Monitor container startup and suspicious syscalls
ausearch -k container-mount-attempt
ausearch -k process-trace-attempt

# Query: what did container PID 12345 do?
ausearch -p 12345 | ausearch-interpret
```

### Compliance Reporting: PCI-DSS Example

```bash
# PCI-DSS requires audit log retention and proof of access control
# Collect evidence for auditors:

# 1. User access logs
aureport --user --summary > pci-user-summary.txt

# 2. File modification logs (card data areas)
ausearch -f /var/www/payment/ | ausearch-interpret > pci-file-changes.txt

# 3. Privilege escalation attempts
ausearch -k sudoers-changes | ausearch-interpret > pci-privilege-escalation.txt

# 4. Authentication failures (detect brute-force)
aureport --auth-summary > pci-auth-summary.txt

# 5. System configuration changes
ausearch -k sysctl-changes | ausearch-interpret > pci-config-changes.txt

# Package for auditors
tar czf pci-audit-evidence-$(date +%Y%m%d).tar.gz \
    pci-user-summary.txt \
    pci-file-changes.txt \
    pci-privilege-escalation.txt \
    pci-auth-summary.txt \
    pci-config-changes.txt
```

---

## ⚠️ Gotchas & Pro Tips

- **Audit rules don't persist by default**: Manually added rules with `auditctl` disappear on reboot. Put permanent rules in `/etc/audit/audit.rules` and reload with `auditctl -R`.

- **Too many rules = log noise and performance impact**: Start with essential rules (file changes, sudo, authentication). Expand only if you have evidence of a problem. Test impact with `auditctl -l` and disk I/O monitoring.

- **Audit logs can fill disk**: Without rotation/forwarding, audit logs grow quickly. Configure log forwarding to remote syslog or set retention policies in `/etc/audit/audit.conf`.

- **ausearch-interpret is helpful**: Raw ausearch output is cryptic. Always pipe to `ausearch-interpret` for human-readable results: `ausearch -k rule | ausearch-interpret`.

- **Audit context loss after boot**: Auditd captures events, but if you reboot, event context (what user logged in) is lost. Keep audit logs across reboots with log forwarding.

- **UID filtering is crucial**: All processes run as some UID. Filter for `auid>1000` (interactive users) to exclude system processes. Otherwise, audit log becomes unreadable noise.

- **auid vs uid**: `auid` = audit user (the original login user), `uid` = current effective user. A compromised process may have uid=0 but auid=1000 (escalated from user 1000). This distinction is critical for forensics.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
