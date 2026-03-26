# 🐧 Disk Quotas & inotify — Linux Mastery

> **Disk quotas prevent a single user or process from filling a filesystem; inotify lets your tooling react to filesystem events the instant they happen — together they are the foundation of reliable multi-tenant and event-driven infrastructure.**

## 📖 Concept

### Disk Quotas

Disk quotas enforce per-user and per-group limits on how much disk space and how many inodes can be consumed on a filesystem. Without quotas, one misbehaving process — a runaway logger, a developer dumping core files, a misconfigured upload handler — can exhaust `/var` or `/home` and bring down every service on the host. In multi-tenant environments (shared Jenkins agents, data science workstations, EFS volumes shared across EC2 instances), quotas are the primary guardrail.

Linux supports two quota implementations: the traditional `quota` tools (stored in `aquota.user` and `aquota.group` files) and, for XFS, a first-class kernel-native quota system managed via `xfs_quota`. Most production AWS EBS volumes use ext4 or XFS; the latter's quota system is more robust and doesn't require a separate quota check step.

Each quota has two limits: the *soft limit* (a warning threshold the user can exceed temporarily) and the *hard limit* (an absolute ceiling the kernel enforces). A *grace period* (typically 7 days by default) is how long a user can exceed the soft limit before it becomes enforced as a hard limit.

### inotify

`inotify` is the Linux kernel's filesystem event notification subsystem. When a process creates, modifies, deletes, or renames a file, the kernel records the event in an inotify instance. User-space tools subscribe to these events via file descriptors — making it highly efficient (no polling). This is how `rsync --inotify`, Kubernetes config-map hot-reloading, `entr`, `watchdog` (Python), `inotifywait`, and many log shippers like Filebeat work under the hood.

In AWS, inotify is commonly used on EC2 instances to trigger automation when files land in a watched directory (e.g., auto-process CSV files dropped to `/data/incoming`), reload configs without restarting services, and implement fast cache invalidation.

---

## 💡 Real-World Use Cases

- Preventing `/var/log` from being filled by a single noisy application on a shared instance
- Enforcing per-team disk limits on an EFS mount shared by multiple teams' CI jobs
- Triggering an S3 upload script the moment a file lands in a local staging directory (inotify + aws s3 cp)
- Automatically reloading an nginx config after a certificate renewal (inotify watching `/etc/ssl/`)
- Detecting unauthorised file changes to `/etc/passwd` or `/etc/sudoers` in real time

---

## 🔧 Commands & Examples

### Setting Up Disk Quotas (ext4)

```bash
# Step 1: Enable quota on the filesystem (add usrquota,grpquota to /etc/fstab)
# Before:
# /dev/xvdb  /data  ext4  defaults  0 2
# After:
# /dev/xvdb  /data  ext4  defaults,usrquota,grpquota  0 2

# Remount with quota options
mount -o remount,usrquota,grpquota /data

# Step 2: Create quota database files
quotacheck -cugm /data
# -c: create quota files if missing
# -u: user quotas
# -g: group quotas
# -m: don't remount read-only during check

ls /data/aquota.user /data/aquota.group

# Step 3: Enable quotas
quotaon -vug /data
# /dev/xvdb [/data]: user quotas turned on
# /dev/xvdb [/data]: group quotas turned on

# Step 4: Set quota for a user
edquota -u naveen
# Opens an editor with:
# Disk quotas for user naveen (uid 1001):
#   Filesystem  blocks  soft  hard  inodes  soft  hard
#   /dev/xvdb   12345   0     0     234     0     0
#
# Set soft=4G hard=5G (in KB):
# /dev/xvdb   12345  4194304  5242880  234  0  0

# Non-interactive quota setting
setquota -u naveen 4194304 5242880 0 0 /data
# setquota -u <user> <soft-blocks-KB> <hard-blocks-KB> <soft-inodes> <hard-inodes> <filesystem>

# Set grace period to 3 days
edquota -t
# Grace period before enforcing soft limits for users:
# Time units may be: days, hours, minutes, or seconds
# Filesystem  Block grace period  Inode grace period
# /dev/xvdb   3days               7days
```

### Setting Up Disk Quotas (XFS)

```bash
# XFS quota is enabled at mount time — add uquota,gquota to /etc/fstab
# /dev/xvdc  /data  xfs  defaults,uquota,gquota  0 2
mount -o remount,uquota,gquota /data

# XFS quotas: use xfs_quota
xfs_quota -x -c 'help'

# Report current usage
xfs_quota -x -c 'report -uh' /data
# User quota on /data
# Filesystem  Blocks  Quota  Limit  Warn  Grace
# naveen      2048    0      0      00    [------]

# Set limits for user
xfs_quota -x -c 'limit bsoft=4g bhard=5g naveen' /data

# Set limits for group
xfs_quota -x -c 'limit -g bsoft=20g bhard=25g developers' /data

# Project quotas (per-directory limits — great for multi-tenant)
# Define project in /etc/projects and /etc/projid
echo "1:/data/team-alpha" >> /etc/projects
echo "team-alpha:1" >> /etc/projid

xfs_quota -x -c 'project -s team-alpha' /data
xfs_quota -x -c 'limit -p bsoft=10g bhard=12g team-alpha' /data

# Check project quota
xfs_quota -x -c 'report -ph' /data
```

### Inspecting and Reporting Quotas

```bash
# Show quotas for all users on a filesystem
repquota -a
repquota -aus   # human-readable sizes

# Show quota for a specific user
quota -u naveen
quota -us naveen   # human-readable

# Show quota for current user
quota

# Check filesystem quota status
quotastats

# Example output of repquota:
# User            used    soft    hard  grace    used  soft  hard  grace
# root      --    156764       0       0              4     0     0
# naveen    +-  4050123 4194304 5242880  6days  12345     0     0
# The +- means: over soft limit but under hard limit, grace period active
```

### Using inotifywait

```bash
# Install inotify-tools
apt-get install inotify-tools   # Debian/Ubuntu
yum install inotify-tools       # RHEL/Amazon Linux

# Watch a directory for any event (one-shot)
inotifywait /etc/nginx/

# Watch recursively for specific events
inotifywait -r -e create,modify,delete /etc/nginx/

# Loop mode: keep watching and print each event
inotifywait -m -r -e create,modify,delete --format '%T %w %f %e' \
  --timefmt '%Y-%m-%d %H:%M:%S' /etc/nginx/

# Available events:
# access        - file read
# modify        - file written
# attrib        - metadata changed (chmod, chown, timestamps)
# close_write   - file opened for writing, then closed (complete write)
# close_nowrite - file opened read-only, then closed
# open          - file opened
# moved_from    - file moved out of watched dir
# moved_to      - file moved into watched dir
# create        - file/dir created
# delete        - file/dir deleted
# delete_self   - watched file itself deleted
# move_self     - watched file itself moved
```

### Practical inotify Automation Patterns

```bash
# Pattern 1: Auto-upload files to S3 when they land in a directory
#!/bin/bash
WATCH_DIR="/data/uploads"
S3_BUCKET="s3://my-bucket/uploads"

inotifywait -m -e close_write --format '%w%f' "$WATCH_DIR" | while read FILE; do
  echo "$(date): Uploading $FILE to $S3_BUCKET"
  aws s3 cp "$FILE" "$S3_BUCKET/$(basename $FILE)"
  echo "$(date): Upload complete, removing $FILE"
  rm "$FILE"
done

# Pattern 2: Reload nginx on config change
inotifywait -m -e close_write /etc/nginx/ | while read path event file; do
  if [[ "$file" =~ \.conf$ ]]; then
    echo "Config changed: $file — testing and reloading nginx"
    nginx -t && systemctl reload nginx
  fi
done

# Pattern 3: Alert on sensitive file modification
inotifywait -m -e attrib,modify,moved_to /etc/passwd /etc/sudoers /etc/ssh/sshd_config | \
while read path event file; do
  MSG="SECURITY ALERT: $event on ${path}${file} at $(date)"
  echo "$MSG" | mail -s "Filesystem Integrity Alert" ops@company.com
  logger -t SECURITY "$MSG"
done

# Pattern 4: Watch for new log files in a directory
inotifywait -m -e create /var/log/app/ | while read path event file; do
  echo "New log file: $file"
  # Tail the new file and ship to your log aggregator
  tail -F "${path}${file}" | logger -t "app-log-$file" &
done
```

### inotifywait in Production Services

```bash
# Systemd service that watches for config changes
cat << 'EOF' > /etc/systemd/system/config-watcher.service
[Unit]
Description=Config File Watcher
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/config-watcher.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# /usr/local/bin/config-watcher.sh
cat << 'EOF' > /usr/local/bin/config-watcher.sh
#!/bin/bash
set -euo pipefail

WATCH_DIR="/etc/myapp"
log() { logger -t config-watcher "$*"; }

log "Starting config watcher on $WATCH_DIR"
inotifywait -m -e close_write,moved_to "$WATCH_DIR" \
  --format '%w%f' 2>/dev/null | while read CHANGED_FILE; do
  log "Detected change: $CHANGED_FILE"
  if systemctl is-active --quiet myapp; then
    log "Sending SIGHUP to myapp for hot reload"
    systemctl kill -s HUP myapp
  fi
done
EOF

chmod +x /usr/local/bin/config-watcher.sh
systemctl enable --now config-watcher
```

### inotify Limits and Tuning

```bash
# View current inotify limits
cat /proc/sys/fs/inotify/max_user_watches    # default: 8192
cat /proc/sys/fs/inotify/max_user_instances  # default: 128
cat /proc/sys/fs/inotify/max_queued_events   # default: 16384

# Increase for large-scale monitoring (e.g., Filebeat, Prometheus node-exporter)
sysctl fs.inotify.max_user_watches=524288
sysctl fs.inotify.max_user_instances=512

# Persist across reboots
cat >> /etc/sysctl.d/99-inotify.conf << 'EOF'
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
fs.inotify.max_queued_events = 65536
EOF
sysctl --system

# Diagnose "inotify watch limit reached" errors
# Usually appears as: "Failed to add inotify watch... Too many open files"
# Find which process is consuming the most watches:
find /proc/*/fd -lname 'anon_inode:inotify' 2>/dev/null | \
  awk -F/ '{print $3}' | \
  xargs -I{} sh -c 'echo -n "{}: "; ls /proc/{}/fd | wc -l' | \
  sort -t: -k2 -rn | head -20
```

---

## ⚠️ Gotchas & Pro Tips

- **Quota soft limits need a grace period to matter:** If you set a soft limit but never set a grace period, the soft limit is effectively useless — users can exceed it indefinitely. Always configure grace with `edquota -t` or `xfs_quota -x -c 'timer ...'`.

- **XFS project quotas are better for multi-tenant:** Unlike user/group quotas, project quotas let you limit disk usage *per directory tree*, regardless of which user writes to it. This is ideal for CI workspaces, application data directories, and EFS path-based tenancy.

- **`close_write` not `modify` for complete files:** When watching for new files to process, use the `close_write` event rather than `modify`. `modify` fires on every write syscall — potentially hundreds of times per file. `close_write` fires once when the writing process closes the file descriptor, meaning the file is complete.

- **inotify doesn't work on NFS/FUSE/EFS (by default):** inotify events only fire for local filesystem operations. NFS mounts (including EFS) require `fanotify` or polling instead. AWS's EFS uses NFS under the hood, so inotify won't catch remote writes. Use `fswatch` with its polling backend for cross-host scenarios.

- **inotify watch count is per-filesystem-object, not per-directory:** Watching a directory recursively with `inotifywait -r` adds one watch per directory. A tree with 10,000 subdirectories uses 10,000 watches. Plan your `max_user_watches` accordingly. On K8s nodes running many pods with Filebeat, exhausting inotify watches is a common operational issue.

- **Quota database must be kept consistent:** On unclean shutdowns, `aquota.user` can become stale. Add `quotacheck` to your startup scripts or boot sequence, or rely on XFS's kernel-native quotas which don't have this problem.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
