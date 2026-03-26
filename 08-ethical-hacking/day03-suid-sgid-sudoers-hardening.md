# 🐧 SUID/SGID Hunting & Sudoers Hardening — Linux Mastery

> **SUID binaries and misconfigured sudoers are the two most exploited privilege escalation vectors in Linux — knowing how to find them as a defender makes the difference between a hardened system and a low-hanging fruit.**

## 📖 Concept

### SUID and SGID

The Set-User-ID (SUID) and Set-Group-ID (SGID) permission bits are special permission flags that change how the kernel handles process ownership during execution. When a binary with the SUID bit set is executed, the resulting process runs with the *owner's* effective UID rather than the caller's. The most familiar example is `/usr/bin/passwd` — owned by root with SUID set — which lets any user run it with root privileges to modify `/etc/shadow`.

This mechanism exists for legitimate reasons, but it creates an inherent security risk: any SUID-root binary with a code execution flaw, or that can be manipulated to run arbitrary commands, becomes a direct path to root. Attackers systematically enumerate SUID binaries on every compromised system. In AWS EC2 environments, custom application binaries installed with SUID, or container breakouts that expose the host filesystem, can expose these vectors.

### Sudoers

`sudo` provides controlled privilege delegation — it allows specific users to run specific commands as root (or other users) based on rules in `/etc/sudoers`. Misconfigured sudoers entries are extremely common and frequently exploitable: `NOPASSWD` entries, wildcard matching in command paths, and allowing execution of interpreters (python, perl, vim, find) all create privilege escalation paths.

Understanding both attack vectors and defences is essential for AWS security posture. EC2 instance hardening baselines (CIS benchmarks, AWS Security Hub rules, SOC2 audits) all include SUID audits and sudoers reviews.

---

## 💡 Real-World Use Cases

- Auditing EC2 AMIs before publishing to ensure no unexpected SUID binaries exist (particularly in custom-built AMIs using Packer)
- Finding and remediating overly permissive sudoers rules identified in a security assessment
- Detecting privilege escalation by a compromised container that shares the host filesystem
- Building a hardened golden AMI that disables SUID on all non-essential binaries
- Setting up sudo rules for a CI/CD service account that can restart services but nothing else

---

## 🔧 Commands & Examples

### Finding SUID and SGID Binaries

```bash
# Find all SUID binaries on the system
find / -perm -4000 -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null    # same, different syntax

# Find all SGID binaries
find / -perm -2000 -type f 2>/dev/null
find / -perm -g=s -type f 2>/dev/null

# Find both SUID and SGID
find / -perm /6000 -type f 2>/dev/null
find / \( -perm -4000 -o -perm -2000 \) -type f 2>/dev/null

# Find SUID binaries owned by root (the dangerous ones)
find / -perm -4000 -user root -type f 2>/dev/null

# With human-readable output (ls format)
find / -perm -4000 -type f -exec ls -la {} \; 2>/dev/null

# Find SUID binaries modified recently (possible compromise indicator)
find / -perm -4000 -type f -newer /tmp/baseline_date 2>/dev/null

# Find SUID binaries not in the expected list
EXPECTED_SUID="/usr/bin/passwd /usr/bin/su /usr/bin/newgrp /usr/bin/chsh /usr/bin/chfn"
find / -perm -4000 -type f 2>/dev/null | while read binary; do
  if ! echo "$EXPECTED_SUID" | grep -qw "$binary"; then
    echo "UNEXPECTED SUID: $binary ($(ls -la $binary))"
  fi
done
```

### Baseline and Audit SUID/SGID

```bash
# Create a baseline of current SUID/SGID binaries
find / -perm /6000 -type f 2>/dev/null | sort > /var/log/suid-baseline-$(date +%Y%m%d).txt

# Compare current state to baseline (detect changes)
find / -perm /6000 -type f 2>/dev/null | sort > /tmp/suid-current.txt
diff /var/log/suid-baseline-20240101.txt /tmp/suid-current.txt
# Lines with < are in baseline but not current (removed)
# Lines with > are in current but not baseline (ADDED — investigate)

# Check if a specific binary has SUID set
stat -c "%a %A %U %n" /usr/bin/passwd
# 4755 -rwsr-xr-x root /usr/bin/passwd
# The 's' in rws = SUID is set

# Verify hash of SUID binaries against known good (integrity check)
# On a trusted system:
find / -perm -4000 -type f 2>/dev/null | xargs sha256sum 2>/dev/null > /etc/security/suid-hashes.txt
# On target system:
sha256sum -c /etc/security/suid-hashes.txt 2>/dev/null | grep FAILED
```

### Removing Unnecessary SUID Bits

```bash
# Remove SUID from a binary
chmod u-s /usr/bin/at        # remove SUID from 'at' command
chmod -s /usr/bin/newgrp     # remove both SUID and SGID

# Common candidates for SUID removal (if not needed in your environment):
# /usr/bin/at          — job scheduling (needed for cron alternatives)
# /usr/bin/chsh        — change login shell
# /usr/bin/chfn        — change full name
# /usr/bin/newgrp      — switch primary group
# /usr/sbin/mount.nfs  — if not mounting NFS
# /usr/bin/rcp         — deprecated, remove entirely
# /usr/bin/rsh         — deprecated, remove entirely
# /usr/bin/rlogin      — deprecated, remove entirely

# Bulk remove SUID from non-essential binaries
KEEP_SUID=("passwd" "sudo" "su" "ping" "mount" "umount")
find / -perm -4000 -type f 2>/dev/null | while read binary; do
  bname=$(basename "$binary")
  if ! printf '%s\n' "${KEEP_SUID[@]}" | grep -qx "$bname"; then
    echo "Removing SUID from: $binary"
    chmod u-s "$binary"
  fi
done

# Mount filesystem with nosuid (prevent SUID on a specific partition)
# /etc/fstab entry:
# /dev/xvdb  /data  ext4  defaults,nosuid  0 2
# This prevents any SUID binary on /data from elevating privileges
```

### Sudoers Configuration Deep Dive

```bash
# View current sudoers (always use visudo to edit)
visudo                          # edits /etc/sudoers
visudo -f /etc/sudoers.d/myfile # edit a drop-in file

# View effective sudo rules for a user
sudo -l -U naveen
# As the user themselves:
sudo -l

# /etc/sudoers syntax:
# user/group  host=(runas_user:runas_group)  command

# Examples:
naveen   ALL=(ALL:ALL) ALL           # full sudo (dangerous)
naveen   ALL=(ALL) NOPASSWD: ALL     # full sudo, no password (very dangerous)

# SAFER patterns:
# Allow a user to restart only specific services
deploy   ALL=(root) NOPASSWD: /bin/systemctl restart nginx, /bin/systemctl restart myapp

# Allow a group to view logs without password
%logs    ALL=(root) NOPASSWD: /usr/bin/journalctl, /usr/bin/tail /var/log/*

# Restrict to a specific host
naveen   webserver01=(root) /usr/bin/systemctl restart nginx

# Use command aliases to group commands
Cmnd_Alias SERVICES = /bin/systemctl start nginx, /bin/systemctl stop nginx, \
                       /bin/systemctl restart nginx, /bin/systemctl status nginx
naveen ALL=(root) NOPASSWD: SERVICES
```

### Identifying Dangerous Sudoers Rules

```bash
# Find all sudoers drop-in files
ls /etc/sudoers.d/

# Search for NOPASSWD entries (require review)
grep -r "NOPASSWD" /etc/sudoers /etc/sudoers.d/

# Search for overly broad rules
grep -r "ALL.*ALL" /etc/sudoers /etc/sudoers.d/

# Dangerous commands that allow privilege escalation via sudo:
# ANY of these allowed via sudo = instant root (GTFOBins)
DANGEROUS_CMDS=(
  "find" "awk" "perl" "python" "ruby" "lua" "bash" "sh" "zsh"
  "vim" "vi" "nano" "less" "more" "man"
  "tee" "dd" "cp" "mv" "chmod" "chown"
  "mount" "env" "xargs" "nmap"
  "docker" "kubectl"
  "tar" "zip" "rsync"
  "gcc" "make"
)

for cmd in "${DANGEROUS_CMDS[@]}"; do
  if sudo -l 2>/dev/null | grep -q "$cmd"; then
    echo "DANGEROUS SUDO RULE FOUND: $cmd"
  fi
done

# Check for wildcard abuse in sudoers
# DANGEROUS: /usr/bin/systemctl *  (allows systemctl edit, which can exec code)
# SAFER: /usr/bin/systemctl restart nginx
grep -r '\*' /etc/sudoers /etc/sudoers.d/ 2>/dev/null
```

### Sudoers Hardening Best Practices

```bash
# --- Minimal Privilege Pattern ---

# 1. Use /etc/sudoers.d/ drop-in files, not editing sudoers directly
# 2. One file per role/service
# 3. Always specify full binary paths

cat > /etc/sudoers.d/ci-deploy << 'EOF'
# CI/CD service account - can restart app services only
Defaults:ci-user !requiretty
Defaults:ci-user passwd_tries=0

Cmnd_Alias CI_COMMANDS = \
  /bin/systemctl restart myapp, \
  /bin/systemctl start myapp, \
  /bin/systemctl stop myapp, \
  /bin/systemctl status myapp, \
  /usr/bin/docker pull *

ci-user ALL=(root) NOPASSWD: CI_COMMANDS
EOF

# Verify the file has correct syntax before saving
visudo -c -f /etc/sudoers.d/ci-deploy

# --- Restrict sudo to specific groups ---
# In /etc/sudoers, ensure only wheel/sudo group has access
%wheel  ALL=(ALL:ALL) ALL
# Remove any direct user entries

# --- Require a separate tty for sudo (prevents sudo abuse from scripts) ---
Defaults requiretty

# --- Log all sudo commands to syslog ---
Defaults log_host, log_year, logfile="/var/log/sudo.log"
Defaults syslog=auth

# --- Timeout: require re-authentication after 5 minutes ---
Defaults timestamp_timeout=5

# --- Per-command timeout ---
Defaults!/usr/bin/apt-get timestamp_timeout=0   # always require password for apt-get
```

### Monitoring and Detecting Sudo Abuse

```bash
# View sudo command log
grep sudo /var/log/auth.log | tail -50
journalctl -t sudo | tail -50

# Find failed sudo attempts (possible brute-force or testing)
grep "sudo.*FAILED\|incorrect password\|authentication failure" /var/log/auth.log

# Find successful privilege escalations (who ran what as root)
grep "sudo:.*COMMAND" /var/log/auth.log | awk '{print $1, $2, $3, $6, $NF}' | tail -30

# Monitor sudo in real time
journalctl -f -t sudo

# Find all users who have used sudo in the last 7 days
last -F | grep -v "^$\|wtmp" | awk '{print $1}' | sort -u | while read user; do
  if grep -q "^$user " /var/log/auth.log; then
    echo "$user: $(grep "sudo.*$user" /var/log/auth.log | wc -l) sudo events"
  fi
done

# Detect new SUID binaries (run in cron or as a systemd timer)
#!/bin/bash
BASELINE="/var/security/suid-baseline.txt"
CURRENT=$(find / -perm /6000 -type f 2>/dev/null | sort)

if [ ! -f "$BASELINE" ]; then
  echo "$CURRENT" > "$BASELINE"
  exit 0
fi

NEW_BINARIES=$(comm -13 "$BASELINE" <(echo "$CURRENT"))
if [ -n "$NEW_BINARIES" ]; then
  ALERT="NEW SUID/SGID BINARIES DETECTED on $(hostname):\n$NEW_BINARIES"
  echo -e "$ALERT" | mail -s "SECURITY ALERT: New SUID Binary" security@company.com
  logger -t SECURITY "New SUID/SGID binary: $NEW_BINARIES"
fi
echo "$CURRENT" > "$BASELINE"
```

### GTFOBins — Common Escalation Techniques (Know What to Block)

```bash
# These are techniques attackers use — document them so you know what to block

# If sudo allows 'find':
sudo find /etc/passwd -exec /bin/bash \;
sudo find / -name / -type d -maxdepth 0 -exec /bin/bash -p \;

# If sudo allows 'vim':
sudo vim -c ':!/bin/bash'
# Or in vim: :set shell=/bin/bash:shell

# If sudo allows 'less':
sudo less /etc/passwd
# Then type: !/bin/bash

# If sudo allows 'tar':
sudo tar -cf /dev/null /dev/null --checkpoint=1 \
  --checkpoint-action=exec=/bin/bash

# If sudo allows 'docker':
sudo docker run -v /:/mnt --rm -it alpine chroot /mnt sh
# This gives you root on the host filesystem

# If sudo allows 'python':
sudo python3 -c 'import os; os.system("/bin/bash")'

# --- DEFENCE: Never allow these in sudoers ---
# Check your sudoers against GTFOBins: https://gtfobins.github.io
# The rule: if it can run code, it can escalate. Always.
```

---

## ⚠️ Gotchas & Pro Tips

- **`sudo -l` without a password check is exploitable:** If a user can run `sudo -l` to see rules without authenticating, attackers can enumerate available escalation paths before attempting. The `Defaults listpw=always` setting in sudoers ensures even `sudo -l` requires authentication.

- **SUID on scripts is ignored by Linux:** The Linux kernel ignores the SUID bit on script files (`.sh`, `.py`, etc.) as a security measure. Only compiled binaries honour SUID. Attackers exploit SUID *binaries* that then read/execute user-controlled files or environment variables.

- **`LD_PRELOAD` with SUID:** The dynamic linker ignores `LD_PRELOAD` and `LD_LIBRARY_PATH` when executing SUID binaries (unless the binary itself uses them). This is an intentional kernel security measure. However, `env` passed via sudo (if `env_keep` is misconfigured) can still be a vector.

- **Wildcard arguments in sudoers are dangerous:** `/usr/bin/systemctl * nginx` seems safe but `systemctl edit nginx` opens a text editor which can execute arbitrary code. Always be explicit: list each exact subcommand allowed.

- **`sudo su` and `sudo -s` are full escalations:** Rules that allow `sudo su`, `sudo bash`, or `sudo -s` are equivalent to `NOPASSWD: ALL`. Audit for these specifically.

- **AWS EC2 default 'ec2-user' has passwordless sudo:** The default AMI configuration gives `ec2-user` full passwordless sudo. In production, replace this with specific rules for your application's service accounts. Apply hardened sudoers as part of your Packer AMI build process.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
