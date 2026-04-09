# 🐧 Hard Links vs Soft Links — Linux Mastery

> **Understanding the difference between hard links and symbolic links is essential for scripting, backups, and debugging mysterious "file disappeared" scenarios in production.**

## 📖 Concept

Every file on a Linux filesystem is represented by an **inode** — a data structure that stores metadata (permissions, ownership, timestamps, block pointers) but NOT the filename. Filenames are stored in **directory entries**, which map a name to an inode number. This separation is the key to understanding both link types.

A **hard link** is simply another directory entry pointing to the same inode. Both the original name and the hard link are equal citizens — there is no "original" vs "link". The inode maintains a **link count** (visible via `ls -l` or `stat`). The data blocks are only freed when the link count drops to zero AND no process has the file open. This is why you can delete a running binary without crashing it — the inode persists until the process exits.

A **symbolic link (symlink)** is a special file type that contains a path string pointing to another file or directory. The kernel dereferences the path at access time. Symlinks can cross filesystem boundaries, can point to directories, and can be dangling (pointing to a non-existent target). They have their own inode and their own permissions (though those permissions are largely ignored — what matters is the target's permissions).

In AWS and DevOps work you encounter both constantly: `/etc/alternatives` uses symlinks for update-alternatives, Lambda layers use symlinks for shared libraries, and log rotation tools rely on hard link semantics to avoid losing log lines during rotation.

---

## 💡 Real-World Use Cases

- **Log rotation without losing writes:** `logrotate` creates a hard link to the old log file before truncating, so running processes keep writing to the original inode while the rotated copy is compressed.
- **Atomic binary deployments:** Deploy a new binary to `/opt/app/bin-v2`, then `ln -sf /opt/app/bin-v2 /opt/app/current` — symlink swap is atomic, zero downtime.
- **Python/Node version management:** `pyenv`, `nvm`, and `update-alternatives` all use symlinks under `/usr/local/bin` or `~/.local/bin` to switch active versions.
- **Docker image layers:** Hard links are used when copying image layers to save disk space (`cp --reflink` on btrfs/xfs, or hard links on overlay filesystems).
- **Kubernetes config injection:** ConfigMap-mounted files in pods are symlinks inside a `..data` directory, allowing atomic config updates without restarting the pod.

---

## 🔧 Commands & Examples

### Creating Links

```bash
# Create a hard link — both names point to the same inode
ln /etc/hosts /tmp/hosts-hardlink

# Create a symbolic link (use -s flag)
ln -s /etc/hosts /tmp/hosts-symlink

# Create a symlink to a directory
ln -s /var/log/nginx /tmp/nginx-logs

# Force overwrite an existing symlink (atomic update)
ln -sf /opt/app/bin-v2 /opt/app/current

# Create relative symlink (preferred for portability)
ln -s ../lib/libfoo.so.1 /usr/lib/libfoo.so
```

### Inspecting Links

```bash
# ls -l shows link type: 'l' for symlink, link count for hard links
ls -la /tmp/hosts-hardlink /tmp/hosts-symlink
# -rw-r--r-- 2 root root 224 Jan  1 00:00 /tmp/hosts-hardlink   ← link count = 2
# lrwxrwxrwx 1 root root  10 Apr  9 10:00 /tmp/hosts-symlink -> /etc/hosts

# stat shows inode number — hard links share the same inode
stat /etc/hosts /tmp/hosts-hardlink
# Both show: Inode: 4194320 (same!)

stat /tmp/hosts-symlink
# Shows its own inode, File: /tmp/hosts-symlink -> /etc/hosts

# Find all hard links to a specific inode
find /etc -inum $(stat -c '%i' /etc/hosts) 2>/dev/null

# readlink — resolve a symlink one level
readlink /tmp/hosts-symlink
# /etc/hosts

# readlink -f — resolve symlink chain to absolute real path
readlink -f /usr/bin/python3
# /usr/bin/python3.11

# Check if a symlink is dangling (broken)
[ -L /tmp/hosts-symlink ] && [ ! -e /tmp/hosts-symlink ] && echo "DANGLING"
```

### Finding and Auditing Links

```bash
# Find all symlinks under a directory
find /etc -type l

# Find all symlinks and print target
find /etc -type l -exec ls -la {} \;

# Find dangling symlinks
find /var -xtype l 2>/dev/null

# Find all hard links with link count > 1 (potential duplicates)
find /home -type f -links +1

# Find files with the same inode (hard link groups)
find /usr/bin -type f -links +1 -printf '%i %p\n' | sort -n | head -20

# Count symlinks vs hard-linked files in /usr/bin
find /usr/bin -type l | wc -l
find /usr/bin -type f -links +1 | wc -l
```

### Practical: Atomic Config/Binary Swap

```bash
# Deploy new app version atomically
NEW_VERSION="v2.3.1"
DEPLOY_DIR="/opt/myapp"

# Install new version
cp myapp-${NEW_VERSION} ${DEPLOY_DIR}/bin-${NEW_VERSION}
chmod 755 ${DEPLOY_DIR}/bin-${NEW_VERSION}

# Atomic symlink swap — no downtime
ln -sf ${DEPLOY_DIR}/bin-${NEW_VERSION} ${DEPLOY_DIR}/bin-current

# Verify
readlink -f ${DEPLOY_DIR}/bin-current

# Rollback in 1 second
ln -sf ${DEPLOY_DIR}/bin-v2.3.0 ${DEPLOY_DIR}/bin-current
```

### Hard Links for Efficient Backups

```bash
# rsync with hard links — only changed files use new inodes
rsync -av --link-dest=/backup/yesterday/ /data/ /backup/today/

# Result: unchanged files share inodes with yesterday's backup
# Check space usage
du -sh /backup/today/     # Shows tiny size (only changes)
du -sh /backup/yesterday/ # Full size
du -sh /backup/           # Total = yesterday + delta (NOT 2x)

# Verify hard link sharing
stat /backup/yesterday/somefile.txt
stat /backup/today/somefile.txt
# Same inode number!
```

### Symlink Safety in Scripts

```bash
# WRONG: Don't blindly follow symlinks when doing security-sensitive work
cat /tmp/user-provided-path  # Could be a symlink to /etc/shadow!

# RIGHT: Check if it's a symlink before acting
FILEPATH="/tmp/user-provided"
if [ -L "$FILEPATH" ]; then
    echo "ERROR: symlink not allowed" >&2
    exit 1
fi

# Check symlink doesn't escape a directory (prevent directory traversal)
REAL=$(readlink -f "$FILEPATH")
ALLOWED_BASE="/var/app/data"
if [[ "$REAL" != "$ALLOWED_BASE"* ]]; then
    echo "ERROR: path escapes allowed directory" >&2
    exit 1
fi

# Use -P flag in cd/pwd to avoid symlink traversal
cd -P /var/log/nginx   # Physical path, no symlinks
pwd -P                 # Shows real path
```

---

## ⚠️ Gotchas & Pro Tips

- **Hard links cannot cross filesystem boundaries.** If `/home` and `/data` are on different partitions, you cannot hard link between them. Use symlinks or `bind mounts` instead.
- **Hard links cannot point to directories** (on most Linux filesystems, root excepted). Attempts will fail with "hard link not allowed for directory". Symlinks to directories work fine.
- **Symlink permissions are irrelevant.** The kernel always uses `lrwxrwxrwx`. What matters is the permission of the target. However, the directory containing the symlink must be writable for you to delete it.
- **`cp -r` follows symlinks by default.** Use `cp -rP` or `cp -rd` to preserve symlinks as-is rather than copying target contents. In Dockerfiles, `COPY` follows symlinks — be aware.
- **Deleting the target breaks the symlink.** In rolling deployments or Kubernetes ConfigMaps, watch for the `..data` symlink pattern — deleting and recreating `..data` is how atomic config updates work, and inotify watchers may fire twice.
- **`rm` on a symlink removes the link, not the target.** `rm /tmp/hosts-symlink` is safe. But `rm -rf /symlinked-dir/` (with trailing slash) will delete the TARGET directory's contents on some shells — always use `unlink` or omit the trailing slash.
- **Pro tip — `namei` for tracing symlink chains:**

```bash
namei -l /usr/bin/python3
# f: /usr/bin/python3
# dr-xr-xr-x root root /
# drwxr-xr-x root root usr
# drwxr-xr-x root root bin
# lrwxrwxrwx root root python3 -> python3.11
# -rwxr-xr-x root root python3.11
```

- **inode exhaustion is a real production issue.** You can run out of inodes before running out of disk space. Each hard link does NOT consume a new inode. Check with `df -i`. This is common on mail servers or systems with millions of small files.

```bash
df -i /
# Filesystem      Inodes  IUsed   IFree IUse% Mounted on
# /dev/xvda1     6553600 512345 6041255    8% /
```

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
