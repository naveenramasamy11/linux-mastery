# Day 02 — Find Command Mastery

The `find` command is one of the most powerful tools in Linux, but most users only scratch the surface. Let's dive deep.

## Basic Syntax

```bash
find [PATH] [EXPRESSION]
```

## Finding by Name

```bash
# Find by exact name
find /path -name "filename.txt"

# Case-insensitive search
find /path -iname "FILENAME.txt"

# Find by pattern (glob)
find /path -name "*.log"
find /path -name "test_*"

# Find by regex pattern
find /path -regex ".*\.log$"
find /path -regextype posix-extended -regex "test_[0-9]+\.txt"
```

## Finding by Type

```bash
# Regular files only
find /path -type f

# Directories only
find /path -type d

# Symlinks
find /path -type l

# Character devices
find /path -type c

# Block devices
find /path -type b

# Sockets
find /path -type s

# Named pipes (FIFO)
find /path -type p
```

## Finding by Size

```bash
# Exactly 1MB
find /path -size 1M

# Larger than 100MB
find /path -size +100M

# Smaller than 10KB
find /path -size -10k

# Size units: c(bytes), k(KB), M(MB), G(GB)
find /path -size +1G -size -10G  # Between 1GB and 10GB

# Find empty files
find /path -size 0
```

## Finding by Time

```bash
# Modified in the last 24 hours
find /path -mtime -1

# Modified more than 30 days ago
find /path -mtime +30

# Modified between 7-14 days ago
find /path -mtime +7 -mtime -14

# Accessed in the last hour (in minutes)
find /path -amin -60

# Changed metadata within 2 days
find /path -ctime -2

# Modified within the last 5 minutes (very recent changes)
find /path -mmin -5
```

### Time Units Explained

```
-mtime -1   → Modified less than 24 hours ago
-mtime +1   → Modified more than 24 hours ago
-mmin -30   → Modified in the last 30 minutes
-atime -7   → Accessed in the last 7 days
-ctime -1   → Metadata changed in the last 24 hours
```

## Finding by Permissions

```bash
# Exact permissions (644 = rw-r--r--)
find /path -perm 644

# At least these permissions (at least 644)
find /path -perm -644

# Any of these permission bits set
find /path -perm /755

# Find files without execute permission
find /path -type f ! -perm /111

# Find files readable by owner only
find /path -perm 400
```

## Finding by Owner

```bash
# Files owned by specific user
find /path -user username

# Files owned by specific UID
find /path -uid 1000

# Files owned by specific group
find /path -group groupname

# Files owned by specific GID
find /path -gid 1000

# Files NOT owned by current user
find /path ! -user $USER
```

## Advanced Expressions

### Combining Conditions with Logical Operators

```bash
# AND (default)
find /path -name "*.log" -mtime +30
# Finds *.log files modified more than 30 days ago

# OR
find /path -name "*.log" -o -name "*.bak"
# Finds files ending in .log OR .bak

# NOT
find /path ! -name "*.log"
# Finds everything EXCEPT *.log files

# Complex expression with parentheses
find /path \( -name "*.log" -o -name "*.tmp" \) -mtime +7
# Finds *.log or *.tmp files older than 7 days
```

### Using -exec for Actions

```bash
# Execute a command for each found file
find /path -name "*.log" -exec rm {} \;

# Safer approach (asks for confirmation)
find /path -name "*.log" -ok rm {} \;

# Multiple commands
find /path -name "*.js" -exec echo "File: {}" \; -exec wc -l {} \;

# Better performance: + instead of \; (pass multiple files at once)
find /path -name "*.log" -exec rm {} +

# Using printf instead of -exec
find /path -name "*.log" -printf "%p\n"
```

### Using -print0 for Handling Special Characters

```bash
# Pipe to xargs safely (handles filenames with spaces/newlines)
find /path -name "*.log" -print0 | xargs -0 rm

# Or with exec
find /path -name "*.log" -print0 | xargs -0 -I {} rm {}
```

## Real-World Examples

### Example 1: Find and Delete Old Log Files

```bash
# Find log files modified more than 90 days ago and delete them
find /var/log -name "*.log" -mtime +90 -delete

# Safer: just list them first
find /var/log -name "*.log" -mtime +90
```

### Example 2: Find Large Files to Free Up Disk Space

```bash
# Find files larger than 100MB
find / -type f -size +100M -exec ls -lh {} \;

# Show as sorted list
find / -type f -size +100M -printf "%s %p\n" | sort -rn | head -20
```

### Example 3: Find Recently Modified Source Code

```bash
# C files modified in the last 7 days
find /src -name "*.c" -mtime -7

# All source files (*.c, *.h, *.java) changed today
find /src \( -name "*.c" -o -name "*.h" -o -name "*.java" \) -mtime -1
```

### Example 4: Find Duplicate Files

```bash
# Find files with same size, then check with diff
find /path -type f -exec md5sum {} \; | sort -k1 | uniq -d -w32

# Or using find alone: find all files with multiple links (duplicates)
find /path -type f -links +1
```

### Example 5: Find and Fix Permissions

```bash
# Find files with wrong permissions and fix them
find /home -type f -perm 777 -exec chmod 644 {} \;

# Find directories and make them executable for owner
find /home -type d -perm 644 -exec chmod 755 {} \;
```

## Performance Tips

### Tip 1: Limit the Search Depth

```bash
# Only search 2 levels deep
find /path -maxdepth 2 -name "*.log"

# Search at least 2 levels deep
find /path -mindepth 2 -name "*.log"
```

### Tip 2: Prune Unnecessary Directories

```bash
# Skip .git directories during search
find /path -name .git -prune -o -name "*.js" -print

# Skip multiple directories
find /path -name .git -prune -o -name node_modules -prune -o -name "*.js" -print

# Skip hidden directories
find /path -not -path "*/\.*" -name "*.log"
```

### Tip 3: Use -printf Instead of -exec for Speed

```bash
# Slow: uses -exec for each file
find /path -type f -exec ls -l {} \;

# Fast: uses printf for formatting
find /path -type f -printf "%p %s\n"
```

### Tip 4: Combine with locate for Speed

```bash
# If mlocate is installed, use locate first
locate "*.log" | head -10

# Then use find for detailed filtering
find /path $(locate "*.log" | head -10) -mtime +30
```

## Common Patterns Cheat Sheet

```bash
# Find all Python files modified today
find . -name "*.py" -mtime 0

# Find empty files (potential junk)
find . -type f -size 0

# Find all symlinks pointing to non-existent files
find . -type l ! -exec test -e {} \; -print

# Find setuid binaries (potential security issue)
find / -type f -perm /4000

# Find files modified by root in /home (suspicious)
find /home -type f -user root ! -path "*/\.*"

# Find world-writable files (security concern)
find / -type f -perm -002

# Find all .zip files and extract them
find /path -name "*.zip" -exec unzip {} \;

# Find and count lines in all Python files
find . -name "*.py" -exec wc -l {} + | tail -1
```

## Gotchas & Tips

⚠️ **Use -print0 with xargs** — Handles files with spaces and newlines

⚠️ **Be careful with -delete** — No undo! Use -print first to verify

⚠️ **Time is in 24-hour blocks** — `-mtime 1` means 24-48 hours, not exactly 1 day

💡 **Use -printf for performance** — Faster than piping to ls or other commands

💡 **Escape special characters** — Use `{}` for filename, `\;` for -exec end

💡 **Combine -print0 with xargs** — Perfect for handling any filename
