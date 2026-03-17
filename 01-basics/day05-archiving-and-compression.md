# 🐧 Archiving & Compression — Linux Mastery

> **Master tar, gzip, zstd, and parallel compression for production backups, CI/CD pipelines, and cloud storage efficiency.**

## 📖 Concept

Archiving and compression are foundational skills for DevOps engineers and SREs. Whether you're backing up `/etc` configurations, packaging artifacts for deployment, or optimizing storage costs in S3, the ability to choose the right tool and understand performance tradeoffs is critical. Linux offers a rich ecosystem of compression algorithms—each with different speed/ratio tradeoffs. Modern production systems increasingly use `zstd` and parallel compression tools like `pigz` to balance throughput and compression ratio.

The humble `tar` command is far more powerful than most realize: it can exclude files, transform paths during archiving, verify integrity without extraction, and pipe directly to cloud storage. Understanding `tar` deeply means faster backups, smaller container layers, and fewer failed deployments.

Compression isn't just about saving space—it's about pipeline efficiency. A 5GB artifact compressed to 1GB saves 4GB of network transfer. In Kubernetes CI/CD with thousands of builds per day, that's the difference between a 30-minute deployment window and a 6-minute one.

---

## 💡 Real-World Use Cases

- **Automated backups to S3**: Archive system configs with `tar czf` and stream directly to S3 via `aws s3 cp -` without intermediate disk writes
- **CI/CD artifact caching**: Compress build outputs with parallel `pigz` to speed up cache layers in Docker builds
- **Container layer optimization**: Use `zstd` instead of gzip in Dockerfile COPY commands to reduce image size and pull time
- **EBS snapshot compression**: Pre-compress data before snapshotting to reduce storage costs and snapshot transfer time
- **Log archival and compliance**: Compress old logs with `xz` for long-term cold storage while maintaining fast access to recent logs with gzip
- **Disaster recovery**: Verify backup integrity with `tar -tzf` before disaster strikes, catching corruption early

---

## 🔧 Commands & Examples

### tar: The Archive Workhorse

```bash
# Create a gzip archive, verbose output, with progress
tar czf backup.tar.gz /etc /home/user --exclude='*.log'

# Create with transform: strip leading path components
tar czf backup.tar.gz -C / etc home/user
# Results in: etc/passwd, home/user/file.txt (not /etc/passwd, /home/user/file.txt)

# Extract specific file without extracting entire archive
tar xzf backup.tar.gz path/to/file.txt

# List archive contents without extracting
tar tzf backup.tar.gz | head -20

# Verify tar.gz integrity without extracting (tar can detect gzip corruption)
tar tzf backup.tar.gz > /dev/null && echo "Archive OK" || echo "Corrupted"

# Create archive with custom compression (zstd)
tar --use-compress-program=zstd -cf backup.tar.zst /etc

# Exclude multiple patterns
tar czf backup.tar.gz /home --exclude='*.tmp' --exclude='.cache' --exclude='node_modules'

# Show compression progress with pv (pipe viewer)
tar cf - /large/dir | pv | gzip > backup.tar.gz
```

### Compression Algorithm Tradeoffs

```bash
# gzip: ubiquitous, moderate speed and ratio (2-5x)
tar czf config.tar.gz /etc
# Time: ~1s, Size: 5.2 MB

# bzip2: better compression (~6-9x) but slower
tar cjf config.tar.bz2 /etc
# Time: ~3s, Size: 4.1 MB

# xz: best compression (~10-15x) but very slow (suitable for cold storage)
tar cJf config.tar.xz /etc
# Time: ~8s, Size: 3.2 MB

# zstd: modern, fast, good compression (~5-8x), ideal for CI/CD
tar --use-compress-program=zstd -cf config.tar.zst /etc
# Time: ~0.5s, Size: 4.8 MB

# Compare sizes of same data
ls -lh config.tar*
# -rw-r--r--  1 user  5.2M  config.tar.gz
# -rw-r--r--  1 user  4.1M  config.tar.bz2
# -rw-r--r--  1 user  3.2M  config.tar.xz
# -rw-r--r--  1 user  4.8M  config.tar.zst
```

### Streaming Archives to Cloud Storage

```bash
# Stream tar.gz directly to S3 (no intermediate disk file)
tar czf - /large/dataset | aws s3 cp - s3://my-bucket/backup-$(date +%Y%m%d).tar.gz

# Stream from S3, extract to local (for recovery)
aws s3 cp s3://my-bucket/backup.tar.gz - | tar xz

# Monitor transfer speed with pv
tar czf - /data | pv | aws s3 cp - s3://bucket/backup.tar.gz

# Parallel gzip compression (pigz) on multi-core systems
tar cf - /data | pigz -p 8 > backup.tar.gz  # Use 8 cores
# 4-8x faster than single-threaded gzip on modern hardware

# Parallel zstd compression
tar cf - /data | zstd -T0 -o backup.tar.zst  # T0 = auto-detect CPU cores
```

### Parallel Compression with pigz

```bash
# Install pigz (parallel gzip)
apt-get install pigz  # Debian/Ubuntu
brew install pigz      # macOS

# Use pigz as default gzip replacement (symlink)
ln -s /usr/bin/pigz /usr/bin/gzip

# Or use explicitly with tar
tar cf - /var/log | pigz -9 -p 8 > logs.tar.gz

# Compare: single-threaded gzip vs parallel pigz
time tar cf - /large/dir | gzip -9 > backup.tar.gz
time tar cf - /large/dir | pigz -9 -p 8 > backup.tar.gz
# pigz is typically 4-7x faster on 8-core systems

# Parallel extraction (unpigz)
pigz -d -p 8 backup.tar.gz  # Decompress using 8 threads
tar xf backup.tar
```

### Checking Archive Integrity

```bash
# Test gzip integrity without full extraction
gzip -t backup.tar.gz && echo "OK" || echo "Corrupted"

# Test tar.gz: list first, then extract to /dev/null
tar tzf backup.tar.gz > /dev/null

# Find specific corrupted file in large archive
tar tzf backup.tar.gz 2>&1 | grep -i error

# Create checksum before backup
sha256sum backup.tar.gz > backup.tar.gz.sha256
# Later, verify:
sha256sum -c backup.tar.gz.sha256

# On-the-fly verification: archive + checksum in one pipe
tar czf - /data | tee backup.tar.gz | sha256sum > backup.tar.gz.sha256
```

### Production Backup Script

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="/backups"
RETENTION_DAYS=30
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/system-backup-${TIMESTAMP}.tar.zst"

# Create backup directory
mkdir -p "${BACKUP_DIR}"

# Archive with zstd (fast + good compression)
tar --use-compress-program='zstd -T0 -10' \
    -cf "${BACKUP_FILE}" \
    --exclude='/proc' \
    --exclude='/sys' \
    --exclude='/dev' \
    --exclude='/tmp' \
    --exclude='/run' \
    --exclude='*.log' \
    /etc /home /root

# Verify integrity
if tar -tzf "${BACKUP_FILE}" > /dev/null 2>&1; then
    echo "✓ Backup verified: ${BACKUP_FILE}"
else
    echo "✗ Backup corrupted!" >&2
    rm "${BACKUP_FILE}"
    exit 1
fi

# Create checksum
sha256sum "${BACKUP_FILE}" > "${BACKUP_FILE}.sha256"

# Clean old backups (older than 30 days)
find "${BACKUP_DIR}" -name "system-backup-*.tar.zst" -mtime +${RETENTION_DAYS} -delete

# Optional: upload to S3
# aws s3 cp "${BACKUP_FILE}" s3://my-backup-bucket/
```

### Cron-based Automated Backups

```bash
# Add to /etc/cron.d/backup-system
0 2 * * * root tar --use-compress-program=zstd -cf /backups/etc-$(date +\%Y\%m\%d).tar.zst /etc && \
    tar -tzf /backups/etc-$(date +%Y%m%d).tar.zst > /dev/null || \
    echo "Backup failed on $(hostname)" | mail -s "Backup Error" admin@example.com
```

---

## ⚠️ Gotchas & Pro Tips

- **Leading slashes in archives**: Always use `-C` to change directory before archiving, or tar will create absolute paths. This prevents `/etc/passwd` from being extracted outside your intended directory.
  ```bash
  tar czf backup.tar.gz -C / etc  # Correct: archive as etc/, not /etc/
  tar czf backup.tar.gz /etc       # Wrong: archive as /etc/ (extraction requires careful handling)
  ```

- **Sparse files**: `tar --sparse` can save enormous space when archiving VMs or disk images with large sparse regions.
  ```bash
  tar czf image.tar.gz --sparse /mnt/image.img  # 100GB sparse → 5GB archive
  ```

- **gzip memory consumption**: Large archives with default gzip compression use significant memory. For constrained environments, use `gzip --fast` or switch to `zstd --fast`.

- **Symlink handling**: By default, tar dereferences symlinks and archives the actual files. Use `--no-dereference` to preserve symlinks themselves.

- **Performance myth**: Highest compression doesn't always mean best space savings. `zstd -10` is often 80% as good as `xz` but 50x faster. For CI/CD, choose speed.

- **Pipe failures silently fail**: In `tar czf - /data | s3_upload`, if `s3_upload` crashes, tar might still exit cleanly. Use `set -o pipefail` in scripts.

- **Incremental backups**: Use `tar --listed-incremental=snapshot.snar` to create incremental backups that only archive changes since last run—huge savings for large systems.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
