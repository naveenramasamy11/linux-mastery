# 🐧 LVM Disk Management — Linux Mastery

> **LVM is the abstraction layer between your physical disks and your filesystems — master it and you can resize, snapshot, and migrate storage without downtime.**

## 📖 Concept

LVM (Logical Volume Manager) adds a powerful abstraction layer between raw block devices and the filesystems mounted on top of them. It operates in three tiers: Physical Volumes (PVs) — the actual block devices or partitions; Volume Groups (VGs) — pools that aggregate multiple PVs; and Logical Volumes (LVs) — the virtual block devices carved from a VG that your OS actually formats and mounts.

The killer features that make LVM indispensable in production: online resizing (extend an LV and its filesystem without unmounting), snapshots (point-in-time consistent copies for backups or testing), thin provisioning (allocate more space than physically exists, growing on demand), and striping/mirroring across physical devices. On AWS, LVM is how you combine multiple EBS volumes into a single logical disk, or how you add capacity to an EC2 instance's root volume without stopping it.

LVM's device mapper backend (`/dev/mapper/`) is also what powers LUKS encrypted volumes and Docker's devicemapper storage driver. The same `dm-*` layer that LVM uses appears in container runtimes, so understanding LVM makes the entire Linux storage stack click into place.

A Volume Group is essentially a storage pool. You add PVs to it to grow the pool, and you carve LVs out of it for filesystems. The VG tracks allocation in **Physical Extents (PEs)** — typically 4MB chunks. An LV is just a contiguous (or striped) range of PEs mapped to a dm device.

---

## 💡 Real-World Use Cases

- Add a new EBS volume to an EC2 instance and extend `/var/lib/elasticsearch` online without any service restart
- Create a pre-backup LVM snapshot of a PostgreSQL data directory, take a consistent backup, then remove the snapshot
- Combine four 500GB EBS gp3 volumes into a single 2TB VG and create a striped LV for improved IOPS
- Thin-provision a dev environment: give developers 500GB logical volumes backed by 100GB of actual storage
- Migrate a PV from one disk to another without downtime using `pvmove`

---

## 🔧 Commands & Examples

### Physical Volume (PV) Operations

```bash
# Initialize a disk/partition as a PV
pvcreate /dev/xvdf
pvcreate /dev/xvdg /dev/xvdh    # multiple at once

# List PVs (brief)
pvs

# Sample pvs output:
#   PV         VG       Fmt  Attr PSize   PFree
#   /dev/xvdf  data_vg  lvm2 a--  500.00g 200.00g
#   /dev/xvdg  data_vg  lvm2 a--  500.00g 500.00g

# Detailed PV info
pvdisplay /dev/xvdf

# PV attributes including UUID, PE size, total/free extents
pvdisplay -m /dev/xvdf    # show physical extent map

# Remove PV label (only if not in a VG)
pvremove /dev/xvdf

# Scan for all PVs on the system
pvscan
```

### Volume Group (VG) Operations

```bash
# Create a VG named "data_vg" from one PV
vgcreate data_vg /dev/xvdf

# Create with custom PE size (default is 4MB, larger PEs = bigger max LV)
vgcreate -s 16M data_vg /dev/xvdf

# Create from multiple PVs (combines them into one pool)
vgcreate data_vg /dev/xvdf /dev/xvdg /dev/xvdh

# List VGs (brief)
vgs

# Detailed VG info
vgdisplay data_vg

# Add a PV to an existing VG (online — no downtime)
vgextend data_vg /dev/xvdi

# Remove a PV from a VG (must move data off first with pvmove)
pvmove /dev/xvdf            # move all data off /dev/xvdf within the VG
vgreduce data_vg /dev/xvdf  # then remove it from the VG
pvremove /dev/xvdf           # clean up PV label

# Rename a VG
vgrename data_vg storage_vg

# Deactivate/activate a VG
vgchange -an data_vg   # deactivate (unmount LVs first)
vgchange -ay data_vg   # reactivate

# Remove a VG entirely (must remove all LVs first)
vgremove data_vg
```

### Logical Volume (LV) Operations

```bash
# Create a 100GB LV named "app_data" in VG "data_vg"
lvcreate -L 100G -n app_data data_vg

# Create using percentage of VG free space
lvcreate -l 50%FREE -n app_data data_vg

# Create using all remaining free space
lvcreate -l 100%FREE -n app_data data_vg

# Create a striped LV across 2 PVs (better IOPS — like RAID 0)
lvcreate -L 200G -n stripe_vol -i 2 -I 64 data_vg

# List LVs (brief)
lvs

# Sample lvs output:
#   LV       VG       Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
#   app_data data_vg  -wi-ao---- 100.00g

# Detailed LV info
lvdisplay /dev/data_vg/app_data

# Show LV segments (stripes, mirrors, thin pools)
lvdisplay -m /dev/data_vg/app_data

# LV device paths (two equivalent symlinks):
# /dev/data_vg/app_data
# /dev/mapper/data_vg-app_data

# Format the LV
mkfs.xfs /dev/data_vg/app_data
# or
mkfs.ext4 /dev/data_vg/app_data

# Mount it
mkdir -p /data/app
mount /dev/data_vg/app_data /data/app

# Persistent mount in /etc/fstab
echo '/dev/data_vg/app_data /data/app xfs defaults 0 0' >> /etc/fstab

# Deactivate an LV (before removal)
lvchange -an /dev/data_vg/app_data

# Remove an LV
lvremove /dev/data_vg/app_data
```

### Online LV Resize (The Key Production Skill)

The most common LVM task in production: add more disk and extend a filesystem **without downtime**.

```bash
# --- SCENARIO: /dev/xvdf is full, attach new EBS volume /dev/xvdg ---

# Step 1: Initialize new disk as PV
pvcreate /dev/xvdg

# Step 2: Add to existing VG
vgextend data_vg /dev/xvdg

# Step 3: Extend the LV
# Option A: add a fixed amount
lvextend -L +200G /dev/data_vg/app_data

# Option B: use all remaining free space
lvextend -l +100%FREE /dev/data_vg/app_data

# Option C: extend AND resize the filesystem in one command (-r flag)
lvextend -L +200G -r /dev/data_vg/app_data

# Step 4a: Resize ext4 filesystem (if not using -r)
resize2fs /dev/data_vg/app_data

# Step 4b: Resize XFS filesystem (if not using -r)
xfs_growfs /data/app    # XFS needs the mount point, not the device

# Verify
df -h /data/app
lvs

# --- SCENARIO: Reduce an LV (ext4 only — XFS cannot shrink) ---
# WARNING: Always backup first! Shrink is risky.

# Step 1: Unmount
umount /data/app

# Step 2: Check filesystem integrity
e2fsck -f /dev/data_vg/app_data

# Step 3: Shrink filesystem first (to new target size)
resize2fs /dev/data_vg/app_data 50G

# Step 4: Then shrink the LV
lvreduce -L 50G /dev/data_vg/app_data

# Step 5: Remount
mount /dev/data_vg/app_data /data/app
```

### LVM Snapshots

Snapshots are COW (copy-on-write) point-in-time images of an LV. Perfect for pre-backup consistency or safe testing.

```bash
# Create a snapshot of app_data (10GB COW buffer)
lvcreate -L 10G -s -n app_data_snap /dev/data_vg/app_data

# The snapshot device is: /dev/data_vg/app_data_snap
# Mount it read-only for backup
mkdir -p /mnt/snap
mount -o ro /dev/data_vg/app_data_snap /mnt/snap

# Take a backup from the snapshot
tar czf /backup/app_data_$(date +%Y%m%d).tar.gz -C /mnt/snap .

# Or use rsync
rsync -avz /mnt/snap/ user@backup-server:/backups/app_data/

# Unmount and remove the snapshot when done
umount /mnt/snap
lvremove /dev/data_vg/app_data_snap

# Monitor snapshot usage (COW buffer must not fill up — if it does, snapshot is invalidated)
lvs -o lv_name,lv_size,data_percent,snap_percent /dev/data_vg/app_data_snap

# For PostgreSQL consistent snapshot: use pg_start_backup / pg_stop_backup wrapper
psql -c "SELECT pg_start_backup('lvm_snap', true);"
lvcreate -L 10G -s -n pg_snap /dev/data_vg/pg_data
psql -c "SELECT pg_stop_backup();"
```

### Thin Provisioning

```bash
# Create a thin pool (metadata LV + data LV managed together)
lvcreate -L 400G --thinpool thin_pool data_vg

# Create thin volumes from the pool (over-provision up to 2TB from 400GB)
lvcreate -V 500G --thin -n vm_disk1 data_vg/thin_pool
lvcreate -V 500G --thin -n vm_disk2 data_vg/thin_pool
lvcreate -V 500G --thin -n vm_disk3 data_vg/thin_pool
lvcreate -V 500G --thin -n vm_disk4 data_vg/thin_pool

# Monitor actual usage
lvs -o lv_name,lv_size,data_percent data_vg

# Create thin snapshot (very space-efficient — only tracks changes)
lvcreate -s -n vm_disk1_snap /dev/data_vg/vm_disk1
```

### pvmove — Live PV Migration

```bash
# Evacuate all data from /dev/xvdf to other PVs in the VG (online — no downtime)
pvmove /dev/xvdf

# Run in background and monitor
pvmove --background /dev/xvdf
lvmpolld    # LVM daemon that handles background operations

# Monitor progress
pvs
watch -n2 pvs

# Move only a specific LV's extents off a PV
pvmove -n app_data /dev/xvdf
```

### Useful Diagnostic Commands

```bash
# Full LVM report in one shot
vgdisplay -v data_vg

# Show device mapper topology
dmsetup ls --tree
dmsetup info /dev/mapper/data_vg-app_data
dmsetup status /dev/mapper/data_vg-app_data

# Check LVM metadata backup location
ls /etc/lvm/backup/
ls /etc/lvm/archive/

# Manually backup VG metadata
vgcfgbackup data_vg -f /backup/data_vg_metadata.txt

# Restore VG metadata (disaster recovery)
vgcfgrestore data_vg -f /backup/data_vg_metadata.txt

# Scan for all LVM devices
lvmdiskscan

# Full inventory in JSON (great for automation)
pvs --reportformat json
vgs --reportformat json
lvs --reportformat json
```

---

## ⚠️ Gotchas & Pro Tips

- **Never shrink XFS:** XFS does not support shrinking — only growing. If you need to shrink an XFS filesystem, you must backup, recreate, and restore. Use ext4 if you need bi-directional resizing.
- **Snapshot COW buffer:** If the snapshot's COW buffer fills up completely (you wrote more data to the origin than the buffer can track), the snapshot is marked invalid and automatically deactivated. Size the snapshot buffer conservatively at ~20% of the origin LV size, or use thin snapshots which don't have this limit.
- **Resize LV before or after filesystem?:** When growing: extend LV first, then filesystem. When shrinking: shrink filesystem first, then shrink LV. Getting this backwards means data corruption.
- **`-r` flag is your friend:** `lvextend -r` calls the appropriate `resize2fs` or `xfs_growfs` automatically. Use it — it's safer than doing it manually.
- **PE size and max LV size:** Default PE size is 4MB, giving a max LV of 255 * 4MB * (max extents) ≈ 16TB per LV. If you need larger, increase PE size when creating the VG (`vgcreate -s 32M`).
- **AWS EC2 + LVM:** After extending an EBS volume in the AWS console, you still need to `growpart` the partition (if any), then `pvresize /dev/xvdf`, then `lvextend`, then `resize2fs/xfs_growfs`. Each layer must be extended in sequence.

```bash
# Full AWS EBS online expand sequence
# 1. Expand EBS volume in AWS Console to new size (e.g., 500GB → 700GB)
# 2. On the EC2 instance:
lsblk                           # confirm new size is visible
sudo growpart /dev/xvdf 1       # resize partition if there is one
sudo pvresize /dev/xvdf         # tell LVM about the new PV size
sudo lvextend -l +100%FREE -r /dev/data_vg/app_data  # extend LV + filesystem
df -h /data/app                 # confirm
```

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
