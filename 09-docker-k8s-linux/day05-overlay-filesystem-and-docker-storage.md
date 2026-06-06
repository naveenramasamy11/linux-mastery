# 🐧 Overlay Filesystem & Docker Storage Internals — Linux Mastery

> **Understand how Docker actually stores data — layers, copy-on-write, and why your container images are the size they are.**

## 📖 Concept

Every Docker container you run is built on Linux's overlay filesystem (`overlayfs`). Understanding how overlayfs works is essential for debugging disk space issues, optimizing image sizes, understanding why bind mounts behave differently than volumes, and diagnosing why containers can't write files in certain directories.

**OverlayFS fundamentals:** overlayfs stacks multiple directories into a single view. It has three layers:
- **LowerDir (read-only):** The image layers — read-only base filesystem. Multiple image layers are stacked here using the `lowerdir:lowerdir2:...` syntax.
- **UpperDir (read-write):** The container-specific writable layer. Any file modified in the container is copied here (copy-on-write).
- **WorkDir:** A scratch directory required by overlayfs for atomic operations.
- **MergedDir:** The unified view seen by the container process.

**Copy-on-write (CoW):** When a container modifies a file from the image (LowerDir), the full file is copied to UpperDir first, then modified. This means: (1) the first write to a file in an image layer is slow (proportional to file size), (2) large files modified frequently (like SQLite databases) belong on volumes not the container filesystem, and (3) the container's UpperDir grows for every file touched.

**Docker storage drivers:** `overlay2` is the default and correct choice for all modern Linux systems. Legacy drivers (`aufs`, `devicemapper`, `btrfs`) are still around but should be avoided.

---

## 💡 Real-World Use Cases

- Debug "no space left on device" errors in containers when `df` shows plenty of disk space
- Understand why a Docker image built with `apt-get install` in one layer and cleanup in another still has a large image
- Optimize Dockerfile layer order to maximize cache efficiency
- Explain to developers why their container can't write to a directory that exists in the base image
- Understand what happens at the kernel level when `docker run` starts a container

---

## 🔧 Commands & Examples

### Exploring OverlayFS Directly

```bash
# Simple overlayfs demo without Docker
mkdir -p /tmp/overlay/{lower,upper,work,merged}

# Create files in the lower (read-only) layer
echo "original content" > /tmp/overlay/lower/base_file.txt
echo "also in lower" > /tmp/overlay/lower/lower_only.txt

# Mount overlayfs
mount -t overlay overlay \
    -o lowerdir=/tmp/overlay/lower,\
upperdir=/tmp/overlay/upper,\
workdir=/tmp/overlay/work \
    /tmp/overlay/merged

# View the merged filesystem — sees both lower files
ls /tmp/overlay/merged/
cat /tmp/overlay/merged/base_file.txt   # "original content"

# Modify a file (copy-on-write happens here)
echo "modified in container" > /tmp/overlay/merged/base_file.txt

# The original is untouched in lower
cat /tmp/overlay/lower/base_file.txt   # "original content"

# The copy is in upper
ls /tmp/overlay/upper/
cat /tmp/overlay/upper/base_file.txt   # "modified in container"

# Create a new file in the container
echo "new file" > /tmp/overlay/merged/new_container_file.txt

# New file is only in upper
ls /tmp/overlay/upper/   # new_container_file.txt

# Deleting a file from lower creates a "whiteout" file in upper
rm /tmp/overlay/merged/lower_only.txt
ls -la /tmp/overlay/upper/   # c---------. 1 root root 0, 0 lower_only.txt (char device = whiteout)

# Cleanup
umount /tmp/overlay/merged
```

### Inspecting Docker's OverlayFS Layers

```bash
# Show Docker storage driver info
docker info | grep -A5 "Storage Driver"
# Storage Driver: overlay2
#  Backing Filesystem: xfs
#  Supports d_type: true
#  Native Overlay Diff: true

# Inspect a running container's overlay mount
CONTAINER_ID=$(docker run -d nginx)
docker inspect $CONTAINER_ID | jq '.[0].GraphDriver'

# Output shows:
# {
#   "Data": {
#     "LowerDir": "/var/lib/docker/overlay2/abc123/diff:/var/lib/docker/overlay2/def456/diff:...",
#     "MergedDir": "/var/lib/docker/overlay2/xyz789/merged",
#     "UpperDir": "/var/lib/docker/overlay2/xyz789/diff",
#     "WorkDir": "/var/lib/docker/overlay2/xyz789/work"
#   },
#   "Name": "overlay2"
# }

# Look at the actual mount
mount | grep overlay | grep $CONTAINER_ID

# Browse the container's root filesystem from the host
MERGED=$(docker inspect $CONTAINER_ID | jq -r '.[0].GraphDriver.Data.MergedDir')
ls $MERGED
ls $MERGED/etc/nginx/

# Browse the writable layer (UpperDir)
UPPER=$(docker inspect $CONTAINER_ID | jq -r '.[0].GraphDriver.Data.UpperDir')
ls $UPPER

# After making changes in the container, watch the UpperDir grow
docker exec $CONTAINER_ID bash -c "echo test > /tmp/test.txt"
ls $UPPER/tmp/

# List all image layers
docker image inspect nginx | jq '.[0].RootFS.Layers'
# Each SHA256 = one layer in LowerDir

# Show layer disk usage
docker system df -v
docker image ls --format "{{.Repository}}:{{.Tag}} {{.Size}}"

# Show what's in each image layer
docker history nginx --no-trunc
# IMAGE               CREATED BY                                      SIZE
# sha256:abc...       /bin/sh -c #(nop)  CMD ["nginx" "-g" "daem...   0B
# sha256:def...       /bin/sh -c #(nop)  EXPOSE 80                   0B
# sha256:ghi...       /bin/sh -c set -x  && addgroup --system --...  61.1MB
```

### Understanding Container Disk Space

```bash
# Common confusion: "no space left on device" but df shows space

# Check overall Docker disk usage
docker system df
# TYPE                TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images              15        3         4.2GB     3.1GB (73%)
# Containers          5         5         2.3GB     0B (0%)
# Local Volumes       8         4         12.4GB    3.2GB (25%)
# Build Cache         23        0         1.8GB     1.8GB

# The real disk usage breakdown
du -sh /var/lib/docker/
du -sh /var/lib/docker/overlay2/   # layer storage
du -sh /var/lib/docker/volumes/    # named volumes
du -sh /var/lib/docker/image/      # image metadata

# Find largest containers (by their UpperDir size)
for dir in /var/lib/docker/overlay2/*/diff; do
    size=$(du -sm "$dir" 2>/dev/null | cut -f1)
    [ "$size" -gt 100 ] && echo "$size MB: $dir"
done | sort -rn | head -10

# Find which container owns a large overlay directory
LAYER_ID="abc123xyz"
docker ps -q | xargs docker inspect | \
    jq -r '.[] | select(.GraphDriver.Data.UpperDir | contains("'$LAYER_ID'")) | .Name'

# Disk space used by a specific container (writable layer only)
docker inspect -f '{{.Id}}' mycontainer | \
    xargs -I{} sh -c 'du -sh /var/lib/docker/overlay2/$(docker inspect {} | jq -r ".[0].GraphDriver.Data.UpperDir | split(\"/\") | .[-2]")/diff'

# Or simpler with docker stats
docker stats --no-stream --format "{{.Name}}\t{{.BlockIO}}"
```

### Docker Volumes vs Bind Mounts vs tmpfs

```bash
# ─── Named Volumes (managed by Docker) ────────────────────────────────────────
# Stored at /var/lib/docker/volumes/<name>/_data
# Best for: persistent data (databases, uploads)

# Create and use a named volume
docker volume create mydata
docker run -v mydata:/var/lib/postgresql/data postgres

# Inspect volume location
docker volume inspect mydata
# Mountpoint: /var/lib/docker/volumes/mydata/_data
ls /var/lib/docker/volumes/mydata/_data

# Backup a volume
docker run --rm \
    -v mydata:/source:ro \
    -v $(pwd):/backup \
    alpine tar -czf /backup/mydata-backup.tar.gz -C /source .

# ─── Bind Mounts (host directory) ────────────────────────────────────────────
# Mounts a host directory directly into the container
# Best for: config files, development source code, log collection

docker run -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf:ro nginx
docker run -v /var/log/myapp:/var/log/app myapp

# Bind mount with specific user (avoid running as root in container)
docker run \
    --user 1000:1000 \
    -v $(pwd)/data:/app/data \
    myapp

# ─── tmpfs Mounts (in-memory) ─────────────────────────────────────────────────
# Ephemeral, in-memory storage — not written to disk
# Best for: secrets, session data, temp files that must not persist

docker run \
    --tmpfs /tmp \
    --tmpfs /run:rw,noexec,nosuid,size=100m \
    myapp

# ─── Comparing storage performance ────────────────────────────────────────────
# Test write speed to different storage locations

# Container filesystem (overlayfs) — slowest for large files
docker run --rm alpine sh -c "dd if=/dev/zero of=/tmp/test bs=1M count=100 2>&1 | tail -1"

# Named volume (direct ext4/xfs) — faster
docker run --rm -v testperf:/tmp alpine sh -c "dd if=/dev/zero of=/tmp/test bs=1M count=100 2>&1 | tail -1"

# tmpfs — fastest
docker run --rm --tmpfs /tmp alpine sh -c "dd if=/dev/zero of=/tmp/test bs=1M count=100 2>&1 | tail -1"
```

### Dockerfile Layer Optimization

```bash
# BAD — three separate layers, cleanup doesn't reduce image size
# (the apt cache is committed in the first layer and can't be removed from it)
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y nginx
RUN rm -rf /var/lib/apt/lists/*

# GOOD — single RUN layer, cleanup included in the SAME layer
FROM ubuntu:22.04
RUN apt-get update && \
    apt-get install -y nginx && \
    rm -rf /var/lib/apt/lists/*

# BEST — BuildKit with cache mounts (doesn't add cache to image at all)
# syntax=docker/dockerfile:1
FROM ubuntu:22.04
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y nginx

# Layer order matters for caching — put frequently-changing layers LAST
FROM node:18-alpine

# FIRST: rarely changes (install deps — cached unless package.json changes)
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# LAST: changes every build (application code)
COPY . .

CMD ["node", "server.js"]

# Use .dockerignore to avoid copying unnecessary files into the image
cat > .dockerignore << 'EOF'
node_modules/
.git/
*.log
.env
.env.*
coverage/
.nyc_output/
dist/
*.test.js
README.md
.dockerignore
Dockerfile
EOF

# Analyze image layers and their sizes
docker history myimage --no-trunc --format "table {{.Size}}\t{{.CreatedBy}}" | \
    sort -h | head -20

# Dive — excellent interactive image layer explorer
# docker run --rm -it wagoodman/dive myimage
```

### Cleanup and Maintenance

```bash
# Remove stopped containers
docker container prune

# Remove unused images (not referenced by any container)
docker image prune

# Remove ALL unused objects (images, containers, networks, build cache)
docker system prune --all --volumes

# More targeted cleanup
docker images | grep "<none>" | awk '{print $3}' | xargs docker rmi  # dangling images
docker volume ls -qf dangling=true | xargs docker volume rm            # unused volumes

# Limit container log size (critical for long-running containers)
# /etc/docker/daemon.json
cat > /etc/docker/daemon.json << 'EOF'
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "100m",
        "max-file": "3"
    },
    "storage-driver": "overlay2"
}
EOF

systemctl reload docker

# Find which images are taking the most space
docker images --format "{{.Repository}}:{{.Tag}}\t{{.Size}}" | \
    sort -t$'\t' -k2 -h | tail -10
```

---

## ⚠️ Gotchas & Pro Tips

- **Never store databases on the container filesystem (UpperDir):** The overlay2 CoW layer is not optimized for database I/O patterns. Every write to a previously-unmodified database page copies the entire 4KB block (or worse, the whole sparse file region). Use named volumes or bind mounts for anything that does frequent writes.

- **`docker system df` vs `du /var/lib/docker`:** `docker system df` shows Docker's view of disk usage. `du -sh /var/lib/docker` shows the actual filesystem usage. They often differ because Docker shares layers between images (deduplication) — the `du` output is more accurate for "how much disk will I free if I remove everything."

- **Layer deduplication only works within one Docker host:** Two EC2 instances each running `nginx` don't share the nginx base layer. This is why ECR pull-through cache, shared EFS for `/var/lib/docker`, or container snapshots matter at scale.

- **Dockerfile `COPY . .` invalidates all subsequent cache:** Once Docker hits a layer where files changed, all subsequent layers are rebuilt. If your Dockerfile ends with `COPY . .` followed by `RUN npm install`, the npm install runs on every code change. Reverse it: install deps first, then copy source.

- **`docker commit` creates unoptimized images:** `docker commit` creates a single large layer from the container's UpperDir — no layer sharing, no cache efficiency. Always use Dockerfiles for reproducible, optimized images.

- **OverlayFS with XFS requires `d_type` support:** Docker overlay2 on XFS requires XFS to be formatted with `ftype=1` (d_type enabled). AWS EC2 instance store and some EBS volumes formatted before 2017 may not have this. Check with `xfs_info / | grep ftype` — if `ftype=0`, Docker will fall back to `vfs` storage driver (very slow).

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
