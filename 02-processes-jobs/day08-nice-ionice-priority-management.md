# 🐧 nice, ionice & Process Priority Management — Linux Mastery

> **Control how much CPU and I/O your processes get — essential for keeping production systems responsive under load.**

## 📖 Concept

Linux uses two separate priority systems for processes: **CPU scheduling priority** controlled by `nice`/`renice`, and **I/O scheduling priority** controlled by `ionice`. Understanding both is critical for production operations — misconfigured priorities cause noisy-neighbor problems on shared hosts, slow backup jobs that tank production databases, and batch workloads that starve interactive services.

**CPU Priority (niceness):** Every process has a nice value from -20 (highest priority) to +19 (lowest priority). A lower nice value means the scheduler gives that process more CPU time relative to others. Only root can set negative nice values; regular users can only increase niceness (be "nicer" to other processes). The default nice value is 0.

**I/O Priority (ionice):** The Linux I/O scheduler has three classes: Real-time (class 1), Best-effort (class 2, default), and Idle (class 3). Within Best-effort and Real-time, there are 8 levels (0-7, lower = higher priority). Idle class processes only get I/O when no other process needs the disk — perfect for backups, indexing, and batch jobs that shouldn't affect production response times.

**Why this matters in production:** A rogue `rsync` backup, `find` crawl, or log compression job running at default priority can saturate your disk I/O and cause 100ms+ latency spikes on your database or application. Setting `ionice -c3` on batch jobs is one of the cheapest performance wins available.

---

## 💡 Real-World Use Cases

- Run nightly backups (`rsync`, `tar`) at idle I/O priority so they don't impact production DB IOPS
- Deprioritize `apt upgrade`, `yum update`, or `pip install` on production servers that can't be taken offline
- Run `baselayout` or `aide` file integrity checks at low priority during business hours
- Give Kubernetes system daemons (`kubelet`, `etcd`) higher CPU priority than workload pods
- Renice a runaway compile job that's starving other processes without killing it

---

## 🔧 Commands & Examples

### nice — Start a Process with a Specific Priority

```bash
# start a process with nice value +10 (lower priority than default)
nice -n 10 tar -czf /backup/app.tar.gz /var/app/

# start a backup at the lowest possible CPU priority
nice -n 19 rsync -av /data/ /backup/data/

# start a process with higher priority (requires root)
sudo nice -n -10 /usr/sbin/nginx

# run a Python data processing job at low priority
nice -n 15 python3 process_data.py

# combine with ionice for both CPU and I/O deprioritization
nice -n 19 ionice -c3 tar -czf /backup/db.tar.gz /var/lib/postgresql/

# run make with low priority to avoid starving other devs on a shared build server
nice -n 10 make -j$(nproc)

# check current nice value of a process
ps -o pid,ni,comm -p $$
```

### renice — Change Priority of a Running Process

```bash
# lower the priority of a running PID
renice -n 10 -p 12345

# renice all processes of a specific user
renice -n 15 -u builduser

# renice all processes in a process group
renice -n 5 -g 5678

# renice a running backup that's causing load
BACKUP_PID=$(pgrep rsync)
renice -n 19 -p $BACKUP_PID

# renice multiple PIDs at once
renice -n 10 -p $(pgrep -d' ' python3)

# check nice values of all processes
ps -eo pid,ni,comm --sort=ni | head -20

# find processes with negative nice (high priority)
ps -eo pid,ni,comm | awk '$2 < 0'
```

### ionice — Control I/O Scheduling Class

```bash
# run rsync at idle I/O priority (class 3 — only gets I/O when disk is otherwise idle)
ionice -c3 rsync -av /data/ /backup/

# run a backup with idle I/O priority
ionice -c3 tar -czf /backup/full.tar.gz /var/data/

# best-effort class with level 7 (lowest within BE class)
ionice -c2 -n7 find / -name "*.log" -mtime +30

# real-time class with level 0 (highest — use with extreme caution)
# this will starve all other I/O processes
sudo ionice -c1 -n0 dd if=/dev/sda of=/dev/sdb bs=64M

# check ionice class of a running process
ionice -p $(pgrep mysqld)
# output: best-effort: prio 4

# change ionice of a running process
ionice -c3 -p $(pgrep rsync)

# run a database import at idle priority
ionice -c3 mysql -u root mydb < dump.sql

# schedule log rotation at low I/O priority
ionice -c2 -n7 logrotate /etc/logrotate.conf
```

### Combining nice + ionice in Production Scripts

```bash
#!/bin/bash
# Production-safe backup script — won't impact production I/O or CPU

set -euo pipefail

SOURCE_DIR="/var/lib/postgresql/14/main"
BACKUP_DIR="/mnt/backups/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/pg_backup_${DATE}.tar.gz"

echo "[$(date)] Starting low-priority backup..."

# Run at lowest CPU and I/O priority
nice -n 19 ionice -c3 tar \
  --create \
  --gzip \
  --file="${BACKUP_FILE}" \
  --exclude="${SOURCE_DIR}/pg_wal" \
  "${SOURCE_DIR}"

echo "[$(date)] Backup completed: ${BACKUP_FILE}"
echo "[$(date)] Size: $(du -sh ${BACKUP_FILE} | cut -f1)"
```

```bash
#!/bin/bash
# Renice all batch/non-critical processes automatically

# Renice known batch jobs
for proc in rsync tar gzip find updatedb mlocate aide; do
    pgrep -x "$proc" | while read pid; do
        current_nice=$(ps -o ni= -p $pid 2>/dev/null | tr -d ' ')
        if [ -n "$current_nice" ] && [ "$current_nice" -lt 10 ]; then
            echo "Renicing $proc (PID $pid) from $current_nice to 15"
            renice -n 15 -p $pid 2>/dev/null || true
            ionice -c3 -p $pid 2>/dev/null || true
        fi
    done
done
```

### Checking and Monitoring Priorities

```bash
# show all processes with nice and ionice info
ps -eo pid,ppid,ni,cls,pri,comm --sort=-ni | head -30

# show CPU time and priority for processes
top -b -n1 | awk 'NR>7 {print $1, $8, $9, $12}' | head -20
# columns: PID, NI, %CPU, COMMAND

# real-time monitoring with nice values visible
htop  # press F6 to sort by NICE, or press 'n' to highlight nice values

# check I/O wait and see which processes are causing it
iotop -o -P  # -o shows only processes doing I/O, -P shows processes not threads

# see I/O stats per process
pidstat -d 1 5  # I/O stats every 1 second for 5 intervals

# find the process consuming most I/O
iotop -b -n5 -o | sort -k10 -rn | head -10

# check scheduler class of a process (RT, SCHED_FIFO, etc.)
chrt -p $(pgrep nginx | head -1)

# set real-time scheduling for a latency-sensitive process (requires root)
sudo chrt -f -p 50 $(pgrep etcd | head -1)
```

### AWS / Cloud Context

```bash
# On EC2 instances with burstable performance (t3/t4g), CPU priority is critical
# because the instance shares physical CPU credits with other VMs

# Run heavy batch jobs at low priority on shared EC2 instances
nice -n 19 ionice -c3 aws s3 sync /var/log/ s3://my-bucket/logs/

# Use ionice for EBS-backed instances — EBS has IOPS limits
# A low-priority rogue process can exhaust your provisioned IOPS
ionice -c3 -p $(pgrep backup_agent)

# CloudWatch agent runs at default priority — renice if it's causing CPU spikes
sudo renice -n 5 -p $(pgrep amazon-cloudwatch)

# On EKS nodes, kubelet and containerd should have higher priority
# Check their current priority
ps -eo pid,ni,comm | grep -E 'kubelet|containerd|dockerd'
```

### systemd Service Priority Configuration

```bash
# Set CPU and I/O priority for a systemd service unit
# /etc/systemd/system/backup.service

cat <<EOF | sudo tee /etc/systemd/system/backup.service
[Unit]
Description=Nightly Backup Service
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
Nice=15
IOSchedulingClass=idle
CPUSchedulingPolicy=batch
EOF

# Apply and check
sudo systemctl daemon-reload
sudo systemctl start backup.service

# Verify nice and IO class were applied
systemctl show backup.service | grep -E 'Nice|IOScheduling'
# Nice=15
# IOSchedulingClass=idle

# CPUQuota in systemd (cgroup-based, more precise than nice)
# Limit a service to 20% of one CPU core
# CPUQuota=20%
# MemoryLimit=512M

# Real example — prevent log rotation from impacting prod
sudo systemctl edit logrotate.service --force <<EOF
[Service]
Nice=15
IOSchedulingClass=idle
EOF
```

---

## ⚠️ Gotchas & Pro Tips

- **nice doesn't guarantee CPU time:** On a lightly loaded system, a nice +19 process can run at near full speed. nice only matters under CPU contention — it's a relative weight, not an absolute cap. Use `cgroups CPUQuota` if you need hard limits.

- **ionice class 1 (Real-time) will starve everything else:** Never use `ionice -c1` without understanding its impact. On production, it can prevent the OS from paging, writing to disk logs, or doing any other I/O. Idle class (`-c3`) is almost always what you want for batch work.

- **Containers and nice:** In Docker/K8s, `nice` works within a container but the effect is relative to processes in the same cgroup. The container's cgroup CPU shares setting (`--cpu-shares`) has more impact at the host level.

- **`updatedb` runs nightly and murders I/O:** The `mlocate` `updatedb` cron job crawls the entire filesystem at default I/O priority. Check `/etc/updatedb.conf` and verify it's running with `ionice -c3` in `/etc/cron.daily/mlocate`.

- **renice requires the same user or root:** A process owner can only renice their own processes to a higher nice value (less priority). Only root can increase priority (lower nice value) or renice other users' processes.

- **`ionice` requires the CFQ I/O scheduler:** `ionice` only works when the block device uses the CFQ (Completely Fair Queuing) or BFQ scheduler. Modern NVMe drives and cloud VMs often use `mq-deadline` or `none` — check with `cat /sys/block/nvme0n1/queue/scheduler`. On these, `ionice` is a no-op and you need cgroup `blkio` weight instead.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
