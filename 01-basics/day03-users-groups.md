# 🧑‍💻 Day 03 — Users, Groups, sudo, su, PAM & /etc/sudoers

> Master Linux identity management. Understanding users, groups, and privilege escalation is the difference between a secure system and a ticking time bomb.

---

## 📖 Table of Contents

1. [The Linux Identity Model](#the-linux-identity-model)
2. [Managing Users](#managing-users)
3. [Managing Groups](#managing-groups)
4. [Switching Users — su vs sudo](#switching-users--su-vs-sudo)
5. [Mastering /etc/sudoers](#mastering-etcsudoers)
6. [PAM — Pluggable Authentication Modules](#pam--pluggable-authentication-modules)
7. [Password Policies & Aging](#password-policies--aging)
8. [Audit Who Did What](#audit-who-did-what)
9. [AWS-Specific Angles](#aws-specific-angles)
10. [Pro Tips & Gotchas](#pro-tips--gotchas)

---

## The Linux Identity Model

Every process and file in Linux has exactly **three identity attributes**:

| Attribute | File representation | What it controls |
|-----------|---------------------|------------------|
| UID       | `/etc/passwd`       | User ownership   |
| GID       | `/etc/group`        | Primary group    |
| Supplementary GIDs | `/etc/group` | Extra group memberships |

```bash
# See your own identity
id
# uid=1000(naveen) gid=1000(naveen) groups=1000(naveen),4(adm),27(sudo),1001(docker)

# See another user's identity
id ec2-user

# Check effective vs real UID (matters when setuid bits are involved)
cat /proc/$$/status | grep -E "^(Uid|Gid)"
# Uid:    1000    1000    1000    1000
# Real    Effective  Saved  FS-UID
```

**Key insight**: When you run `sudo`, your effective UID flips to 0 (root) but your real UID stays as your original user. Setuid binaries like `/usr/bin/passwd` exploit this same mechanism.

---

## Managing Users

### Creating Users

```bash
# Basic user creation (useradd is low-level, adduser is friendlier on Debian/Ubuntu)
sudo useradd -m -s /bin/bash -c "SRE Engineer" sre_user
#             ^  ^             ^
#             |  shell         comment/GECOS
#             create home dir

# Create a system account (no home, no login) — for services
sudo useradd -r -s /usr/sbin/nologin -c "App service account" appuser
# -r = system account (UID < 1000, no password aging)

# Create user with specific UID/GID (critical for NFS consistency across hosts)
sudo useradd -u 2001 -g 2001 -m -s /bin/bash nfsuser

# Set password immediately after creation
sudo passwd sre_user

# One-liner: create user + set password (useful in Ansible/Packer scripts)
echo "sre_user:S3cur3P@ss!" | sudo chpasswd
```

### Modifying Users

```bash
# Add user to supplementary group (docker, sudo, wheel, etc.)
sudo usermod -aG docker naveen
#             ^^ -a = APPEND (without -a, you wipe all existing groups!)

# Change user's shell
sudo usermod -s /bin/zsh naveen

# Lock a user account (prepends ! to password hash in /etc/shadow)
sudo usermod -L sre_user
sudo passwd -l sre_user   # alternative

# Unlock it
sudo usermod -U sre_user

# Expire an account on a specific date (great for contractors)
sudo usermod -e 2025-12-31 contractor_user

# Rename a user (login name only, home dir is NOT renamed automatically)
sudo usermod -l newname oldname
sudo usermod -d /home/newname -m newname  # move home dir too
```

### Deleting Users

```bash
# Delete user and home directory
sudo userdel -r olduser
# WARNING: files owned by that UID outside /home become "orphaned" (owned by UID number)

# Find orphaned files after user deletion
sudo find / -nouser -ls 2>/dev/null
# These are a security risk — fix or delete them
```

### Reading /etc/passwd and /etc/shadow

```bash
# /etc/passwd format (world-readable):
# username:x:UID:GID:GECOS:home:shell
grep naveen /etc/passwd
# naveen:x:1000:1000:Naveen Ramasamy:/home/naveen:/bin/bash
# The 'x' means password hash is in /etc/shadow (root-only)

# /etc/shadow format (root-only):
# username:hash:lastchange:min:max:warn:inactive:expire
sudo grep naveen /etc/shadow
# naveen:$6$rounds=...:19800:0:99999:7:::
#         ^                   ^  ^    ^
#         SHA-512 hash        min max warn days

# Quickly check if an account is locked
sudo passwd -S naveen
# naveen P 2024-01-15 0 99999 7 -1 (P = password set, L = locked, NP = no password)
```

---

## Managing Groups

```bash
# Create a group
sudo groupadd devops

# Create with specific GID (again: NFS/container consistency)
sudo groupadd -g 3001 devops

# Add multiple users to a group
sudo gpasswd -a naveen devops
sudo gpasswd -a sre_user devops

# Remove a user from a group
sudo gpasswd -d naveen devops

# See group members
getent group devops
# devops:x:3001:naveen,sre_user

# See all groups a user belongs to
groups naveen
id -Gn naveen

# IMPORTANT: After adding yourself to a group, you must log out/in
# OR use this trick to activate the new group in the current shell:
newgrp docker
# This spawns a subshell with docker as your active primary group
```

### /etc/group format

```bash
# groupname:password:GID:member1,member2,...
cat /etc/group | grep -E "^(sudo|docker|wheel)"
# sudo:x:27:naveen
# docker:x:998:naveen,ci_runner
```

---

## Switching Users — su vs sudo

### su (Switch User)

```bash
# Switch to root (requires root's password)
su -
# The '-' gives a login shell (loads root's environment, PATH, etc.)
# Without '-', you keep your current env — often causes "command not found" issues

# Switch to another user
su - sre_user

# Run a single command as another user without interactive shell
su -c "systemctl restart nginx" root

# su as a specific user but keep current environment (rarely what you want)
su sre_user  # no dash = no login shell
```

### sudo (Superuser Do)

```bash
# Run as root
sudo whoami  # root

# Run as a specific user
sudo -u sre_user whoami

# Run as a specific user AND group
sudo -u sre_user -g devops id

# Open a root shell (equivalent to su -)
sudo -i   # login shell — loads root's profile
sudo -s   # non-login shell — keeps current env

# Run a command with a different environment
sudo -u postgres psql -c "SELECT version();"

# List what sudo permissions you have
sudo -l
# Matching Defaults entries for naveen...
# (ALL : ALL) ALL  ← you're a full sudoer

# Re-execute the last command with sudo (bash trick)
sudo !!

# Run sudo without password prompt re-entry (cached for 15 min by default)
# Check timestamp: sudo -v refreshes the timer
sudo -v
```

---

## Mastering /etc/sudoers

**Never edit `/etc/sudoers` directly.** Always use `visudo` — it validates syntax before saving (a typo can lock you out of sudo entirely).

```bash
sudo visudo
# Or edit a specific drop-in file (preferred for production)
sudo visudo -f /etc/sudoers.d/devops-team
```

### sudoers Syntax

```
# WHO  WHERE=(AS_WHO)  WHAT
naveen ALL=(ALL:ALL) ALL
#      ^    ^   ^    ^
#      |    |   |    commands (ALL = everything)
#      |    |   run as this group
#      |    run as this user
#      from any host
```

### Real-World Examples

```bash
# Allow a user to run specific commands without password
sre_user ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl status *

# Allow a group to run all commands without password (CI/CD runners)
%cicd_runner ALL=(ALL) NOPASSWD: ALL

# Allow a user to run commands as another specific user
deploy_user ALL=(app_user) NOPASSWD: /opt/app/deploy.sh

# Restrict to specific hosts (useful in large environments)
naveen web01,web02=(ALL) ALL

# Use aliases for cleaner rules
Cmnd_Alias DOCKER_CMDS = /usr/bin/docker, /usr/bin/docker-compose
User_Alias SRE_TEAM = naveen, sre_user, ops_lead
SRE_TEAM ALL=(ALL) NOPASSWD: DOCKER_CMDS

# Require TTY (prevents sudo from cron — a common security requirement)
Defaults requiretty
# To allow specific accounts to bypass this:
Defaults:cicd_runner !requiretty

# Set sudo timeout (0 = always prompt, -1 = never expire)
Defaults timestamp_timeout=5   # 5 minutes
Defaults:naveen timestamp_timeout=30
```

### Drop-in Files (/etc/sudoers.d/)

```bash
# Production best practice: never touch /etc/sudoers directly
# Create per-team or per-app drop-ins:
echo "sre_user ALL=(ALL) NOPASSWD: /usr/bin/systemctl" | \
  sudo tee /etc/sudoers.d/sre-systemctl
sudo chmod 440 /etc/sudoers.d/sre-systemctl  # must be 440, not 644!

# List all active sudoers config
sudo cat /etc/sudoers
# At the bottom you'll see: #includedir /etc/sudoers.d
```

---

## PAM — Pluggable Authentication Modules

PAM is the authentication framework that sits beneath sudo, su, login, sshd, and almost everything that authenticates users on Linux.

```bash
# PAM config files live here
ls /etc/pam.d/
# common-auth  sshd  sudo  login  passwd  su  ...

# A typical /etc/pam.d/sudo:
cat /etc/pam.d/sudo
# auth       include      common-auth
# account    include      common-account
# session    include      common-session-noninteractive

# PAM control flags (the most important ones):
# required   — must pass; failure continues but always denies
# requisite  — must pass; failure immediately denies (no further checks)
# sufficient — if passes and no prior required failures, grant access
# optional   — result ignored (for logging/session modules)
```

### Useful PAM Modules

```bash
# pam_tally2 / pam_faillock — lock accounts after N failed attempts
# Ubuntu 20.04+
sudo cat /etc/pam.d/common-auth | grep faillock

# Check failed attempts
sudo faillock --user naveen

# Reset failed attempts
sudo faillock --user naveen --reset

# pam_limits — enforce resource limits (ulimit-style, persistent)
cat /etc/security/limits.conf
# @devops hard nofile 65536      # max open files for devops group
# sre_user soft nproc 4096       # max processes for sre_user

# Verify limits in a user's session
ulimit -a
cat /proc/$$/limits

# pam_time — restrict login hours
cat /etc/security/time.conf
# sshd;*;sre_user;Al0800-1800  # sre_user can only SSH 8am-6pm

# pam_access — restrict login by source IP
cat /etc/security/access.conf
# -:ALL EXCEPT naveen:192.168.1.0/24  # deny all except naveen from this subnet
```

---

## Password Policies & Aging

```bash
# View current password aging settings
sudo chage -l naveen
# Last password change          : Jan 15, 2025
# Password expires              : never
# Maximum number of days between password change : 99999

# Set password to expire in 90 days
sudo chage -M 90 naveen

# Force password change on next login (useful after creating accounts)
sudo chage -d 0 sre_user
# Sets last change to epoch 0, forcing immediate change

# Set minimum days between changes (prevents cycling back to old passwords)
sudo chage -m 7 naveen

# Set warning 14 days before expiry
sudo chage -W 14 naveen

# Full interactive chage
sudo chage naveen

# Password complexity via PAM (Ubuntu: pam_pwquality / pam_cracklib)
cat /etc/pam.d/common-password | grep pwquality
# password requisite pam_pwquality.so retry=3 minlen=12 difok=3 ucredit=-1 lcredit=-1 dcredit=-1

# Enforce password history (don't reuse last 5 passwords)
# In /etc/pam.d/common-password:
# password sufficient pam_unix.so ... remember=5
```

---

## Audit Who Did What

```bash
# Last logins and logouts
last | head -20
last naveen  # specific user

# Failed login attempts
lastb | head -20  # requires /var/log/btmp

# Currently logged in users
who
w  # more detailed — shows what they're running

# Who ran sudo and when (critical for compliance)
sudo grep 'sudo' /var/log/auth.log | tail -20
# Jan 15 10:23:45 web01 sudo: naveen : TTY=pts/0 ; PWD=/home/naveen ; USER=root ; COMMAND=/usr/bin/systemctl restart nginx

# Enable detailed audit logging of all commands (via auditd)
sudo auditctl -a always,exit -F arch=b64 -S execve -F uid=0 -k root_commands
sudo ausearch -k root_commands | tail -30

# Who changed /etc/sudoers?
sudo ausearch -f /etc/sudoers

# Real-time auth log monitoring (great for detecting brute-force)
sudo tail -f /var/log/auth.log | grep -E "(Failed|Accepted|sudo)"

# Check login history from wtmp (binary file)
last -F | grep -v reboot | head -20
```

---

## AWS-Specific Angles

### EC2 Instance Users

```bash
# Default users per AMI:
# Amazon Linux 2/2023: ec2-user
# Ubuntu:              ubuntu
# CentOS/RHEL:         centos or ec2-user
# Debian:              admin
# SUSE:                ec2-user
# Bitnami:             bitnami

# ec2-user is in /etc/sudoers.d/90-cloud-init-users
cat /etc/sudoers.d/90-cloud-init-users
# ec2-user ALL=(ALL) NOPASSWD:ALL

# Adding a new IAM-style user via cloud-init (UserData):
cat <<'EOF'
#cloud-config
users:
  - name: sre_user
    groups: sudo, docker
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-rsa AAAA...your-key...
EOF
```

### SSM Session Manager & sudo

```bash
# When using AWS SSM Session Manager, you connect as ssm-user
# Check its sudo rights:
sudo cat /etc/sudoers.d/ssm-agent-users
# ssm-user ALL=(ALL) NOPASSWD:ALL

# Audit SSM session activity (CloudTrail + S3 session logging)
# In SSM -> Session Manager -> Preferences -> Enable S3/CloudWatch logging

# IAM Identity Center + Linux SSSD (for larger orgs)
# Users authenticate against AWS IAM Identity Center
sudo realm join --membership-software=adcli your-domain.example.com
id someIAMuser@your-domain.example.com
```

---

## Pro Tips & Gotchas

```bash
# ⚠️ GOTCHA: usermod -G without -a WIPES all supplementary groups
sudo usermod -G docker naveen       # DANGER: removes naveen from sudo, adm, etc.
sudo usermod -aG docker naveen      # SAFE: appends to existing groups

# ⚠️ GOTCHA: new group membership requires re-login
sudo usermod -aG docker naveen
# naveen still can't run docker until they log out and back in
# Workaround: newgrp docker (spawns subshell with new group active)

# 💡 TIP: Run visudo with a specific editor
EDITOR=vim sudo visudo

# 💡 TIP: Test sudoers rules without applying them
sudo -l -U sre_user   # list what sre_user can do

# 💡 TIP: See real UID/GID of a running process
ls -la /proc/$(pgrep nginx | head -1)/status | head -5
cat /proc/$(pgrep nginx | head -1)/status | grep -E "^(Uid|Gid|Groups)"

# 💡 TIP: Find all SUID/SGID binaries (attack surface audit)
find / -perm /4000 -type f 2>/dev/null    # SUID
find / -perm /2000 -type f 2>/dev/null    # SGID
# Common legit ones: passwd, sudo, ping, su
# Unexpected ones: security risk!

# 💡 TIP: Use 'sudo -E' to preserve environment variables
export MY_ENV_VAR=value
sudo -E env | grep MY_ENV_VAR   # survives sudo with -E

# 💡 TIP: Check if a user can run a specific command via sudo
sudo -l -U sre_user | grep systemctl

# ⚠️ GOTCHA: /etc/sudoers.d/ files must be chmod 440 (not 644)
# If permissions are wrong, sudo ignores the file silently!
sudo chmod 440 /etc/sudoers.d/myfile

# 💡 TIP: Temporarily disable a user's sudo access
sudo mv /etc/sudoers.d/naveen /etc/sudoers.d/naveen.disabled
# Re-enable:
sudo mv /etc/sudoers.d/naveen.disabled /etc/sudoers.d/naveen

# 💡 TIP: getent is better than grep /etc/passwd for LDAP/AD environments
getent passwd naveen     # works even if user is in LDAP, not local files
getent group devops
```

---

## Quick Reference Cheat Sheet

```bash
# USER MANAGEMENT
useradd -m -s /bin/bash username     # create user with home + bash
usermod -aG groupname username        # add to group (safe append)
userdel -r username                   # delete user + home
passwd username                       # set/change password
chage -l username                     # view password aging
chage -d 0 username                   # force password change on login

# GROUP MANAGEMENT
groupadd groupname                    # create group
gpasswd -a username groupname         # add user to group
gpasswd -d username groupname         # remove user from group
groups username                       # list user's groups
id username                           # detailed identity info

# SUDO / PRIVILEGE
sudo -l                               # what can I sudo?
sudo -l -U username                   # what can another user sudo?
sudo -i                               # root login shell
sudo -u username command              # run as specific user
visudo                                # edit sudoers safely
visudo -f /etc/sudoers.d/file         # edit drop-in file

# AUDIT
last                                  # login history
lastb                                 # failed logins
w                                     # who is online + what running
sudo grep sudo /var/log/auth.log      # sudo activity
```

---

## Next:

➡️ [Day 04 — Text Processing Mastery: grep, awk, sed, cut, sort, uniq, tr](../01-basics/day04-text-processing.md)
