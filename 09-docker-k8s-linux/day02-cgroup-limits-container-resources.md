# 🐧 cgroup Limits & Container Resource Control — Linux Mastery

> **Every Docker `--memory` flag and Kubernetes `resources.limits` setting is just a cgroup file write under the hood — knowing the kernel layer lets you diagnose throttling, OOM kills, and resource contention that no container abstraction exposes.**

## 📖 Concept

Container resource limits in Docker and Kubernetes are not magic — they are kernel cgroup parameters written to the cgroup filesystem at container start time. When you set `--memory=512m` in Docker or `resources.limits.memory: 512Mi` in a Kubernetes pod spec, the container runtime (containerd, cri-o, or docker) writes `536870912` to the container's `memory.max` (cgroups v2) or `memory.limit_in_bytes` (cgroups v1) file.

Understanding this mapping is critical for three reasons. First, it lets you read the ground truth — when kubectl or docker inspect disagree with what you're seeing, the cgroup file is always authoritative. Second, it explains the behaviour you see: why a container that appears to use less than its limit still gets OOM-killed (the kernel includes page cache in cgroup memory accounting), why CPU throttling happens even on an idle host, and why Kubernetes QoS classes matter. Third, it helps you tune limits correctly — setting limits too tight causes throttling and OOM kills, setting them too loose causes noisy-neighbour problems on the same node.

In AWS EKS and ECS environments, getting resource limits right is a cost and reliability optimisation. Over-provisioned pods waste EC2 capacity; under-provisioned pods cause latency spikes and crashes. The kernel telemetry (cgroup stat files) is the most accurate source for right-sizing.

---

## 💡 Real-World Use Cases

- Diagnosing why a K8s pod keeps restarting despite appearing healthy (OOM kill, visible in container's cgroup `memory.events`)
- Right-sizing ECS task definitions by reading actual memory/CPU usage from cgroup files on the EC2 instance
- Understanding why a pod hits CPU throttling even when node CPU is at 20% (cpu.stat throttled periods)
- Debugging containerd or Docker cgroup v2 migration issues on Amazon Linux 2023 or Ubuntu 22.04 nodes
- Configuring init containers with separate resource limits to avoid starving the main container during startup

---

## 🔧 Commands & Examples

### Docker Resource Limits — Flags to cgroup Mapping

```bash
# --- Memory Limits ---

# Set hard memory limit (container OOM-killed if exceeded)
docker run --memory=512m nginx

# Set both memory and swap limit
# --memory-swap = memory + swap total
docker run --memory=512m --memory-swap=1g nginx   # 512MB RAM + 512MB swap
docker run --memory=512m --memory-swap=-1 nginx   # unlimited swap
docker run --memory=512m --memory-swap=512m nginx  # NO swap (swap limit = RAM limit)

# Set memory soft limit (kernel tries to stay under, but doesn't kill)
docker run --memory=512m --memory-reservation=256m nginx

# --- CPU Limits ---

# Limit to specific CPUs (CPU pinning via cpuset)
docker run --cpuset-cpus="0,1" nginx     # only CPUs 0 and 1
docker run --cpuset-cpus="0-3" nginx     # CPUs 0,1,2,3

# CPU quota (period-based hard limit)
# --cpu-period: length of scheduling window (default 100ms = 100000us)
# --cpu-quota: how much of the period the container can use
docker run --cpu-period=100000 --cpu-quota=50000 nginx   # 0.5 CPU
docker run --cpus=1.5 nginx                               # shorthand for 1.5 CPUs

# CPU shares (relative weight, not a hard limit)
# Default weight is 1024; lower = lower priority
docker run --cpu-shares=512 nginx    # half the default priority

# --- I/O Limits ---

# Limit block device read/write bandwidth
docker run --device-read-bps /dev/nvme0n1:50mb nginx
docker run --device-write-bps /dev/nvme0n1:50mb nginx

# Limit IOPS
docker run --device-read-iops /dev/nvme0n1:1000 nginx
docker run --device-write-iops /dev/nvme0n1:1000 nginx

# --- PIDs ---
docker run --pids-limit=100 nginx
```

### Verifying Docker Container cgroup Files

```bash
# Find the container's cgroup path
CONTAINER_ID="abc123"
docker inspect $CONTAINER_ID | grep -i cgroup
# Or:
docker inspect $CONTAINER_ID --format '{{.HostConfig.CgroupParent}}'

# Find the actual cgroup directory (cgroups v2 on modern systems)
CONTAINER_ID=$(docker ps -q --filter "name=myapp" | head -1)
CGROUP_PATH=$(find /sys/fs/cgroup -type d -name "*${CONTAINER_ID:0:12}*" 2>/dev/null | head -1)
echo "cgroup path: $CGROUP_PATH"

# Read memory limit
cat "$CGROUP_PATH/memory.max"
# If output is "max" = no limit set

# Read actual memory usage
cat "$CGROUP_PATH/memory.current"

# Read memory statistics breakdown
cat "$CGROUP_PATH/memory.stat"
# anon   = heap/stack (non-evictable)
# file   = page cache (evictable)
# slab   = kernel slab allocator

# Read OOM events
cat "$CGROUP_PATH/memory.events"
# oom_kill > 0 means the container was OOM-killed

# Read CPU throttling statistics
cat "$CGROUP_PATH/cpu.stat"
# nr_throttled = number of periods throttled
# throttled_usec = total time spent throttled

# Read CPU quota
cat "$CGROUP_PATH/cpu.max"
# "50000 100000" = 50% of one CPU per 100ms window
# "max 100000" = no quota

# Monitor in real time
watch -n1 "cat $CGROUP_PATH/memory.current"
```

### Kubernetes Resource Requests and Limits

```yaml
# Pod spec with proper resource configuration
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: myapp:latest
    resources:
      requests:
        memory: "256Mi"    # scheduler uses this for placement
        cpu: "250m"        # 250 millicores = 0.25 CPU
      limits:
        memory: "512Mi"    # cgroup memory.max = 536870912
        cpu: "500m"        # cgroup cpu.max = "50000 100000"
    # QoS class is determined by requests vs limits:
    # Guaranteed: requests == limits (best for latency-sensitive workloads)
    # Burstable: limits > requests (most common)
    # BestEffort: no requests or limits (evicted first under pressure)
```

```bash
# Check what resources a pod is actually using vs limits
kubectl top pod myapp --containers

# See QoS class
kubectl get pod myapp -o jsonpath='{.status.qosClass}'

# Find pods that have no resource limits (risky for production)
kubectl get pods --all-namespaces -o json | \
  jq -r '.items[] | select(.spec.containers[].resources.limits == null) |
  [.metadata.namespace, .metadata.name] | @tsv'

# Find pods close to their memory limit
kubectl top pods --all-namespaces | awk 'NR>1 {print $1, $2, $4}'
```

### Reading K8s Container cgroups on the Node

```bash
# On the K8s node, find a pod's cgroup
POD_UID="abc123-..."   # from kubectl get pod mypod -o jsonpath='{.metadata.uid}'
CONTAINER_ID="..."     # from kubectl get pod mypod -o jsonpath='{.status.containerStatuses[0].containerID}' | sed 's/containerd:\/\///'

# cgroup path structure (cgroups v2 with containerd):
# /sys/fs/cgroup/kubepods.slice/
#   kubepods-burstable.slice/                           ← QoS class
#     kubepods-burstable-pod<pod-uid>.slice/            ← Pod
#       cri-containerd-<container-id>.scope/            ← Container

CGROUP=$(find /sys/fs/cgroup/kubepods.slice -name "cri-containerd-${CONTAINER_ID:0:12}*" -type d 2>/dev/null)

# Check memory
cat "$CGROUP/memory.current"
cat "$CGROUP/memory.max"
cat "$CGROUP/memory.events"

# Check CPU throttling
cat "$CGROUP/cpu.stat" | grep -E "nr_throttled|throttled_usec"

# Compute throttle ratio (high = CPU limits too tight)
STATS=$(cat "$CGROUP/cpu.stat")
PERIODS=$(echo "$STATS" | grep nr_periods | awk '{print $2}')
THROTTLED=$(echo "$STATS" | grep nr_throttled | awk '{print $2}')
echo "Throttle ratio: $(echo "scale=2; $THROTTLED * 100 / $PERIODS" | bc)%"
```

### Docker Compose Resource Limits

```yaml
# docker-compose.yml with resource limits (v2+ format)
version: '3.8'
services:
  api:
    image: myapi:latest
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M

  database:
    image: postgres:15
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
        reservations:
          cpus: '1.0'
          memory: 2G
    environment:
      - POSTGRES_PASSWORD=secret
      # Tune postgres to respect memory limits:
      - POSTGRES_SHARED_BUFFERS=1GB
      - POSTGRES_WORK_MEM=64MB
```

### Detecting OOM Kills

```bash
# Check kernel OOM kill log
dmesg | grep -i "oom\|killed process" | tail -20
journalctl -k | grep -i oom | tail -20

# On K8s: check pod exit reason
kubectl get pod mypod -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# "OOMKilled" means the container hit its memory.max limit

# Get OOM kill count from cgroup
cat "$CGROUP/memory.events" | grep oom_kill

# Real-time OOM monitoring with dmesg -w
dmesg -w | grep -i oom

# Find the container that was OOM-killed
journalctl -k -b | grep "oom_kill_process\|Out of memory" | tail -20
# Shows which process was killed, its memory usage, and which cgroup triggered it

# Pod-level OOM history
kubectl get events --field-selector reason=OOMKilling --all-namespaces
```

### Resource Limit Patterns for Production

```bash
# Pattern 1: Set container ulimits for high-connection services
docker run \
  --ulimit nofile=65536:65536 \
  --ulimit nproc=65536:65536 \
  --memory=2g \
  --cpus=2 \
  nginx

# Pattern 2: Init container with separate limits for database migrations
# (prevents migration from consuming app container's CPU allocation)
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  initContainers:
  - name: migrate
    image: myapp:latest
    command: ["python", "manage.py", "migrate"]
    resources:
      limits:
        memory: "256Mi"
        cpu: "500m"
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        memory: "512Mi"
        cpu: "250m"
      limits:
        memory: "1Gi"
        cpu: "1000m"
EOF

# Pattern 3: Use LimitRange to enforce defaults in a namespace
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - default:
      memory: 512Mi
      cpu: 500m
    defaultRequest:
      memory: 256Mi
      cpu: 100m
    type: Container
EOF

# Pattern 4: Use ResourceQuota to limit total namespace consumption
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
EOF
```

---

## ⚠️ Gotchas & Pro Tips

- **`requests` affect scheduling; `limits` affect runtime:** In Kubernetes, the scheduler places pods based on `requests`. A pod with `requests.cpu: 100m` can be scheduled on a node with 100m available, even if it actually uses 2 CPU in production. Set realistic requests or you'll get noisy-neighbour problems.

- **Memory OOM is instant; CPU throttling is silent:** When a container hits its memory limit, it gets OOM-killed — a loud, obvious failure. When it hits its CPU quota, it just gets throttled — silently slow. CPU throttling is one of the most common undiagnosed performance issues in K8s workloads.

- **Page cache counts against memory limits:** Docker/K8s memory limits account for page cache (file-backed memory). A container reading large files will appear to use more memory than its heap size suggests. This is evictable memory, but it still counts toward the limit. If your container reads lots of data from disk, set memory limits with this headroom in mind.

- **`--memory-swap` default doubles the effective limit:** If you set `--memory=1g` without `--memory-swap`, Docker defaults to `--memory-swap=2g` — effectively allowing 1GB of swap. On EC2 instances without swap configured, this has no effect, but on instances with swap enabled, your container can use twice the RAM you intended before being OOM-killed.

- **cgroups v2 on Amazon Linux 2023:** AL2023 uses cgroups v2 by default, while AL2 uses v1. If you have Terraform or scripts that reference legacy v1 paths like `/sys/fs/cgroup/memory/`, they'll break on AL2023. Always check which version is active with `mount | grep cgroup2`.

- **`Guaranteed` QoS for latency-sensitive workloads:** Setting `requests == limits` gives a pod the `Guaranteed` QoS class, which means the node will never evict it for resource pressure and its cgroup gets the highest scheduling priority. Use this for databases, caches, and API services where latency matters.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
