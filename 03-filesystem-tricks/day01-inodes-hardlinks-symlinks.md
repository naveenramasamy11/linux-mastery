# Day 01 — Inodes, Hard Links & Symbolic Links

## Understanding Inodes

An **inode** (index node) is a data structure that stores metadata about a file or directory. Every file on a Linux filesystem has an associated inode, even directories and devices.

### What Does an Inode Contain?

```
File size
Owner UID and GID
Permissions (rwx)
Timestamps (atime, mtime, ctime)
Link count
Disk block pointers
```

### View Inode Information

```bash
# See the inode number
ls -i /path/to/file

# Detailed inode info
stat /path/to/file

# See all inodes used in a directory
ls -ia /

# Find inode number and path
find / -inum 12345 2>/dev/null
```

**Example Output:**

```bash
$ stat /etc/passwd
  File: /etc/passwd
  Size: 2234      Blocks: 8          IO Block: 4096   regular file
Device: 10302h/66306d  Inode: 1048593   Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2025-03-15 10:22:45.123456789 +0000
Modify: 2025-03-10 08:15:32.987654321 +0000
Change: 2025-03-10 08:15:32.987654321 +0000
 Birth: -
```

## Hard Links

A **hard link** is a directory entry that points directly to the same inode as the original file. Multiple hard links can reference the same inode.

### Key Characteristics

- Same inode number as the original file
- Cannot cross filesystem boundaries
- Cannot link to directories (except `.` and `..`)
- Removing a hard link only decrements the link count
- The file persists if any hard link remains

### Creating Hard Links

```bash
# Create a hard link
ln /path/to/original /path/to/hardlink

# Create multiple hard links in one go
ln /path/to/original /path/to/link1 /path/to/link2

# Verify they point to the same inode
ls -i /path/to/original /path/to/hardlink
# Both will show the same inode number
```

### Practical Example: Backup Without Disk Space

```bash
# Original file
$ ls -li database.db
1048593 -rw-r--r-- 1 user user 5G database.db

# Create hard link (appears as a separate file but uses same disk blocks)
$ ln database.db database.db.backup

# Verify same inode
$ ls -li database.db*
1048593 -rw-r--r-- 2 user user 5G database.db
1048593 -rw-r--r-- 2 user user 5G database.db.backup

# Link count is now 2
# Disk usage is STILL 5G, not 10G!
$ du -h database.db*
5.0G    database.db
```

### Real-World Use: CoW with Hard Links

```bash
# Backup a log file with hard link (instant, no disk usage)
ln /var/log/app.log /var/log/app.log.backup

# The original and backup point to same data
# When one is modified, it's copy-on-write
```

## Symbolic Links (Soft Links)

A **symbolic link** is a special file that contains the path to another file. It's a pointer by pathname, not by inode.

### Key Characteristics

- Different inode than the target
- Can cross filesystem boundaries
- Can link to directories
- Broken if the target is deleted or moved
- Smaller file size (just contains the path)

### Creating Symbolic Links

```bash
# Create a soft link
ln -s /path/to/target /path/to/symlink

# Create relative symbolic link
cd /home/user/myproject
ln -s ../data/input.csv ./input  # Relative path

# Create absolute symbolic link
ln -s /home/user/data/input.csv /home/user/myproject/input

# Verify target
readlink /path/to/symlink
```

### Practical Example: Symlink for Easy Access

```bash
# Access deep nested project quickly
ln -s /var/www/myapp/public/uploads /home/user/uploads

# Now access it directly
ls /home/user/uploads  # Shows contents of /var/www/myapp/public/uploads
```

### Find and Verify Symlinks

```bash
# Find all symlinks in a directory
find /path -type l

# Find broken symlinks
find /path -type l ! -exec test -e {} \; -print

# Find symlink and its target
ls -lha /path/to/symlink
# Output: lrwxrwxrwx 1 user user 18 Mar 15 10:22 symlink -> /path/to/target

# Resolve symlink chain (follow all links)
readlink -f /path/to/symlink
```

## Hard Links vs Symbolic Links: Quick Comparison

| Feature | Hard Link | Symbolic Link |
|---------|-----------|--------------|
| Inode | Same as original | Different |
| Disk space | No extra space | Tiny (path string) |
| Cross filesystem | ❌ No | ✅ Yes |
| Link directories | ❌ No | ✅ Yes |
| Broken by mv | ❌ No | ✅ Yes (if absolute) |
| Command | `ln` | `ln -s` |

## Advanced Tricks

### Count Number of Links

```bash
# See link count
stat -c %h /file

# Only show files with multiple links
find . -type f -links +1

# Find duplicate files (same inode)
find . -type f -printf '%i %p\n' | sort | uniq -d -w10
```

### Remove Links But Keep Files

```bash
# Hard links: just delete the link
rm /path/to/hardlink
# Original still exists, link count decreases

# Symbolic links: same thing
rm /path/to/symlink
# Original is unaffected
```

### Create Symlink with Relative Path

```bash
# From /home/user/project, create link to ../data/file.txt
cd /home/user/project
ln -s ../data/file.txt ./myfile

# Verify relative path is stored
readlink myfile
# Output: ../data/file.txt (not absolute path!)
```

## Gotchas & Tips

⚠️ **Hard links to directories are dangerous** — Can break filesystem tools

⚠️ **Circular symlinks** — Create recursion, use `readlink -f` carefully

⚠️ **Moving files with symlinks** — Absolute symlinks break, relative ones may not

💡 **Use hard links for backups** — They're space-efficient and atomic

💡 **Use symlinks for flexibility** — Easier to manage than hard links

## Real-World Scenarios

### Scenario 1: Archive Old Logs Without Duplication

```bash
# Create hard link instead of copying
ln /var/log/app.log /archive/app.log.2025-03-15

# The log still appears in /var/log/
# But we have a reference in /archive/ with zero disk overhead
```

### Scenario 2: Manage Different Versions

```bash
# Software installed in versioned directory
/opt/app/v2.1.0/

# Symlink to current version
ln -s /opt/app/v2.1.0 /opt/app/current

# Applications use /opt/app/current
# Updating is just: rm /opt/app/current && ln -s /opt/app/v2.2.0 /opt/app/current
```

### Scenario 3: Detect Modified Files

```bash
# Create hard link as a snapshot
ln /home/user/important.txt /snapshot/important.txt.orig

# Compare inodes later
stat -c %i /home/user/important.txt  # Will be same as original
stat -c %i /snapshot/important.txt.orig  # Will be same as original

# If file is modified, it gets a new inode (CoW)
# Snapshot inode stays the same — you know it's unchanged!
```

## Commands Summary

```bash
# Create hard link
ln TARGET LINK_NAME

# Create symbolic link
ln -s TARGET LINK_NAME

# Show inode numbers
ls -i

# Get detailed inode info
stat FILE

# Find files by inode
find / -inum INODE_NUMBER

# Find all symlinks
find /path -type l

# Follow symlink
readlink LINK
readlink -f LINK  # Follow all chains

# Check link count
stat -c %h FILE
```
