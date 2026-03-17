# 🐧 Bind Mounts & Loop Devices — Linux Mastery

> **Master advanced filesystem mounting techniques—critical for containers, forensics, and advanced storage scenarios.**

## 📖 Concept

Bind mounts and loop devices are the hidden scaffolding of modern Linux infrastructure. Docker volumes rely on bind mounts, Kubernetes hostPath volumes expose them directly, and forensic analysis often requires mounting disk images via loop devices. Most engineers encounter these concepts only when things break—container volumes don't persist, extracted disk images can't be accessed, or dynamic storage provisioning fails.

A bind mount is a way to remount a directory tree (or single file) at a new location in the filesystem. It's not a copy—it's the same underlying inode visible from two paths. Changes in one location appear in the other. At the kernel level, this is incredibly efficient: the VFS (Virtual File System) layer simply adds another reference to the same filesystem. Containers use this extensively: a volume mount like `docker run -v /host/path:/container/path` is implemented as a bind mount.

Loop devices are virtual block devices backed by files. They unlock powerful use cases: mounting ISO images, mounting compressed disk snapshots, running nested VMs, and forensic analysis of acquired disk images. A loop device is the bridge between the block device interface and a regular file.

---

## 💡 Real-World Use Cases

- **Docker volume mounting**: Under the hood, `docker run -v /host:/container` creates bind mounts to share host directories with containers
- **Kubernetes persistent volumes**: hostPath volumes use bind mounts; understanding them helps debug volume permission issues
- **Forensic analysis**: Mount acquired EC2 snapshots (EBS snapshots exported as .img) to analyze filesystem and recover data
- **Development environments**: Bind mount your source code into containers for live reload, or mount `/dev` for hardware access
- **Chroot jails and containers**: Bind mount essential directories to create minimal container rootfs
- **ISO and disk image forensics**: Mount .iso files or recovered disk images without extracting
- **Nested virtualization and KVM**: Create sparse disk images with loop devices for test VMs
- **CI/CD artifact caching**: Bind mount cache directories from host to containers for faster builds

---

## 🔧 Commands & Examples

### Bind Mounts: Remounting Directories

```bash
# Basic bind mount: mount /host/source at /mnt/target
sudo mount --bind /host/source /mnt/target

# Both paths now reference the same files
echo "hello" > /host/source/file.txt
cat /mnt/target/file.txt  # Output: hello

# Read-only bind mount
sudo mount --bind -o remount,ro /host/source /mnt/target

# Verify mount with findmnt
findmnt /mnt/target

# Unmount
sudo umount /mnt/target
```

### Creating Persistent Bind Mounts in /etc/fstab

```bash
# /etc/fstab example:
# SOURCE              MOUNT_POINT      TYPE    OPTIONS         DUMP PASS
/data/archive        /var/backups     none    bind,ro         0    0
/home/user/projects  /srv/workdir     none    bind,defaults   0    0
/dev                 /containers/dev  none    bind,rbind      0    0

# After editing /etc/fstab, test with:
sudo mount -a

# rbind is "recursive bind" — also mounts submounts
# Useful for mounting /dev or /proc into container roots
```

### Bind Mount Tricks for Development

```bash
# Mount source code into running container for live reload
docker run -it -v $(pwd):/app ubuntu:22.04 bash
# Now /app in container mirrors your host pwd
# Edit files on host, changes appear instantly in container

# Combine bind mount with read-only mode
docker run -v $(pwd):/app:ro myapp:latest
# App can read from /app but not write

# Bind mount individual files (not just directories)
docker run -v ~/.ssh/id_rsa:/root/.ssh/id_rsa:ro ubuntu:22.04

# Problematic: bind mounting /etc into container
docker run -v /etc:/etc myapp:latest
# If app modifies /etc in container, affects host /etc!
# Use with extreme caution.
```

### Loop Devices: Mounting Disk Images

```bash
# Create a sparse disk image (doesn't allocate full size upfront)
fallocate -l 10G disk.img

# Or using dd (slower but works on all filesystems)
dd if=/dev/zero of=disk.img bs=1G count=10

# Create a filesystem in the image
mkfs.ext4 disk.img

# Mount the image via loop device
sudo mount -o loop disk.img /mnt/image
# Kernel automatically finds available loop device (/dev/loop0, /dev/loop1, etc.)

# Verify
mount | grep loop

# Unmount
sudo umount /mnt/image

# Manual loop device setup (less common, but educational)
sudo losetup /dev/loop0 disk.img    # Attach image to loop0
sudo mkfs.ext4 /dev/loop0           # Format as ext4
sudo mount /dev/loop0 /mnt/image    # Mount
sudo umount /mnt/image              # Unmount
sudo losetup -d /dev/loop0          # Detach loop device
```

### losetup: Managing Loop Devices

```bash
# List all loop devices
sudo losetup -l

# Attach disk image to specific loop device
sudo losetup /dev/loop0 disk.img

# Find free loop device
sudo losetup -f   # Returns /dev/loop0, /dev/loop1, etc.

# Detach loop device
sudo losetup -d /dev/loop0

# Find which image is attached to a device
sudo losetup /dev/loop0

# Attach and immediately mount in one command
sudo losetup -f disk.img   # Find free device
sudo mount $(sudo losetup -f) /mnt/image  # Mount it

# Resize loop-backed filesystem
sudo e2fsck -f /dev/loop0          # Check filesystem
sudo resize2fs /dev/loop0 20G      # Resize to 20GB
```

### ISO Image Mounting

```bash
# Mount ISO without burning to disc
sudo mount -o loop,ro ubuntu-22.04-live-server-amd64.iso /mnt/iso

# Browse contents
ls /mnt/iso
cat /mnt/iso/README.md

# Extract specific files
cp /mnt/iso/casper/initrd /tmp/

# Unmount
sudo umount /mnt/iso

# Alternative: Mount directly without explicit loop (kernel handles it)
sudo mount -o ro ubuntu.iso /mnt/iso
```

### EBS Snapshot Recovery: Mounting Attached Volume

```bash
# Scenario: EC2 instance has root volume /dev/xvda
# You launch recovery instance, attach original volume as /dev/xvdf

# Find the attached volume
lsblk
# Shows: xvdf (newly attached volume)

# If it's an LVM volume:
sudo vgscan           # Find volume groups
sudo vgchange -ay     # Activate volume groups
sudo lvscan           # Find logical volumes
sudo mount /dev/vg0/root /mnt/recovery

# If it's a simple partition:
sudo mount /dev/xvdf1 /mnt/recovery

# Browse and recover files
ls /mnt/recovery/etc/
cp /mnt/recovery/home/user/important.txt /home/ubuntu/

# Unmount
sudo umount /mnt/recovery
sudo vgchange -an     # Deactivate volume groups if using LVM
```

### Bind Mounting Device Files for Containers

```bash
# Container needs access to host's /dev/tty
docker run -v /dev/tty:/dev/tty --device /dev/tty myapp:latest

# But what if you want a minimal /dev inside container?
# Bind mount only what's needed:
docker run \
  -v /dev/null:/dev/null \
  -v /dev/zero:/dev/zero \
  -v /dev/random:/dev/random \
  -v /dev/urandom:/dev/urandom \
  myapp:latest

# Kubernetes equivalent (hostPath volume):
apiVersion: v1
kind: Pod
metadata:
  name: privileged-access
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: dev-tty
      mountPath: /dev/tty
  volumes:
  - name: dev-tty
    hostPath:
      path: /dev/tty
```

### Creating Minimal Chroot Environments with Bind Mounts

```bash
#!/bin/bash
# Create a minimal chroot jail

JAIL_ROOT=/tmp/jail
mkdir -p "$JAIL_ROOT"

# Create minimal directory structure
mkdir -p "$JAIL_ROOT"/{bin,lib,lib64,etc,proc,sys}

# Bind mount essential directories
sudo mount --bind /bin "$JAIL_ROOT/bin"
sudo mount --bind /lib "$JAIL_ROOT/lib"
sudo mount --bind /lib64 "$JAIL_ROOT/lib64"
sudo mount --bind /etc "$JAIL_ROOT/etc"

# Mount proc and sys (read-only)
sudo mount --bind -o remount,ro,bind /etc "$JAIL_ROOT/etc"
sudo mount -t proc none "$JAIL_ROOT/proc"
sudo mount -t sysfs none "$JAIL_ROOT/sys"

# Enter jail
sudo chroot "$JAIL_ROOT" /bin/bash

# Cleanup (reverse order, unmount child mounts first)
sudo umount "$JAIL_ROOT/sys"
sudo umount "$JAIL_ROOT/proc"
sudo umount "$JAIL_ROOT/etc"
sudo umount "$JAIL_ROOT/bin"
sudo umount "$JAIL_ROOT/lib"
sudo umount "$JAIL_ROOT/lib64"
```

### Detecting Bind Mounts and Shared Filesystems

```bash
# Find all bind mounts on system
mount | grep bind

# More detailed view with filesystem type
findmnt -t tmpfs,ext4,btrfs -l

# Show mount propagation (shared, slave, private, unbindable)
findmnt --show-propagation

# Check if two paths are on same mount
stat --printf='%d\n' /host/path /container/path
# If device numbers match, they're on same filesystem (might be bind mount)

# Inspect bind mount details
cat /proc/mounts | grep bind

# Unmount all bind mounts under directory
sudo umount -R /mnt/target  # Recursive unmount
```

### tmpfs: RAM-Backed Filesystem

```bash
# Create RAM-backed mount (faster than disk, but volatile)
sudo mount -t tmpfs -o size=2G tmpfs /mnt/ramdisk

# Use in CI for fast caches
sudo mount -t tmpfs -o size=5G tmpfs /var/cache/docker

# Docker compose example:
version: '3'
services:
  app:
    image: myapp:latest
    tmpfs:
      - /tmp          # tmpfs at /tmp
      - /run:noexec   # tmpfs at /run, no-exec for security

# Performance: tmpfs is much faster for I/O but uses RAM
# Useful for: build caches, temporary files, CI/CD speed
```

### Troubleshooting Bind Mounts

```bash
# Permission denied on bind mount?
# Check source directory permissions
ls -ld /host/source

# If bind mount is read-only, but should be r/w
sudo mount -o remount,rw /mnt/target

# Bind mount disappears after reboot?
# Add to /etc/fstab for persistence

# Container can't access bind mounted volume?
# Check SELinux or AppArmor restrictions
sudo sestatus                   # SELinux status
sudo getenforce                 # SELinux mode
sudo apparmor_status            # AppArmor status

# Mount still in use, can't unmount?
sudo fuser -m /mnt/target       # Find processes using mount
sudo lsof +D /mnt/target        # List open files in mount
```

---

## ⚠️ Gotchas & Pro Tips

- **Bind mount permissions**: If source directory is owned by root with 700 permissions, and your container runs as non-root, you'll get permission denied. Use proper ownership or change mount permissions.

- **Loop device limits**: By default, Linux supports 8 loop devices (/dev/loop0 through /dev/loop7). Modern kernels allow more, but if you run out, images won't mount. Increase with `modprobe loop max_loop=32`.

- **Loop devices and qemu-nbd**: For modern cloud VM images, use `qemu-nbd` instead of loop devices. It's faster and handles complex partition schemes better.

- **Recursive bind mounts (-rbind)**: Regular bind mounts don't mount submounts. Use `-rbind` to recursively bind all mounted filesystems under source. Critical for `/dev`, `/proc`, `/sys`.

- **Propagation modes matter**: By default, mounts are `private`. Changes in bind-mounted directory don't propagate to host. Use `shared` propagation for bidirectional updates: `mount --make-shared /mnt/target`.

- **tmpfs and swap**: tmpfs uses RAM and can spill to swap. In containers with memory limits, tmpfs growth can cause OOMKill. Monitor tmpfs usage with `df /mnt/ramdisk`.

- **FUSE for forensics**: For complex disk images (BitLocker, encrypted partitions), consider FUSE-based tools like `fuse-archive` or specialized forensics tools instead of loop devices.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
