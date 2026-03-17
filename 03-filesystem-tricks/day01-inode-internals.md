# 🐧 Inode Internals & Filesystem Structure — Linux Mastery Day 1

> **Master the hidden architecture that powers Linux filesystems: how files, metadata, and disk blocks really work.**

## 📖 Concept

An **inode** (index node) is the core metadata structure that represents a file or directory on a Linux filesystem. It stores everything about that file *except* the filename itself — permissions, ownership, timestamps, size, and crucially, the pointers to the actual data blocks on disk. Understanding inodes is fundamental to becoming a Linux power user, especially when diagnosing filesystem issues, recovering deleted files, or optimizing storage.

Each inode has a unique number within its filesystem. When you run `ls -i`, that number is the inode. The filename is just a *name → inode mapping* stored in the directory's data block. This separation is why you can have hard links: multiple names pointing to the same inode. It's also why moving a file within the same filesystem is instant (just updating the directory listing), while copying it requires duplicating all the data blocks.

On production systems, running out of inodes is a real problem: you could have plenty of free disk space but no inodes left to create new files. This happens in scenarios with millions of small files (temp files, logs, container layers). Understanding the inode table's size and how filesystems allocate inodes helps you prevent "Disk quota exceeded" errors that baffled many engineers.

The internal structure varies slightly between filesystems (ext4, XFS, Btrfs), but the core concept is universal. We'll focus on ext4 here, the most common on Linux servers and EC2 instances.

---

## 💡 Real-World Use Cases

- **Debugging "No space left on device" errors** — You have 50GB free, but you can't create files. Check inode exhaustion with `df -i`.
- **Hard link strategies for backups** — Using hard links to avoid duplicating data when creating snapshots or copies.
- **Recovering deleted files** — Understanding inode recovery tools like `extundelete` and how they work.
- **Optimizing Docker layers and container storage** — Container images are built from layered filesystems where inode efficiency matters.
- **Capacity planning for log directories** — Predicting when you'll run out of inodes, not just disk space.
- **Identifying files with zero links** — "Deleted" files still consuming space because a process holds them open (see `/proc/[pid]/fd`).

---

## 🔧 Commands & Examples

### Reading Inode Information

```bash
# Display inode number for a file
ls -i /etc/passwd
# Output: 12345 /etc/passwd

# List inodes for all files in a directory
ls -i /tmp/

# Show inode information in long format (including inode in most-detailed output)
stat /etc/hostname
# Shows: Access: (0644/-rw-r--r--), Uid: (0/root), Gid: (0/root), inode: 5243

# Get detailed inode info, including all timestamps
stat -c "Inode: %i, Links: %h, Size: %s bytes, Access: (%a/%A), Owner: %U:%G, Modified: %y" /etc/fstab
```

### Understanding Hard Links and Link Counts

```bash
# Create a hard link
touch original.txt
ln original.txt hardlink.txt

# Check link count (third column after permissions)
ls -li original.txt hardlink.txt
# Output:
# 12345 -rw-r--r-- 2 user user 1024 Mar 17 10:00 original.txt
# 12345 -rw-r--r-- 2 user user 1024 Mar 17 10:00 hardlink.txt
# Both have same inode (12345) and link count (2)

# Verify they point to the same inode
test original.txt -ef hardlink.txt && echo "Same inode" || echo "Different inode"

# Remove one name; the inode stays because link count is still 1
rm hardlink.txt
stat original.txt  # Still exists, link count is now 1

# Try creating hard link to directory (FAILS on most systems)
mkdir testdir
ln testdir hardlink_dir  # Permission denied
# Reason: prevents filesystem cycles
```

### Detecting and Diagnosing Inode Exhaustion

```bash
# Check inode usage across all filesystems
df -i
# Output:
# Filesystem     Inodes   IUsed   IFree IUse% Mounted on
# /dev/xvda1    524288  490000   34288   94% /
# /dev/xvdb1   1048576  100000  948576   10% /data

# Detailed inode usage with human-readable format
df -ih
# Shows percentages; 95%+ is a warning sign

# Find directories with the most files (heavy inode usage)
find / -type d -exec sh -c 'echo "$(find "$1" -maxdepth 1 -type f | wc -l) $1"' _ {} \; 2>/dev/null | sort -rn | head -20

# Check inodes used by a specific directory tree
find /var/log -type f -printf '.' 2>/dev/null | wc -c
# Counts files (each dot represents a file, thus an inode)

# Identify and remove old temporary files consuming inodes
find /tmp -type f -atime +30 -exec rm -f {} \;  # Delete files not accessed in 30 days

# On systemd systems, find temporary files
systemd-tmpfiles --clean --verbose  # Review and clean automatically managed temp files
```

### Viewing Inode Table and Filesystem Metadata

```bash
# Display filesystem superblock information (includes inode count)
dumpe2fs /dev/xvda1 | head -30
# Shows: Filesystem volume name, Inode count, Block count, Block size, etc.

# Show inode size for the filesystem
dumpe2fs /dev/xvda1 | grep "Inode size"
# ext4 typically uses 256 or 512 bytes per inode

# Get block group information (inodes are allocated per block group)
dumpe2fs -h /dev/xvda1 | grep "Block group"

# List inode and block allocation bitmaps (diagnostic)
dumpe2fs /dev/xvda1 | grep "Inode bitmap\|Block bitmap"
```

### Finding Files Holding Deleted Inode Data

```bash
# A "deleted" file still on disk: process holds the file descriptor open
lsof | grep deleted
# Shows files that exist in the filesystem but have been unlinked from directories

# Example: Find the large "deleted" file consuming space
lsof | grep deleted | awk '{print $7}' | sort -rn | head -5

# Truncate a large deleted file (without restarting the process)
# First, find the process and file descriptor
lsof | grep "deleted" | grep "large_log"
# Output: rsyslog 1234 root 12 REG 10,1 5000000000 1048576 /var/log/messages (deleted)

# Truncate it via the /proc filesystem (sends the file to zero size without closing it)
truncate -s 0 /proc/1234/fd/12
# The space is freed but the process continues to run

# Verify the file is now smaller
ls -l /proc/1234/fd/12
```

### Hard Links vs Symbolic Links: Inode Perspective

```bash
# Create both types of links
touch original.txt
ln original.txt hardlink.txt       # Hard link: same inode
ln -s original.txt symlink.txt     # Symbolic link: different inode

# Check their inode numbers
ls -i original.txt hardlink.txt symlink.txt
# original.txt:  12345 (original inode)
# hardlink.txt:  12345 (same inode)
# symlink.txt:   12346 (different inode, points to the name "original.txt")

# Move the original file; symlink breaks, hardlink still works
mv original.txt original_moved.txt

# Try to access each
cat hardlink.txt        # Works! hardlink points to the inode, not the filename
cat symlink.txt         # Fails: symlink points to the name "original.txt", which no longer exists

# Verify symlink breakage
stat symlink.txt        # Shows "No such file or directory"
ls -l symlink.txt       # Shows a dangling symlink
```

### Inode Allocation Strategies and Block Groups

```bash
# ext4 allocates inodes in block groups to improve locality
# Check block group layout
dumpe2fs /dev/xvda1 | grep -A 2 "Group 0:"
# Output shows inodes per group, inode table block ranges

# For XFS (alternative filesystem), check inode clusters
xfs_db -c "stat" /dev/xvda1
# Shows inode cluster size and allocation preferences

# Monitor inode allocation in real time (ext4)
cat /proc/fs/ext4/xvda1/mb_groups  # Shows block allocation state
```

---

## ⚠️ Gotchas & Pro Tips

- **Hard links and directories:** You *cannot* create hard links to directories (on most filesystems). The filesystem prevents cycles by disallowing this. Use bind mounts instead if you need multiple names for the same directory.

- **Hard links across filesystems don't work:** A hard link must point to an inode on the *same filesystem*. If you try `ln /home/file /data/copy` and `/home` and `/data` are different filesystems, it fails. This is why `cp` is needed for cross-filesystem copying.

- **Inode number reuse:** Once you delete a file (unlink the inode), that inode number can be reused for new files. There's no guarantee a specific inode number persists.

- **Small filesystems run out of inodes first:** A filesystem formatted with default inode density might allocate only 1 inode per 16KB of disk space. On a 1TB filesystem with millions of 4KB files, you'll exhaust inodes before disk space. Re-format with `-N` flag to increase inode count (e.g., `mkfs.ext4 -N 10000000 /dev/xvda1`).

- **Tip: Recovering deleted files:** If you've accidentally deleted files, tools like `extundelete` can scan the inode table for unlinked inodes and recover their data (if not overwritten). Time is critical—stop writing to the filesystem immediately.

- **Tip: Using `find` with `-inum`:** Once you identify an inode number (e.g., from `lsof`), find all hard links to it: `find / -inum 12345 2>/dev/null`. Useful for tracking which names point to the same inode.

- **Symlink sizes are deceptive:** A symlink's size is the length of the path it points to, not the target file's size. `ls -l` shows symlink size, not target size.

- **Inode caching by the kernel:** The kernel keeps a cache of inode metadata in RAM (dentry cache). On systems under memory pressure, this cache shrinks, causing more frequent disk reads. Monitor with `cat /proc/slabinfo | grep dentry`.

---

## 🔗 Related Topics

- **Day 02 (upcoming):** `find` command wizardry — using `-inode`, `-exec`, and `-xargs` for advanced file searches
- **Day 03 (upcoming):** Bind mounts and `mount` tricks — alternative ways to organize filesystems
- **Section 02 (processes):** Understanding file descriptors and `/proc/[pid]/fd` — how processes reference open files
- **Section 06 (kernel):** `/proc` and `/sys` deep dive — where the kernel exposes inode and filesystem info
- **Bonus:** `extundelete`, `testdisk`, and file recovery techniques

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series — adding new content daily.*
