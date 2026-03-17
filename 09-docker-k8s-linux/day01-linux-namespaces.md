# 🐧 Linux Namespaces — Linux Mastery

> **Understand Linux namespaces—the kernel primitives that make containers possible and the foundation for Kubernetes security.**

## 📖 Concept

Linux namespaces are the low-level Linux kernel feature that isolates system resources. Each container runs in its own namespaces, seeing only its own processes, filesystems, network interfaces, and users. There are eight namespace types: pid, net, mnt, uts, ipc, user, cgroup, and time. Understanding how they work reveals why containers are isolated but not truly sandboxed, and how to build minimal containers from scratch.

Most engineers interact with namespaces indirectly through Docker or Kubernetes, not realizing that `docker run --network=host` bypasses network namespace isolation, or that running containers as non-root requires user namespace mapping. The `unshare` command allows you to create and enter new namespaces manually—useful for debugging container behavior or testing namespace isolation.

For DevOps, the key insight is that namespaces are isolated from but not independent of the kernel. Containers share the host kernel; they're not lightweight VMs. This means all containers on a host compete for kernel resources, and a privilege escalation vulnerability in the kernel affects all containers.

---

## 💡 Real-World Use Cases

- **Container isolation verification**: Check what namespaces a running container uses with `nsenter` to debug networking or permission issues
- **Container escape detection**: Monitor for processes trying to exit namespaces (sign of an exploit attempt)
- **Rootless containers**: User namespace mapping allows non-root users to run containers (critical for security)
- **Network debugging in Kubernetes**: Use `nsenter` to enter a pod's network namespace and run network tools as if inside the pod
- **Building minimal containers**: Understand namespace requirements to build containers from scratch with minimal components
- **Multi-tenancy compliance**: Verify namespace isolation between customer workloads in shared clusters

---

## 🔧 Commands & Examples

### The Eight Linux Namespaces

```bash
# 1. PID namespace: isolates process IDs
#    Each container sees different PID 1 (init process)
#    Processes in different PID namespaces can have same PID

# 2. Network namespace: isolates network interfaces, routes, firewall rules
#    Each container sees own eth0, loopback, iptables

# 3. Mount namespace: isolates filesystems
#    Container filesystem separate from host
#    --bind mounts visible in container

# 4. UTS namespace: isolates hostname and domain name
#    Each container has own hostname

# 5. IPC namespace: isolates inter-process communication
#    Shared memory, message queues separate per container

# 6. User namespace: isolates user and group IDs
#    Container running as root (UID 0) maps to non-root on host
#    Critical for rootless containers

# 7. Cgroup namespace: isolates cgroup paths
#    Container sees cgroup as root of hierarchy

# 8. Time namespace: isolates system time (Linux 5.6+)
#    Container can have different system time than host

# Query namespace support
uname -r          # Kernel version (need 2.6.24+ for namespaces)
ls -la /proc/self/ns/      # Available namespaces on this system
```

### unshare: Creating Namespaces

```bash
# Create new namespace(s) and run command in it
unshare [options] <command>

# Create new PID namespace
unshare --pid --fork /bin/bash
# Now PID 1 is /bin/bash (not init), other processes invisible

# Create isolated network namespace
unshare --net /bin/bash
# ifconfig shows only loopback; no eth0

# Create isolated mount namespace
unshare --mount /bin/bash
# mount/unmount changes affect only this namespace

# Combined: simulate Docker container
unshare --pid --uts --mount --ipc --net --fork /bin/bash
# UID still 0 (root); for rootless, need --user

# List current namespaces
echo $$; ps aux | grep bash
# Show namespace IDs
ls -la /proc/$$/ns/
```

### nsenter: Entering Existing Namespaces

```bash
# Enter running container's namespaces for debugging
nsenter [options] -t <target_pid> <command>

# Find container PID
docker inspect mycontainer | grep Pid
# Output: "Pid": 12345

# Enter container's network namespace and run ifconfig
nsenter -t 12345 -n ifconfig
# Shows container's network interfaces

# Enter all namespaces (like being inside container)
nsenter -t 12345 -a /bin/bash
# Now exploring container as if inside it

# Common namespace shortcuts:
# -p, --pid=<ns>        PID namespace
# -n, --net=<ns>        Network namespace
# -m, --mount=<ns>      Mount namespace
# -u, --uts=<ns>        UTS namespace
# -i, --ipc=<ns>        IPC namespace
# -U, --user=<ns>       User namespace
# -a, --all              All namespaces

# Debug pod network issue in Kubernetes
kubectl get pod myapp -o wide  # Get node
ssh NODE
docker ps | grep myapp         # Get container ID
docker inspect CONTAINER_ID | grep Pid
nsenter -t CONTAINER_PID -n ip addr
nsenter -t CONTAINER_PID -n route -n
# Now you're looking at pod's network from inside
```

### /proc/PID/ns: Inspecting Namespace Membership

```bash
# Each process belongs to namespaces
ls -la /proc/self/ns/
# Output:
# lrwxrwxrwx cgroup -> cgroup:[4026531835]
# lrwxrwxrwx ipc -> ipc:[4026531839]
# lrwxrwxrwx mnt -> mnt:[4026531840]
# lrwxrwxrwx net -> net:[4026531956]
# lrwxrwxrwx pid -> pid:[4026531836]
# lrwxrwxrwx uts -> uts:[4026531838]
# lrwxrwxrwx user -> user:[4026531837]

# The inode numbers identify namespaces
# Same inode = same namespace (shared isolation)
# Different inode = different namespace (isolated)

# Check if two processes share namespaces
ls -L /proc/1234/ns/ | md5sum
ls -L /proc/5678/ns/ | md5sum
# Different checksums = different namespaces

# Find all processes in specific namespace
find /proc -name ns -prune -o -path '*/ns/net' -print0 | \
    xargs -0 ls -la | grep 'net:\[4026532509\]' | grep -oP '\d+' | sort -u

# Check if container is rootless (user namespace mapping)
grep ^/proc/PID/uid_map
# 0          0       1   -> Host UID 0 = Container UID 0 (not rootless)
# 0     1000000   65536  -> Host UID 1000000-1065535 = Container UID 0-65535 (rootless)
```

### PID Namespace: Container Init Problem

```bash
# Problem: Container with bash as PID 1 doesn't forward signals
FROM ubuntu:22.04
ENTRYPOINT ["/bin/bash", "app.sh"]

# app.sh:
#!/bin/bash
python worker.py &
wait    # Parent bash ignores SIGTERM, doesn't forward to python

# User sends SIGTERM (or Kubernetes terminates pod)
# Bash doesn't forward to python -> unclean shutdown

# Solution 1: Use tini (init replacement)
FROM ubuntu:22.04
RUN apt-get install tini
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["python", "worker.py"]

# Now tini is PID 1, forwards signals properly

# Solution 2: Use exec (replaces bash with python)
ENTRYPOINT ["/bin/sh", "-c", "exec python worker.py"]

# Solution 3: Handle signals in bash
#!/bin/bash
trap 'kill $PID 2>/dev/null; exit 0' SIGTERM
python worker.py &
PID=$!
wait $PID

# Verify PID 1 in running container
docker run --name myapp myimage:latest sleep 1000 &
docker exec myapp ps aux | head -5
# PID 1 should be your app, not bash/sh
```

### Network Namespace: Virtual Network Interfaces

```bash
# Create network namespace
ip netns add mynamespace

# View namespace interfaces (isolated from host)
ip netns exec mynamespace ip link show
# Only loopback, no eth0

# Create veth pair (virtual ethernet pair)
# One end in host, one in container (how Docker networking works)
ip link add veth-host type veth peer name veth-container

# Move one end to namespace
ip link set veth-container netns mynamespace

# Configure IP addresses
ip addr add 10.0.1.1/24 dev veth-host
ip netns exec mynamespace ip addr add 10.0.1.2/24 dev veth-container

# Bring up interfaces
ip link set veth-host up
ip netns exec mynamespace ip link set veth-container up

# Now host can ping container
ping 10.0.1.2

# Cleanup
ip link del veth-host  # Automatically deletes veth-container
ip netns delete mynamespace
```

### User Namespace: Rootless Containers

```bash
# Without user namespace: container root = host root (security risk)
docker run -u 0 ubuntu id
# uid=0(root) gid=0(root)

# With user namespace: container root = non-root on host
# Enable in Docker daemon config:
# /etc/docker/daemon.json:
# {
#   "userns-remap": "myuser:myuser"
# }

# systemctl restart docker

# Now:
docker run ubuntu id
# uid=0(root) gid=0(root) (inside container)
# But on host, processes run as myuser

# Verify with host-side ps
ps aux | grep containerized-process
# Shows process running as myuser, not root

# Check UID mapping
cat /proc/CONTAINER_PID/uid_map
# Shows mapping: container UID 0-65535 -> host UID 100000-165535

# Rootless Docker (even more secure)
dockerd --userns-remap=rootless
# Container root maps to unprivileged user on host
```

### Mount Namespace: Isolated Filesystems

```bash
# Create mount namespace
unshare --mount /bin/bash

# In the namespace, mount a new filesystem
mount -t tmpfs tmpfs /mnt/test
df | grep test

# In another terminal (host):
df | grep test
# Mount NOT visible (isolated)

# Exit namespace
exit

# Mount disappeared (cleanup)
df | grep test

# Docker uses mount namespaces for volume isolation:
# Host directory /data mounted at /app in container
# Is isolated: host can't see container modifications to /app
# But share content: changes visible to both
```

### Minimal Container from Scratch

```bash
#!/bin/bash
# Build minimal container manually with namespaces

# Create rootfs (minimal filesystem)
mkdir -p myroot/{bin,lib,lib64,proc,sys}
cp /bin/bash myroot/bin/
cp /lib/x86_64-linux-gnu/libc.so.6 myroot/lib/
cp /lib64/ld-linux-x86-64.so.2 myroot/lib64/

# Create isolated environment
unshare --pid --uts --mount --ipc --net --fork --root=myroot /bin/bash

# Now inside container:
# - PID 1 is /bin/bash
# - Hostname is container-specific
# - /proc and /sys are isolated
# - No network interfaces (except loopback if setup)

# This is essentially what Docker does (plus cgroups, seccomp, etc.)
```

### Kubernetes Network Namespace Debugging

```bash
# Common issue: pod can't reach service
# Debug by entering pod's network namespace

# Find pod on node
kubectl get pods -o wide | grep NODE_NAME
ssh NODE_NAME

# Find container PID
docker ps | grep POD_NAME
docker inspect CONTAINER_ID | grep Pid

# Enter pod's network namespace
nsenter -t CONTAINER_PID -n

# Now you're inside pod's network. Test:
ip addr                    # Pod's IP
ip route                   # Pod's routes
iptables -L -n             # Pod's firewall rules
ss -tlnp                   # Listening ports

# Check if service IP is reachable
curl SERVICE_CLUSTER_IP:PORT

# If not, check on host:
iptables -t nat -L -n | grep SERVICE_IP    # Service routing rules
```

---

## ⚠️ Gotchas & Pro Tips

- **Namespaces are not sandboxes**: Containers share the host kernel. A privilege escalation vulnerability in the kernel affects all containers. Namespaces provide isolation for convenience, not security against kernel exploits.

- **PID namespace doesn't hide kernel threads**: Kernel threads (kthreadd, etc.) visible from inside container. This is normal and not a security risk.

- **Network namespace without veth pairing**: If you create a network namespace without veth pairing, it's completely isolated (no connectivity). Docker handles pairing automatically.

- **Mount propagation**: By default, mounts in one namespace don't affect others. Use `mount --make-shared` or `rbind` for shared propagation.

- **User namespace IDs**: When user namespace is enabled, UID 0 in container maps to UID 100000+ on host. Files owned by container root are actually owned by unprivileged user on host—critical insight for security.

- **Container escape via namespace**: A process can attach to a new namespace if it has CAP_SYS_ADMIN in the target namespace's user namespace. Dropping CAP_SYS_ADMIN in `docker run --cap-drop=SYS_ADMIN` prevents this.

- **nsenter requires root**: To use nsenter, you need root or CAP_SYS_ADMIN. In Kubernetes, debug pods must be privileged to use nsenter.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
