# 🐧 Inodes Deep Dive — Linux Mastery Day 1

> **Understanding the invisible backbone of Linux filesystems: inodes, hard links, and why disk space isn't what you think it is.**

## 📖 Concept

An **inode** is the core metadata structure that Unix filesystems use to track files. When you create a file, the filesystem doesn't just store data — it creates an inode (index node) that holds the file's metadata: ownership, permissions, size, timestamps, and pointers to where the actual data lives on disk. The filename is just a convenient label — internally, the filesystem identifies everything by inode number.

This distinction between filenames and inodes is fundamental to understanding hard links, symlinks, and why you might suddenly run out of inodes before running out of disk space. On a typical ext4 filesystem, roughly 1 inode is created per 16KB of disk space (this ratio is configurable during mkfs), so a 1TB partition might have ~67 million inodes. If you create millions of tiny files, you can exhaust inodes and be unable to create new files even though 99% of disk space is free — a scenario that happens regularly in production environments with log files, temporary files, or container image layers.

Hard links are direct references to the same inode. When you create a hard link, you're not copying the file or the inode — you're creating a new directory entry that points to the existing inode. The inode tracks a reference count (link count); the file only gets deleted when the last hard link is removed. This is why `rm` doesn't actually delete the file — it removes a directory entry. Symlinks (soft links) are different: they're files themselves that contain a pathname (they have their own inode), and they point to another file by name, not by inode.

Understanding inodes is critical for production troubleshooting. On Kubernetes nodes, inode exhaustion in `/var/lib/kubelet/pods` can silently break container scheduling. On EC2 instances running log aggregation, millions of tiny rotated logs can consume all inodes before using significant disk space. Knowing how to audit inode usage, find the culprits, and clean them up is an essential DevOps skill.

---

## 💡 Real-World Use Cases

- **Kubernetes node inode exhaustion** — A Kubernetes node stops scheduling pods because `/var/lib/kubelet` has millions of symlinks from dead containers, exhausting inodes.
- **Log file explosion** — A misconfigured logging pipeline creates millions of tiny rotated log files, consuming all inodes on `/var/log` before disk fills.
- **Docker image layer bloat** — Excessive small files in container layers waste inodes; optimizing the Dockerfile reduces inode footprint and improves build performance.
- **NFS mount issues** — NFS filesystems export inode limits; understanding these limits prevents issues when mounting home directories for thousands of users.
- **Hard link backup strategies** — Tools like `rsync --hard-links` or backup systems use hard links to deduplicate identical files, reducing inode usage and storage costs.

---

## 🔧 Commands & Examples

### Understanding Your Filesystem's Inode Capacity

```bash
# Check inode usage on all mounted filesystems
df -i

# Output example:
# Filesystem     Inodes IUsed IFree IUse% Mounted on
# /dev/sda1    65928192 3248176 62680016  5% /
# /dev/sdb1   134217728 89000000 45217728 66% /data

# Check inode details for a specific mount
df -i /var

# List inodes by usage percentage, sorted (shows which mounts are close to limits)
df -i | sort -k5 -rn
```

### Inspecting Individual Files and Inodes

```bash
# Show the inode number of a file
ls -i /path/to/file
# Output: 12345678 /path/to/file

# Show full inode information with stat
stat /path/to/file
# Output includes: Access: inode number, Size in bytes, Blocks (512-byte units)
# Also shows: Access/Modify/Change times, Uid/Gid, Link count (number of hard links)

# Find the inode number without the filename
stat -c '%i' /path/to/file

# Compare inodes of files (same inode = hard links)
ls -i file1 file2
# If both show 12345678, they're hard links to the same inode

# Get inode count for a directory tree
find /path -type f | wc -l  # File count
find /path -type l | wc -l  # Symlink count
find /path | wc -l          # Total entry count (approximates inode usage)
```

### Finding Files Consuming the Most Inodes

```bash
# Find directories with the most files (highest inode usage)
find /var -mindepth 1 -maxdepth 1 -type d -exec echo -n '{} ' \; -exec find '{}' -type f | wc -l \; | sort -k2 -rn | head -10

# More efficient approach using bash loops
for dir in /var/*/; do echo -n "$dir "; find "$dir" -type f 2>/dev/null | wc -l; done | sort -k2 -rn | head -10

# Find the single directory consuming the most inodes
find / -maxdepth 3 -type d 2>/dev/null -exec sh -c 'echo -n "{}:"; find "{}" -type f 2>/dev/null | wc -l' \; | sort -F: -k2 -rn | head -1

# Practical example: find the Kubernetes culprit
for pod_dir in /var/lib/kubelet/pods/*/; do 
  count=$(find "$pod_dir" -type f | wc -l)
  echo "$count inodes: $pod_dir"
done | sort -rn | head -10
```

### Working with Hard Links

```bash
# Create a hard link (both reference the same inode)
touch original.txt
ln original.txt hardlink.txt
ls -i original.txt hardlink.txt
# Output: 12345 original.txt  12345 hardlink.txt  (same inode number)

# Check link count before and after
stat original.txt | grep Links
# Output: Links: 2
# After creating the hard link, link count increased from 1 to 2

# Remove a hard link (the file still exists via the other link)
rm original.txt
cat hardlink.txt  # Still works!
stat hardlink.txt | grep Links
# Output: Links: 1  (now only one link remains)

# Create hard links in rsync backups (saves inodes and space)
rsync -av --hard-links /source/ /backup/
# If /source/file1 and /source/file2 are identical,
# both will be hard linked in /backup/, saving disk space

# Find all hard links to a file (all entries pointing to the same inode)
find / -inum $(stat -c %i /path/to/file) 2>/dev/null
# Lists all filenames pointing to inode of /path/to/file

# Count hard links to a file
stat -c %h /path/to/file
# Output: 3 means 3 hard links exist to this inode
```

### Working with Symlinks

```bash
# Create a symlink (creates a new inode containing the target path as text)
ln -s /path/to/original symlink
ls -i /path/to/original symlink
# Output: 12345 /path/to/original  54321 symlink  (different inode numbers)

# Symlinks have their own inode and can point to missing targets
ln -s /nonexistent/file broken_link
ls -i broken_link  # Has an inode number
cat broken_link    # Fails: No such file or directory

# Read what a symlink points to
readlink /path/to/symlink
# Output: /target/path

# Follow symlink chains
readlink -f /path/to/symlink  # Resolves all symlinks in the chain

# Find all symlinks pointing to a specific target
find / -type l -ls 2>/dev/null | grep 'target_path'

# Find broken symlinks (targets that don't exist)
find / -type l ! -exec test -e {} \; -print 2>/dev/null
```

### Monitoring Inode Usage in Production

```bash
# Alert if inode usage exceeds threshold (useful in monitoring scripts)
USAGE=$(df -i / | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$USAGE" -gt 80 ]; then
  echo "ALERT: Inode usage on / is ${USAGE}%" | mail -s "Inode Warning" ops@company.com
fi

# Nagios/Icinga check plugin style
INODE_PCT=$(df -i /var | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$INODE_PCT" -ge 90 ]; then
  echo "CRITICAL: inode usage ${INODE_PCT}%"
  exit 2
elif [ "$INODE_PCT" -ge 80 ]; then
  echo "WARNING: inode usage ${INODE_PCT}%"
  exit 1
else
  echo "OK: inode usage ${INODE_PCT}%"
  exit 0
fi

# Track inode trends over time
date >> /tmp/inode_log.txt
df -i | grep '/' >> /tmp/inode_log.txt

# Prometheus-style metric export
echo "node_filesystem_files_free{device=\"/dev/sda1\",fstype=\"ext4\"} $(df -i / | tail -1 | awk '{print $4}')" 
```

### Debugging Inode Exhaustion

```bash
# Quick scan: which filesystem is running out?
df -i | awk '$5 > 80 {print $NF}' | while read mount; do
  echo "Scanning $mount for file count..."
  find "$mount" -type f -print0 | wc -c
done

# Deep dive: find directories with excessive files
find /var -type d -exec sh -c 'count=$(find "$1" -maxdepth 1 -type f | wc -l); [ $count -gt 1000 ] && echo "$count: $1"' _ {} \;

# Kubernetes specific: check kubelet's inode pressure
kubectl top nodes  # Shows capacity
kubectl describe node <node-name> | grep -A5 Allocated
# Or check kubelet logs:
grep -i inode /var/log/kubelet.log | tail -20

# Find files by age (often accumulated old logs can be safely deleted)
find /var/log -type f -mtime +30 -ls | head -20  # Files not modified in 30 days
```

### Creating and Managing Inode-Intensive Filesystems

```bash
# Create a filesystem with custom inode ratio (fewer bytes per inode)
# Default is usually 16KB per inode; this creates 1 inode per 4KB
mkfs.ext4 -i 4096 /dev/sdX1

# For tiny files or expected many-files workloads:
mkfs.ext4 -i 2048 /dev/sdX1  # 2KB per inode (very dense)

# Check current inode ratio of an existing filesystem
fsstat /dev/sdX1 | grep -i "bytes per inode"

# Estimate inode count on a new filesystem
fsstat /dev/sdX1 | grep -i "total inode"

# For NFS exports: check server-side inode limits
df -i /export/nfs_share
# Clients mounting this share are limited by server's inode count

# Check inode soft/hard limits for a user (filesystem quotas)
quota -i username
```

---

## ⚠️ Gotchas & Pro Tips

- **Hard links can't cross filesystem boundaries** — `ln` creates hard links only within the same filesystem. Trying to hard link across mounts fails silently in some tools. Use `ln -P` to prevent this, or use symlinks instead.

- **Deleted files still consume inodes until all hard links are gone** — If a file has 3 hard links and you delete one, the inode remains. The file only becomes truly deletable when the last link is removed. This is why `lsof` shows open files that can't be found by `find` — the inode still exists because a process holds it open.

- **Symlinks to directories can create directory traversal confusion** — `cd symlink; cd ..` may not behave as expected. Use `cd -P` (physical) or `pwd -P` (print physical path) to resolve symlinks.

- **Inode exhaustion is silent on some systems** — The filesystem doesn't proactively warn you. You discover it when `touch` fails with "No space left on device" despite `df` showing free blocks. Always monitor with `df -i`.

- **Cloud environments are inode-hostile** — On AWS EC2, some instance types have gp2 volumes with sub-optimal inode ratios for applications that create many small files. Always test with realistic workloads before production.

- **Container images and inode waste** — Dockerfile layers with many tiny files create large inode footprints. Use `RUN rm -rf /tmp/*` to clean up layers, and combine RUN statements to reduce layer count. Docker buildkit's `--rm` flag helps, but monitoring inode usage in builds is essential.

- **NFS has different inode semantics** — NFS filesystems may report inode usage differently than local ext4. Always test quota behavior before relying on it. Some NFS versions don't support hard links properly.

- **Stat on /proc files is unreliable** — `stat /proc/XXX` shows inode info for the /proc entry, not the process it represents. Use `ls -i /proc/` for accurate process inode numbers.

- **FUSE and inode caching** — Network-based filesystems (FUSE, NFS, SMB) may cache inode data. Changes on the server may not immediately reflect in client `stat` calls. Use `getfacl` or `getfattr` for more accurate metadata.

- **Pro tip: Use hard links for deduplication in backups** — If you're backing up similar servers, hard linking identical files saves inodes and storage. Tools like `borg` or `restic` do this automatically, but rsync with `--hard-links` also works for simpler setups.

- **Pro tip: Monitor /proc/pressure/ on modern kernels** — Newer kernels track PSI (Pressure Stall Information) for I/O. High inode pressure correlates with I/O stalls. Check `/proc/pressure/io` to detect filesystem bottlenecks before they become critical.

---

## 🔗 Related Topics

- **02-processes-jobs/day01-process-mastery** — Processes and open file descriptors; inodes matter when debugging "too many open files"
- **03-filesystem-tricks/day02-find-mastery** — The `find` command internally traverses inodes efficiently
- **04-networking/** — NFS and filesystem mounting over networks; inode behavior differs
- **06-kernel-internals/** — The VFS layer and how the kernel manages inodes

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series — adding new content daily.*