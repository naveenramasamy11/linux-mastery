# 🐧 mdadm Software RAID Management — Linux Mastery

> **Build, monitor, and recover Linux software RAID arrays — the tool that keeps data alive when hardware fails.**

## 📖 Concept

`mdadm` (Multiple Device Admin) is Linux's software RAID tool, allowing you to combine multiple block devices into a single redundant array at the kernel level. While modern cloud environments (EC2 instance store, EBS RAID) and storage appliances often abstract this away, understanding mdadm is essential for bare-metal infrastructure, VMware datastores with multiple disks, on-premises Kubernetes nodes, and any situation where you need high-throughput or redundancy without a hardware RAID controller.

**RAID Levels you'll actually use:**
- **RAID 0 (Striping):** Splits data across all drives for maximum performance and capacity. Zero redundancy — one drive failure loses everything. Use for temp/scratch storage on EKS worker nodes or CI build caches.
- **RAID 1 (Mirroring):** Exact copy on two drives. Read performance can be doubled. Write speed limited by slowest drive. Use for OS disks, `/var/lib/etcd`, critical config.
- **RAID 5 (Distributed Parity):** Requires 3+ drives. Tolerates 1 drive failure. Capacity = (n-1) drives. Good balance of performance, capacity, and redundancy.
- **RAID 6 (Double Parity):** Like RAID 5 but tolerates 2 simultaneous failures. Requires 4+ drives. Use for large storage pools where rebuild time creates long vulnerability windows.
- **RAID 10 (Stripe of Mirrors):** Requires 4+ drives (even number). RAID 1+0 — striped across mirrored pairs. Best performance + redundancy. Use for high-IOPS database disks.

**The mdadm workflow** involves creating the array, adding a filesystem, mounting it, making it persistent via `/etc/fstab`, and configuring monitoring so you get alerted on failures before they cascade.

---

## 💡 Real-World Use Cases

- Build a RAID 10 array from 4 NVMe drives on a bare-metal Kubernetes node for high-IOPS persistent volumes
- Create RAID 1 for the OS disk on physical servers to survive a single disk failure without downtime
- Set up RAID 5 for a large data archive on a cost-optimized on-prem server
- Rebuild a degraded array after replacing a failed drive in a production database server
- Monitor RAID health via email alerts and integrate with Ansible/Terraform provisioning

---

## 🔧 Commands & Examples

### Creating RAID Arrays

```bash
# Check available disks before creating
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
fdisk -l /dev/sdb /dev/sdc /dev/sdd /dev/sde

# Zero superblock from any previous RAID (important before reuse)
mdadm --zero-superblock /dev/sdb /dev/sdc

# Create RAID 1 (mirror) — 2 drives
mdadm --create /dev/md0 \
  --level=1 \
  --raid-devices=2 \
  /dev/sdb /dev/sdc

# Create RAID 5 — 3 drives
mdadm --create /dev/md0 \
  --level=5 \
  --raid-devices=3 \
  /dev/sdb /dev/sdc /dev/sdd

# Create RAID 5 with a hot spare (auto-replaces a failed drive)
mdadm --create /dev/md0 \
  --level=5 \
  --raid-devices=3 \
  --spare-devices=1 \
  /dev/sdb /dev/sdc /dev/sdd /dev/sde

# Create RAID 10 — 4 drives (best for databases)
mdadm --create /dev/md0 \
  --level=10 \
  --raid-devices=4 \
  /dev/sdb /dev/sdc /dev/sdd /dev/sde

# Create RAID 0 — 2 drives (striping, no redundancy)
mdadm --create /dev/md0 \
  --level=0 \
  --raid-devices=2 \
  /dev/sdb /dev/sdc

# Watch sync progress (wait for initial sync before using in production)
watch -n2 cat /proc/mdstat
# md0 : active raid5 sdd[2] sdc[1] sdb[0]
#       2095104 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
#       [====>................]  resync = 22.4% (234112/1047552) finish=2.3min speed=5914K/sec
```

### Format, Mount, and Make Persistent

```bash
# Create a filesystem on the RAID device
mkfs.xfs /dev/md0
# or: mkfs.ext4 /dev/md0

# Create mount point
mkdir -p /data

# Mount
mount /dev/md0 /data

# Make mount persistent — use UUID (not device name, which can change)
UUID=$(blkid -s UUID -o value /dev/md0)
echo "UUID=${UUID} /data xfs defaults,nofail 0 0" >> /etc/fstab

# Verify fstab
mount -a
df -h /data

# Save mdadm configuration (required for auto-reassembly on boot)
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
# or
mdadm --detail --scan | tee -a /etc/mdadm/mdadm.conf

# Update initramfs so the array assembles at boot (Debian/Ubuntu)
update-initramfs -u

# Enable mdadm monitoring service
systemctl enable mdmonitor
systemctl start mdmonitor
```

### Monitoring RAID Health

```bash
# Quick status check
cat /proc/mdstat
# md0 : active raid5 sdc[2] sdb[1] sdd[0]
#       2095104 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
#       (U = working, _ = failed/missing)

# Detailed array info
mdadm --detail /dev/md0

# Example output:
# /dev/md0:
#            Version : 1.2
#      Creation Time : Sat Jun  6 05:00:00 2026
#         Raid Level : raid5
#         Array Size : 2095104 (2046.34 MiB 2145.39 MB)
#      Used Dev Size : 1047552 (1023.17 MiB 1072.69 MB)
#       Raid Devices : 3
#      Total Devices : 3
#    Persistence : Superblock is persistent
#        Update Time : Sat Jun  6 05:05:00 2026
#              State : clean
#     Active Devices : 3
#    Working Devices : 3
#     Failed Devices : 0
#      Spare Devices : 0

# Check all arrays at once
mdadm --detail --scan

# Monitor all arrays in real time
watch -n5 'mdadm --detail /dev/md0 | grep -E "State|Active|Failed|Rebuild"'

# Check individual drive status within array
mdadm --examine /dev/sdb

# Configure email alerts for failures
# Edit /etc/mdadm/mdadm.conf
echo "MAILADDR naveenramasamy11@gmail.com" >> /etc/mdadm/mdadm.conf
systemctl restart mdmonitor

# Test alert configuration
mdadm --monitor --test --oneshot /dev/md0

# Check mdadm event log
journalctl -u mdmonitor -n 50
```

### Simulating and Handling Drive Failures

```bash
# Simulate a drive failure (for testing — USE WITH CAUTION in production)
mdadm --manage /dev/md0 --fail /dev/sdb

# Check degraded state
cat /proc/mdstat
# md0 : active raid5 sdc[2] sdb[1](F) sdd[0]
#       2095104 blocks super 1.2 level 5 [3/2] [_UU]
#       [F = failed, _ = missing slot]

# Remove the failed drive from the array
mdadm --manage /dev/md0 --remove /dev/sdb

# Add replacement drive
mdadm --manage /dev/md0 --add /dev/sde

# Watch rebuild progress
watch -n2 cat /proc/mdstat
# md0 : active raid5 sde[3] sdc[2] sdd[0]
#       2095104 blocks super 1.2 level 5 [3/2] [_UU]
#       [=>...................]  recovery = 5.2% (55296/1047552) finish=3.2min

# Full replacement workflow on production
# 1. Identify failed disk
mdadm --detail /dev/md0 | grep "Failed\|removed"

# 2. Find which physical disk it is
ls -la /sys/block/sdb/  # check by-id path
ls /dev/disk/by-id/ | grep sdb

# 3. Mark failed and remove
mdadm --manage /dev/md0 --fail /dev/sdb
mdadm --manage /dev/md0 --remove /dev/sdb

# 4. (Physical disk swap happens here)

# 5. Zero superblock on new disk
mdadm --zero-superblock /dev/sdb

# 6. Add new disk to array
mdadm --manage /dev/md0 --add /dev/sdb

# 7. Confirm rebuild started
cat /proc/mdstat
```

### Growing and Reshaping Arrays

```bash
# Add a new drive to an existing RAID 5 (online growth)
# First add as spare
mdadm --manage /dev/md0 --add /dev/sde

# Grow the array from 3 to 4 devices
mdadm --grow /dev/md0 --raid-devices=4

# Watch reshape (can take hours for large arrays)
watch -n5 cat /proc/mdstat

# After reshape, grow the filesystem to use new space
xfs_growfs /data
# or for ext4:
resize2fs /dev/md0

# Verify new size
df -h /data

# Convert RAID 5 to RAID 6 (requires at least 4 drives)
# Add spare first
mdadm --manage /dev/md0 --add /dev/sdf
# Then reshape (online, but slow — schedule in maintenance window)
mdadm --grow /dev/md0 --level=6 --raid-devices=4
```

### Automation and Ansible Integration

```bash
# Full automated RAID setup script (idempotent)
#!/bin/bash
set -euo pipefail

DRIVES=(/dev/sdb /dev/sdc /dev/sdd /dev/sde)
ARRAY=/dev/md0
MOUNT=/data
LEVEL=10

# Zero superblocks
for drive in "${DRIVES[@]}"; do
    mdadm --zero-superblock "$drive" 2>/dev/null || true
done

# Create array
mdadm --create "$ARRAY" \
    --level="$LEVEL" \
    --raid-devices="${#DRIVES[@]}" \
    --run \
    "${DRIVES[@]}"

# Wait for sync to be in progress (enough to use)
sleep 5

# Create filesystem
mkfs.xfs "$ARRAY"

# Mount
mkdir -p "$MOUNT"
mount "$ARRAY" "$MOUNT"

# Persist
UUID=$(blkid -s UUID -o value "$ARRAY")
if ! grep -q "$UUID" /etc/fstab; then
    echo "UUID=${UUID} ${MOUNT} xfs defaults,nofail 0 0" >> /etc/fstab
fi

# Save config
mdadm --detail --scan | grep -v "^$" >> /etc/mdadm/mdadm.conf
update-initramfs -u 2>/dev/null || dracut -f 2>/dev/null || true

# Enable monitoring
echo "MAILADDR naveenramasamy11@gmail.com" >> /etc/mdadm/mdadm.conf
systemctl enable --now mdmonitor 2>/dev/null || true

echo "RAID${LEVEL} array ${ARRAY} created and mounted at ${MOUNT}"
df -h "$MOUNT"
```

```bash
# Ansible task snippet for RAID creation
# tasks/raid.yml
# - name: Zero superblocks
#   command: "mdadm --zero-superblock {{ item }}"
#   loop: "{{ raid_drives }}"
#   ignore_errors: true
#
# - name: Create RAID array
#   command: >
#     mdadm --create /dev/md0
#     --level={{ raid_level }}
#     --raid-devices={{ raid_drives | length }}
#     {{ raid_drives | join(' ') }}
#   args:
#     creates: /dev/md0
```

---

## ⚠️ Gotchas & Pro Tips

- **Never use RAID as a backup:** RAID protects against drive failure, not accidental deletion, filesystem corruption, or ransomware. RAID 1 faithfully mirrors `rm -rf /data/` to both drives instantly. Always back up separately.

- **RAID 5 write hole / rebuild risk:** During RAID 5 rebuild, if another drive fails, you lose everything. For large disks (8TB+), rebuild time can be 12-48 hours — a second failure during rebuild is statistically likely. Use RAID 6 or RAID 10 for critical data with large drives.

- **`nofail` in fstab is critical:** Without `nofail`, a degraded or missing RAID device at boot will drop the system into emergency mode and halt boot. Always use `nofail` for RAID-backed mounts.

- **Chunk size affects performance:** RAID 5/6 chunk size (default 512KB) should match your workload. For databases with 8KB random I/O, smaller chunks (64KB) are better. For streaming video or backups, larger chunks (1MB+) improve throughput.

- **`mdadm.conf` must be updated after changes:** Every time you create, add, or modify an array, re-run `mdadm --detail --scan >> /etc/mdadm/mdadm.conf` and `update-initramfs -u`. Forgetting this means the array won't assemble correctly after a reboot.

- **Monitor `/proc/mdstat` in your monitoring stack:** Add a check for `_` characters (degraded drives) in `/proc/mdstat` to Prometheus node_exporter, Nagios, or a simple cron alert script. Don't wait for email — RAID degradation often gets lost in spam.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
