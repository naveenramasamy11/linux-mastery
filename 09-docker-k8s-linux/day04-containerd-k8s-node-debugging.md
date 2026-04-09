# 🐧 containerd Internals & K8s Node Debugging — Linux Mastery

> **When `kubectl` can't reach a pod and the control plane is healthy, you need to go one layer deeper — to containerd and the Linux host — to find out why containers are failing to start, crashing, or being evicted.**

## 📖 Concept

Modern Kubernetes dropped Docker as the default container runtime in favor of **containerd**, a CNCF-graduated runtime that implements the Container Runtime Interface (CRI). Understanding containerd's architecture is essential for debugging nodes that `kubectl` can't see inside.

containerd runs as a daemon (`/usr/bin/containerd`) and manages the full container lifecycle: image pulling, snapshot management (using **overlayfs** or **btrfs**), OCI bundle creation, and container process lifecycle via **runc** (or other OCI runtimes like **crun**, **kata-containers**). The Kubelet communicates with containerd over a Unix socket (`/run/containerd/containerd.sock`) using the CRI gRPC protocol.

The **containerd namespace** model is important: `k8s.io` namespace contains all Kubernetes-managed containers, while `moby` is used by Docker. Tools like `ctr` (containerd CLI) and `crictl` (CRI-compatible tool, works with any CRI runtime) operate at different abstraction levels. `crictl` speaks CRI (what kubelet sees), while `ctr` speaks the containerd API directly.

When a pod is stuck in `ContainerCreating`, `CrashLoopBackOff`, or disappears from `kubectl`, the debugging workflow moves from K8s → CRI → containerd → runc → Linux namespaces and cgroups.

---

## 💡 Real-World Use Cases

- **Pod stuck in ContainerCreating:** Image pull failures, snapshot mount errors, or CNI plugin failures — none visible in `kubectl describe`, but all visible in `journalctl -u containerd` and `crictl`.
- **CrashLoopBackOff investigation:** `crictl logs` shows logs even for crashed containers that have been restarted too many times.
- **Node pressure and eviction:** `crictl stats` shows per-container CPU/memory directly from cgroup accounting — the same data the kubelet eviction manager uses.
- **Image garbage collection issues:** `ctr -n k8s.io images ls` shows what containerd actually has cached vs what K8s thinks exists.
- **Runtime class debugging:** When using Kata Containers or gVisor, debugging requires understanding which OCI runtime handled the container and what shim process is running.

---

## 🔧 Commands & Examples

### crictl — CRI-Level Debugging

```bash
# crictl config (point to containerd socket)
cat /etc/crictl.yaml
# runtime-endpoint: unix:///run/containerd/containerd.sock
# image-endpoint: unix:///run/containerd/containerd.sock
# timeout: 30
# debug: false

# Or set via env
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/containerd/containerd.sock

# List all pods (from CRI perspective)
crictl pods
# POD ID              CREATED             STATE     NAME                          NAMESPACE       ATTEMPT
# 8d9f123abc         2 hours ago         Ready     nginx-deployment-xxx-yyy      default         0

# List pods with detailed output
crictl pods --output json | jq '.items[].status.state'

# List containers (all states including failed/stopped)
crictl ps -a
# CONTAINER ID   IMAGE                      CREATED         STATE     NAME        ATTEMPT  POD ID
# abc123def456   nginx@sha256:...           2 hours ago     Running   nginx       0        8d9f123abc

# Get container details
crictl inspect abc123def456

# Get container logs (works even after CrashLoop restarts)
crictl logs abc123def456
crictl logs --tail 100 abc123def456
crictl logs --since 1h abc123def456

# Execute command in running container
crictl exec -it abc123def456 /bin/sh
crictl exec abc123def456 cat /etc/nginx/nginx.conf

# Container stats (live, from cgroup)
crictl stats
crictl stats abc123def456

# Pod stats
crictl statsp

# Pull an image (test if image registry is reachable)
crictl pull nginx:latest

# List images
crictl images
crictl images | grep nginx

# Remove unused images
crictl rmi --prune

# Image info
crictl inspecti nginx:latest
```

### ctr — containerd Native CLI

```bash
# List all namespaces (k8s.io = kubernetes containers)
ctr namespaces ls
# NAME      LABELS
# k8s.io
# moby

# List containers in k8s.io namespace
ctr -n k8s.io containers ls

# List running tasks (actual Linux processes)
ctr -n k8s.io tasks ls
# TASK                    PID      STATUS
# abc123def456           12345    RUNNING

# List images in k8s.io namespace
ctr -n k8s.io images ls

# Inspect a container
ctr -n k8s.io containers info abc123def456

# Get OCI spec (full container configuration)
ctr -n k8s.io containers info abc123def456 | jq '.Spec'

# Attach to a running task
ctr -n k8s.io tasks attach abc123def456

# Execute command in task
ctr -n k8s.io tasks exec --exec-id debug1 abc123def456 /bin/sh

# List snapshots (overlay layers)
ctr -n k8s.io snapshots ls | head -20

# Content store (pulled image layers)
ctr -n k8s.io content ls | head -10
```

### containerd Service Debugging

```bash
# Check containerd service status
systemctl status containerd
journalctl -u containerd -f             # Follow logs
journalctl -u containerd --since "1 hour ago"
journalctl -u containerd -p err         # Errors only

# containerd config
cat /etc/containerd/config.toml

# Check containerd socket
ls -la /run/containerd/containerd.sock
# srw-rw---- 1 root root 0 Apr  9 10:00 /run/containerd/containerd.sock

# Check containerd version
containerd --version
ctr version

# Restart containerd (carefully — all running containers continue via shims)
systemctl restart containerd
# Note: containers keep running! containerd reconnects to existing shims

# Check for stuck shim processes
ps aux | grep containerd-shim
# root  12345  0.0  0.0  /usr/bin/containerd-shim-runc-v2 ...

# Force cleanup of a stuck shim
kill -9 <shim_pid>  # Last resort — will kill the container too
```

### K8s Node-Level Debugging

```bash
# SSH into a worker node (via bastion or Session Manager on AWS)
# All of the following run on the NODE directly

# Check kubelet status and logs
systemctl status kubelet
journalctl -u kubelet -f
journalctl -u kubelet --since "30 min ago" | grep -i "error\|fail\|warn"

# Kubelet config
cat /var/lib/kubelet/config.yaml

# Check node disk/memory pressure triggers
# Kubelet eviction thresholds
grep -A5 "evictionHard\|evictionSoft" /var/lib/kubelet/config.yaml

# Check kubelet-reported node conditions
kubectl get node <nodename> -o json | jq '.status.conditions'

# Check cgroup v2 setup
mount | grep cgroup
ls /sys/fs/cgroup/
# If cgroup v2: single unified hierarchy under /sys/fs/cgroup/

# Find cgroup for a specific pod
# First get pod UID from kubectl
POD_UID=$(kubectl get pod mypod -o jsonpath='{.metadata.uid}')
ls /sys/fs/cgroup/kubepods/besteffort/pod${POD_UID}/

# Check CPU/memory limits enforced by cgroup
cat /sys/fs/cgroup/kubepods/besteffort/pod${POD_UID}/*/cpu.max
# 50000 100000  → 50ms per 100ms = 500m CPU limit

cat /sys/fs/cgroup/kubepods/besteffort/pod${POD_UID}/*/memory.max
# 268435456  → 256Mi memory limit

# Real-time resource usage from cgroup
cat /sys/fs/cgroup/kubepods/besteffort/pod${POD_UID}/*/memory.current
cat /sys/fs/cgroup/kubepods/besteffort/pod${POD_UID}/*/cpu.stat
```

### Overlayfs Debugging (Container Image Layers)

```bash
# containerd stores snapshots in /var/lib/containerd/
du -sh /var/lib/containerd/
du -sh /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/

# Find the overlay mount for a running container
# Get PID from task
ctr -n k8s.io tasks ls | grep abc123def456
# abc123def456  12345  RUNNING

# Find mounts for that PID
cat /proc/12345/mounts | grep overlay
# overlay / overlay rw,relatime,lowerdir=.../sha256:abc:sha256:def,...,upperdir=.../upper,workdir=.../work 0 0

# View active overlay mounts on the node
mount | grep overlay | wc -l   # Number of running containers
mount | grep overlay | head -3

# Check for overlay mount errors
dmesg | grep -i "overlay\|overlayfs" | tail -20

# Check inode/disk usage (overlay can exhaust inodes)
df -i /var/lib/containerd/
```

### Pod Startup Failure Diagnosis

```bash
# Full debugging workflow for a stuck pod:

# Step 1: kubectl view
kubectl describe pod failing-pod -n mynamespace
# Look for: Events section, Conditions, ContainerStatuses

# Step 2: CRI-level check
crictl pods | grep failing-pod-prefix
# Get the POD ID
crictl inspectp <pod-id>
# Check "state", "reason", "message"

# Step 3: Check containers in the pod
crictl ps -a | grep <pod-id>
# Even stopped/failed containers appear here

# Step 4: Get logs from failed container
crictl logs <container-id>   # Current instance
crictl logs --previous <container-id>   # Previous terminated instance (if supported)

# Step 5: Check containerd logs for this container
journalctl -u containerd | grep <container-id> | tail -50

# Step 6: Check kubelet for CNI/volume errors
journalctl -u kubelet | grep <pod-id> | grep -i "error\|fail"

# Step 7: Check systemd for OOM
journalctl -k | grep -i "oom\|killed" | grep <container-pid>
# Or:
dmesg | grep -i "oom_kill"
```

---

## ⚠️ Gotchas & Pro Tips

- **`crictl` and `ctr` are for debugging only — never use them to manage containers that K8s manages.** Creating or deleting containers via `ctr`/`crictl` bypasses the Kubelet and will cause reconciliation chaos. Use `kubectl` for management, CRI tools only for read-only inspection.
- **Restarting containerd does NOT kill running containers.** containerd connects to long-running `containerd-shim` processes which stay alive independently. This means `systemctl restart containerd` is safe during a running workload — use it to fix containerd config issues.
- **Image GC can race with pod scheduling.** containerd's garbage collector runs on a schedule and can delete an image while a pod is being scheduled but before it's pulled. This causes `ErrImageNeverPull` or pull races. Check containerd config `gc.schedule` and `gc.delay`.
- **cgroup v1 vs v2 changes the path structure.** On cgroup v2 systems (Ubuntu 22.04+, Amazon Linux 2023), the unified hierarchy is at `/sys/fs/cgroup/` with no sub-hierarchies. On cgroup v1, each subsystem has its own mount point (`/sys/fs/cgroup/memory/`, `/sys/fs/cgroup/cpu/`). Scripts checking cgroup limits must handle both.
- **Pro tip — `nsenter` for direct container namespace access:**

```bash
# Get PID of containerd task
PID=$(ctr -n k8s.io tasks ls | grep mycontainer | awk '{print $2}')

# Enter all namespaces of that container
nsenter -t $PID --mount --uts --ipc --net --pid -- /bin/bash
# Now you're inside the container's namespaces but with HOST tools!
# Great for debugging containers that don't have sh/bash inside them
```

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
