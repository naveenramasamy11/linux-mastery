# 🗺️ Day 01 — Navigation & Filesystem Basics

> You use `ls` and `cd` every day. But do you know ALL the tricks hiding behind them?

---

## The Linux Filesystem Hierarchy (Quick Map)

```
/
├── bin/        → Essential binaries (ls, cp, mv)
├── sbin/       → System binaries (fdisk, iptables)
├── etc/        → Configuration files
├── home/       → User home directories
├── var/        → Variable data (logs, spools, caches)
├── tmp/        → Temporary files (cleared on reboot)
├── proc/       → Virtual fs — kernel & process info (live!)
├── sys/        → Virtual fs — device & kernel interfaces
├── dev/        → Device files
├── usr/        → User programs & libraries
├── opt/        → Optional third-party software
├── mnt/        → Temporary mount points
├── run/        → Runtime data (PIDs, sockets)
└── lib/        → Shared libraries
```

---

## 🔧 Navigation Tricks You Should Know

### 1. Jump back to previous directory instantly
```bash
cd -
# Toggles between current and last visited directory
# Great for switching between two dirs quickly
```

### 2. Go home, fast
```bash
cd        # goes to $HOME
cd ~      # same thing
cd ~user  # goes to another user's home (if permitted)
```

### 3. `pushd` / `popd` — directory stack
```bash
pushd /var/log      # push current dir, jump to /var/log
pushd /etc/nginx    # push again, jump to /etc/nginx
dirs -v             # list the stack with indices
popd                # pop back to /var/log
popd                # pop back to original dir
```
> 💡 **Use case:** When you're navigating 3+ directories during debugging — much faster than `cd -`

### 4. `ls` tricks that save time
```bash
ls -lhtr            # Sort by time, oldest first — great for logs
ls -lSr             # Sort by size, smallest first
ls -la --color=auto # Color output with hidden files
ls -1               # One file per line (great for scripting)
ls **/*.log         # Glob — all .log files recursively (zsh / bash 4+)
\ls                 # Bypass aliases — raw ls output
```

### 5. `find` — the real filesystem navigator
```bash
# Find files modified in last 24 hours
find /var/log -mtime -1 -type f

# Find files larger than 100MB
find / -size +100M -type f 2>/dev/null

# Find and delete tmp files older than 7 days
find /tmp -mtime +7 -type f -delete

# Find SUID files (security check!)
find / -perm -4000 -type f 2>/dev/null

# Find files by inode number
find / -inum 123456 2>/dev/null
```

### 6. `tree` — visual directory structure
```bash
tree -L 2           # Show 2 levels deep
tree -d             # Directories only
tree -h             # Human-readable sizes
tree --gitignore    # Respect .gitignore
```

### 7. Brace expansion — create multiple paths at once
```bash
mkdir -p project/{src,tests,docs,scripts/{deploy,build}}
ls project/

# Create files in one shot
touch app/{main,config,utils}.py
```

### 8. Absolute vs Relative paths — know the difference
```bash
# Absolute — always starts from /
/etc/nginx/nginx.conf

# Relative — from current directory
./scripts/deploy.sh
../config/settings.yaml

# Tilde expansion
~/scripts/   →  /home/naveen/scripts/
```

---

## 🧠 Understanding Inodes

Every file = **inode** (metadata) + **data blocks**. The inode holds: permissions, owner, timestamps, size — but NOT the filename.

```bash
ls -i filename.txt        # Show inode number
stat filename.txt         # Full inode metadata
df -i                     # Inode usage per filesystem
                          # (Disk full but df shows space? Check inodes!)
```

> ⚠️ **Gotcha:** You can run out of inodes before running out of disk space. This happens with millions of small files (email spools, cache dirs).

---

## 🔗 Hard Links vs Soft Links

```bash
# Soft (symbolic) link — like a shortcut, can cross filesystems
ln -s /etc/nginx/nginx.conf ~/nginx.conf

# Hard link — same inode, survives source deletion
ln /etc/hosts /tmp/hosts_backup

# Check what a symlink points to
readlink -f ~/nginx.conf

# Find broken symlinks
find . -type l ! -exec test -e {} \; -print
```

---

## ⚡ Pro Tips

```bash
# Last argument of previous command
!$          # e.g., if you ran: cat /etc/nginx/nginx.conf
            # then:  vim !$  →  vim /etc/nginx/nginx.conf

# All arguments of previous command
!*

# Repeat last command with sudo
sudo !!

# Run a command in a subshell (doesn't change your current dir)
(cd /tmp && ls -la)

# Show disk usage of current directory, sorted
du -sh * | sort -rh | head -20
```

---

## 📋 Quick Reference

| Command | What it does |
|---------|-------------|
| `pwd` | Print working directory |
| `cd -` | Toggle between last two directories |
| `pushd/popd` | Directory stack navigation |
| `ls -lhtr` | List files, sorted by time (newest last) |
| `find / -size +100M` | Find large files |
| `stat file` | Full file metadata |
| `readlink -f` | Resolve full symlink path |
| `du -sh * \| sort -rh` | Disk usage sorted by size |

---

> **Next:** [Day 02 — File Permissions Deep Dive](./day02-file-permissions.md)
