# 🔐 Day 02 — File Permissions Deep Dive

> Permissions are the backbone of Linux security. Most people know `chmod 755` — but do you know WHY?

---

## The Permission Model

Every file/directory has three permission sets:

```
-rwxr-xr--  1  naveen  devops  4096  Mar 15 10:00  deploy.sh
│└──┘└──┘└──┘
│ │   │   └── Others (everyone else)
│ │   └─────── Group
│ └─────────── Owner (User)
└───────────── File type: - (file), d (dir), l (link), b (block), c (char)
```

Each set has: **r** (read=4), **w** (write=2), **x** (execute=1)

---

## 🔢 Octal Notation — Memorize This

```
rwx = 4+2+1 = 7
rw- = 4+2+0 = 6
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
--- = 0+0+0 = 0

Common patterns:
755 → rwxr-xr-x  (executable, world-readable)
644 → rw-r--r--  (regular file, world-readable)
600 → rw-------  (private file, e.g., SSH keys!)
700 → rwx------  (private executable)
777 → rwxrwxrwx  (NEVER use in production!)
```

---

## chmod — Change Permissions

```bash
# Octal (absolute) mode
chmod 755 script.sh
chmod 644 config.yaml
chmod 600 ~/.ssh/id_rsa     # SSH key must be 600 or ssh refuses!

# Symbolic (relative) mode — more precise
chmod u+x script.sh         # Add execute for owner
chmod g-w file.txt          # Remove write from group
chmod o=r file.txt          # Set others to read-only
chmod a+r file.txt          # Add read for ALL (u,g,o)
chmod ug=rw,o= secret.txt   # rw for owner&group, nothing for others

# Recursive
chmod -R 755 /var/www/html/
chmod -R u+X directory/     # Capital X = execute only if directory or already executable
```

> 💡 **Capital X trick:** `chmod -R a+X dir/` adds execute only to directories (not regular files). Perfect for web dirs.

---

## chown — Change Ownership

```bash
chown naveen file.txt               # Change owner
chown naveen:devops file.txt        # Change owner AND group
chown :devops file.txt              # Change only group
chown -R www-data:www-data /var/www # Recursive (web server files)
chown --reference=ref.txt new.txt   # Copy ownership from another file
```

---

## chgrp — Change Group

```bash
chgrp docker /var/run/docker.sock   # Why docker group can run docker without sudo!
chgrp -R developers /opt/project/
```

---

## 🕵️ Special Permissions — The Hidden Ones

### 1. SUID (Set User ID) — Bit 4
When set on an **executable**, it runs as the **file owner** regardless of who executes it.

```bash
chmod u+s /usr/bin/passwd
# or
chmod 4755 /usr/bin/passwd

ls -la /usr/bin/passwd
# -rwsr-xr-x  →  's' in owner execute position = SUID

# Find ALL SUID files (security audit!)
find / -perm -4000 -type f 2>/dev/null
```

> ⚠️ **Security:** SUID on shell or interpreter = instant privilege escalation. Always audit!

### 2. SGID (Set Group ID) — Bit 2
On **files**: runs as file's group.
On **directories**: new files inherit the directory's group.

```bash
chmod g+s /shared/team-dir/
chmod 2775 /shared/team-dir/

# Find SGID files
find / -perm -2000 -type f 2>/dev/null
```

> 💡 **Use case:** Shared project directories — all files created inside inherit the group. No more `chown` after every new file.

### 3. Sticky Bit — Bit 1
On **directories**: only the file owner (or root) can delete their own files. Classic example: `/tmp`

```bash
chmod +t /shared/uploads/
chmod 1777 /tmp      # This is the default /tmp permission!

ls -la /
# drwxrwxrwt  →  't' at end = sticky bit
```

---

## umask — Default Permission Mask

`umask` subtracts permissions from the default (666 for files, 777 for dirs).

```bash
umask          # Show current umask (typically 022)
umask 027      # New umask: files=640, dirs=750

# Math:
# File default 666 - umask 022 = 644
# Dir  default 777 - umask 022 = 755

# Set in ~/.bashrc for persistent change
echo "umask 027" >> ~/.bashrc
```

---

## Access Control Lists (ACLs) — Beyond Basic Permissions

When you need fine-grained control beyond user/group/other:

```bash
# Install
apt install acl   /   yum install acl

# Give specific user read access to a file
setfacl -m u:alice:r-- /opt/project/config.yaml

# Give a group write access to a directory
setfacl -m g:developers:rwx /opt/project/

# Default ACL — applied to new files in directory
setfacl -d -m g:developers:rw /opt/project/

# View ACLs
getfacl /opt/project/config.yaml

# Remove ACL
setfacl -x u:alice /opt/project/config.yaml
setfacl -b /opt/project/config.yaml   # Remove ALL ACLs

# Check if file has ACL (look for '+' in ls output)
ls -la /opt/project/
# -rw-rw-r--+  ← the '+' means ACL is set
```

---

## File Attributes — Beyond Permissions

```bash
# Make a file immutable (even root can't delete/modify!)
chattr +i /etc/resolv.conf
chattr +i /etc/hosts

# Append-only (logs!)
chattr +a /var/log/app.log

# View attributes
lsattr /etc/resolv.conf
# ----i--------e-- /etc/resolv.conf

# Remove immutable
chattr -i /etc/resolv.conf
```

> 💡 **Use case on AWS/cloud:** Lock `/etc/resolv.conf` so DHCP doesn't overwrite your DNS settings!

---

## 📋 Permission Quick Reference

| Permission | Files | Directories |
|-----------|-------|-------------|
| r (4) | Read file content | List directory contents |
| w (2) | Modify file | Create/delete files in dir |
| x (1) | Execute file | Enter directory (cd) |
| SUID (4000) | Run as owner | (no effect) |
| SGID (2000) | Run as group | New files inherit group |
| Sticky (1000) | (no effect) | Only owner can delete |

---

## ⚡ Pro Tips

```bash
# Check effective permissions for a user
namei -l /path/to/file      # Shows each component's permissions

# View numeric permissions
stat -c "%a %n" *           # Shows octal + name for all files

# Fix common web server permission issue
find /var/www -type d -exec chmod 755 {} \;
find /var/www -type f -exec chmod 644 {} \;

# Check who can write to a file
ls -la /etc/passwd   # Should NEVER be world-writable!
```

---

> **Next:** [Day 03 — Users, Groups & sudo](./day03-users-groups.md)
