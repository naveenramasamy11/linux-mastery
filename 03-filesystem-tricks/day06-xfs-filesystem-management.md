# 🐧 XFS Filesystem Management — Linux Mastery

> **XFS is the default filesystem on RHEL/CentOS/Amazon Linux 2023 and the go-to choice for high-throughput workloads — knowing how to manage, repair, and tune it is non-negotiable for production Linux work.**

## 📖 Concept

XFS is a high-performance 64-bit journaling filesystem originally developed by SGI. It uses **B+ trees** extensively — for directory entries, extent maps, and free space tracking — making it extremely efficient at large-scale random I/O. Unlike ext4 which uses fixed-size block groups, XFS uses **Allocation Groups (AGs)**, parallel independent structures that allow concurrent I/O from multiple CPU cores. This is why XFS scales beautifully on NVMe arrays and EBS `io2` volumes.

The XFS journal (also called the log) records metadata changes before committing them to disk. The log can be placed on a **separate device** (e.g., a fast NVMe) for dramatic performance improvement on write-heavy workloads. XFS supports **online defragmentation** (`xfs_fsr`), **online resizing** (`xfs_growfs` — grow only, shrink is not supported), and **online metadata checking** (`xfs_scrub`).

One critical XFS characteristic: **you cannot shrink an XFS filesystem**. This is by design — the B+ tree structures make online shrink impractical. If you need to shrink, you must backup, reformat, and restore. Plan your LV sizes accordingly (another reason LVM thin provisioning pairs well with XFS).

In AWS, XFS on `gp3`/`io2` EBS with proper mount options can saturate 16,000 IOPS. EC2 instances running Amazon Linux 2023 default to XFS on the root volume.

---

## 💡 Real-World Use Cases

- **Amazon Linux 2023 root volume management:** Default XFS root filesystem — growing EBS volume + `xfs_growfs` for zero-downtime disk expansion.
- **Database data directories:** PostgreSQL/MySQL on XFS with `noatime,nodiratime,logbsize=256k` mount options for 20-40% write performance improvement.
- **High-throughput log ingestion:** XFS allocation groups allow parallel writes from multiple fluentd/filebeat workers without contention.
- **NFS server backend:** XFS handles millions of files in a single directory efficiently (ext4 struggles above ~100k files per directory).
- **Container image storage:** Docker/containerd `overlay2` driver on XFS with `ftype=1` (required for overlay2 — always set at mkfs time).

---

## 🔧 Commands & Examples

### Creating and Mounting XFS

```bash
# Create XFS filesystem on a raw device
mkfs.xfs /dev/xvdf

# Create with specific options for production use
# -f: force (overwrite existing)
# -L: volume label
# -b size=4096: block size (match application I/O size)
# -l size=128m: journal/log size (larger = better write performance)
mkfs.xfs -f -L data01 -b size=4096 -l size=128m /dev/xvdf

# Create XFS for Docker overlay2 (CRITICAL: ftype=1)
mkfs.xfs -n ftype=1 /dev/xvdf

# Verify ftype setting
xfs_info /dev/xvdf | grep ftype
# naming   =version 2              bsize=4096   ascii-ci=0, ftype=1

# Mount with production-recommended options
mount -o noatime,nodiratime,logbsize=256k /dev/xvdf /data

# /etc/fstab entry for persistent mount
echo "UUID=$(blkid -s UUID -o value /dev/xvdf)  /data  xfs  defaults,noatime,nodiratime,logbsize=256k  0 2" >> /etc/fstab

# Mount all from fstab
mount -a
```

### XFS Filesystem Information

```bash
# Detailed filesystem info (Allocation Groups, log, naming config)
xfs_info /data
# meta-data=/dev/xvdf      isize=512    agcount=4, agsize=655360 blks
#          =               sectsz=512   attr=2, projid32bit=1
#          =               crc=1        finobt=1, sparse=1, rmapbt=0
#          =               reflink=1
# data     =               bsize=4096   blocks=2621440, imaxpct=25
# naming   =version 2     bsize=4096   ascii-ci=0, ftype=1
# log      =internal log  bsize=4096   blocks=2560, version=2
#          =               sectsz=512   sunit=0 blks, lazy-count=1
# realtime =none          extsz=4096   blocks=0, rtextents=0

# Show filesystem UUID
xfs_admin -l /dev/xvdf   # label
xfs_admin -u /dev/xvdf   # UUID

# Check space usage (standard tools work fine)
df -hT /data
# Filesystem     Type  Size  Used Avail Use% Mounted on
# /dev/xvdf      xfs    10G  1.2G  8.8G  12% /data

# Inode usage
df -i /data
```

### Online Resize (Grow Only)

```bash
# Scenario: EBS volume /dev/xvdf resized from 10GB to 20GB in AWS console
# Step 1: Verify kernel sees new size
lsblk /dev/xvdf
# NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
# xvdf 202:80   0  20G  0 disk /data   ← still shows 10G until grown

# If using LVM: first extend the LV, then grow XFS
lvextend -L +10G /dev/mapper/vg0-data
xfs_growfs /data
# meta-data=/dev/mapper/vg0-data   isize=512    agcount=8, agsize=655360 blks
# ...
# data blocks changed from 2621440 to 5242880

# For raw device (no LVM): XFS grows to fill the block device automatically
xfs_growfs /data

# Verify
df -h /data
```

### XFS Repair

```bash
# IMPORTANT: xfs_repair must be run on UNMOUNTED filesystem
# (Exception: read-only mount is OK for inspection)

# First, try journal replay (less invasive)
xfs_repair -n /dev/xvdf   # -n = dry run, no changes
# Phase 1 - find and verify superblock...
# Phase 2 - using internal log
# ...

# Full repair (unmount first!)
umount /data
xfs_repair /dev/xvdf

# If journal is dirty and repair fails, clear the log first
xfs_repair -L /dev/xvdf   # WARNING: -L clears the log, may lose recent data

# For mounted filesystem issues: use xfs_scrub (online, non-destructive)
xfs_scrub /data
# Checking filesystem at /data (device /dev/xvdf)
# Phase 1: Metadata integrity checks
# Phase 2: Check directory structure
# Phase 3: Check the inodes
# Phase 4: Defer geometry checks
# Phase 5: Full inode scan
# Phase 6: Checking summary counters
```

### XFS Dump and Restore (Best Practice Backup)

```bash
# xfsdump preserves XFS-specific attributes (extended attrs, etc.)
# Level 0 = full backup
xfsdump -l 0 -f /backup/data01-full.dump /data

# Level 1 = incremental since last level 0
xfsdump -l 1 -f /backup/data01-inc.dump /data

# Restore full backup
xfsrestore -f /backup/data01-full.dump /restore-point/

# List contents of a dump file
xfsrestore -f /backup/data01-full.dump -t

# Restore single file/directory
xfsrestore -f /backup/data01-full.dump -s path/to/file /restore-point/
```

### XFS Defragmentation and Optimization

```bash
# Check fragmentation level
xfs_db -r -c 'frag' /dev/xvdf
# actual 12345, ideal 12000, fragmentation factor 2.68%

# Online defragmentation
xfs_fsr /data          # Defrag mount point
xfs_fsr /dev/xvdf      # Defrag device
xfs_fsr -v /data       # Verbose output

# Preallocation for specific files (avoids fragmentation for growing files)
# Preallocate 1GB for a database file
xfs_io -c "falloc 0 1g" /data/db/large-table.dat

# Check extent count for a file (high count = fragmented)
xfs_io -c "fiemap" /data/somefile | wc -l

# Set real-time extent size hint for large sequential writes
xfs_io -c "extsize 1m" /data/video/
```

### Performance Tuning Mount Options

```bash
# Production mount options explained:
# noatime     — don't update atime on reads (saves IOPS)
# nodiratime  — don't update directory atime
# logbsize=256k — larger log buffer = better write throughput
# allocsize=64m — preallocate 64MB at a time for growing files
# largeio     — prefer large I/O requests (good for streaming workloads)

mount -o noatime,nodiratime,logbsize=256k,allocsize=64m /dev/xvdf /data

# For NVMe with high queue depth:
mount -o noatime,nodiratime,logbsize=256k,inode64 /dev/nvme1n1 /data

# Check currently active mount options
findmnt -o OPTIONS /data
# noatime,nodiratime,logbsize=262144,inode64,...

# Benchmark: compare with/without noatime
fio --filename=/data/test --rw=randread --bs=4k --direct=1 \
    --loops=5 --name=noatime-test --numjobs=4 --iodepth=32
```

---

## ⚠️ Gotchas & Pro Tips

- **XFS cannot be shrunk.** Ever. Plan your filesystem sizes carefully. Use LVM so you can add space without needing to shrink (thin provisioning helps here too).
- **`ftype=1` is MANDATORY for Docker overlay2.** If you create an XFS filesystem without `ftype=1`, Docker will refuse to start or fall back to a slower storage driver. Always use `mkfs.xfs -n ftype=1` when the filesystem will host containers.
- **`xfs_repair` on a mounted filesystem will corrupt it.** The tool checks if the filesystem is mounted, but always double-check with `findmnt /dev/xvdf` before running repair.
- **Journal size matters on write-heavy workloads.** Default journal is often too small (around 10MB). Use `-l size=512m` for databases or log-heavy workloads. You cannot change journal size post-creation.
- **`xfs_admin -U generate /dev/xvdf`** generates a new UUID — useful after cloning a volume in AWS (to avoid UUID collisions in fstab).
- **Pro tip — `xfs_io` for low-level testing:**

```bash
# Test sequential write performance without filesystem cache
xfs_io -f -c "pwrite -b 1m -S 0xAB 0 1g" -c "fsync" /data/testfile

# Check if a file is sparse
xfs_io -c "fiemap -v" /data/sparsefile | grep hole
```

- **Allocation Group count affects parallelism.** By default, XFS creates 4 AGs. For large filesystems on fast storage (NVMe RAID), increase AG count at mkfs time with `-d agcount=16` for better parallel write performance.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
