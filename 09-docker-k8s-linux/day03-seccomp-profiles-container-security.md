# 🐧 seccomp Profiles & Container Security — Linux Mastery

> **seccomp is the syscall firewall for your containers — without it, a container escape is a single kernel vulnerability away from full host compromise.**

## 📖 Concept

`seccomp` (secure computing mode) is a Linux kernel feature that restricts the system calls a process can make. In its modern BPF-based form (seccomp-BPF), you define a filter program that runs for every syscall, deciding whether to allow it, return an error, or kill the process. Docker and Kubernetes apply seccomp profiles to containers as a defence-in-depth measure — even if an attacker achieves arbitrary code execution inside a container, they cannot call syscalls outside the whitelist to escalate privileges or escape to the host.

The attack surface reduction is significant. A standard Linux system exposes ~380+ syscalls. A typical containerised application might legitimately need only 50-70 of them. Every unnecessary syscall is a potential exploit vector — kernel vulnerabilities often manifest as specific syscall bugs (e.g., `ptrace`, `keyctl`, `clone` with CLONE_NEWUSER, `perf_event_open`). A well-crafted seccomp profile closes these doors entirely.

Docker ships with a default seccomp profile that blocks ~44 dangerous syscalls. Kubernetes did not apply any seccomp by default until 1.27, when `RuntimeDefault` became the default. Understanding how to build custom profiles — by auditing actual syscall usage rather than guessing — is the professional approach. The toolchain for this is `strace`, `bpftrace`, or the `oci-seccomp-bpf-hook`.

seccomp profiles compose with other Linux security mechanisms: **namespaces** (isolation), **capabilities** (privilege granularity), and **AppArmor/SELinux** (MAC policies). A container with all four applied is substantially harder to escape than one with just namespaces.

---

## 💡 Real-World Use Cases

- Apply a custom seccomp profile to a Kubernetes pod running a web API to block all syscalls except the ~60 it actually uses
- Block `ptrace` in all containers to prevent one compromised container from attaching to sibling containers
- Use `strace` + `oci-seccomp-bpf-hook` to generate a minimal seccomp profile from actual observed syscall usage
- Audit existing EKS pod security posture by checking which pods have `securityContext.seccompProfile` set
- Harden an Alpine-based microservice to a 40-syscall whitelist, passing CIS Kubernetes Benchmark checks

---

## 🔧 Commands & Examples

### Understanding Seccomp Modes

```bash
# Check if seccomp is enabled in the kernel
grep CONFIG_SECCOMP /boot/config-$(uname -r)
# CONFIG_SECCOMP=y
# CONFIG_SECCOMP_FILTER=y

# Check seccomp status of a running process
grep Seccomp /proc/PID/status
# Seccomp: 0 = not restricted
# Seccomp: 1 = strict mode (only read/write/exit/sigreturn)
# Seccomp: 2 = filter mode (BPF profile applied)

# Check seccomp mode on Docker containers
for pid in $(docker inspect --format '{{.State.Pid}}' $(docker ps -q)); do
  name=$(docker inspect --format '{{.Name}}' $(ls -la /proc/$pid/exe 2>/dev/null | head -1) 2>/dev/null)
  mode=$(grep Seccomp /proc/$pid/status 2>/dev/null | awk '{print $2}')
  echo "PID $pid: Seccomp mode $mode"
done
```

### Docker Default seccomp Profile

```bash
# Docker's default profile blocks ~44 syscalls including:
# add_key, keyctl, request_key    — kernel keyring
# ptrace (partially)               — process tracing
# clone (with CLONE_NEWUSER)       — user namespaces from inside container
# mount, umount2                   — filesystem mounts
# reboot, kexec_load               — system control
# nfsservctl, get_mempolicy        — admin ops
# perf_event_open                  — perf monitoring (can leak info)

# Run a container with Docker's default profile (this is the default)
docker run --security-opt seccomp=/etc/docker/seccomp.json alpine sh

# Run WITHOUT seccomp (dangerous — only for debugging)
docker run --security-opt seccomp=unconfined alpine sh

# Check what profile a container is using
docker inspect <container_id> | jq '.[].HostConfig.SecurityOpt'

# Download Docker's default seccomp profile for inspection/modification
curl -L https://raw.githubusercontent.com/docker/engine/master/profiles/seccomp/default.json \
  -o docker-default-seccomp.json
```

### Building a Custom seccomp Profile

The profile is a JSON file with a default action and a list of syscall rules.

```bash
# Minimal whitelist profile structure
cat > /tmp/custom-seccomp.json << 'EOF'
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
      "names": [
        "read", "write", "open", "close", "stat", "fstat", "lstat",
        "poll", "lseek", "mmap", "mprotect", "munmap", "brk",
        "rt_sigaction", "rt_sigprocmask", "rt_sigreturn",
        "ioctl", "pread64", "pwrite64", "readv", "writev",
        "access", "pipe", "select", "sched_yield", "mremap",
        "msync", "mincore", "madvise", "dup", "dup2", "pause",
        "nanosleep", "getitimer", "alarm", "setitimer", "getpid",
        "sendfile", "socket", "connect", "accept", "sendto", "recvfrom",
        "sendmsg", "recvmsg", "shutdown", "bind", "listen",
        "getsockname", "getpeername", "socketpair", "setsockopt", "getsockopt",
        "clone", "fork", "vfork", "execve", "exit", "wait4",
        "kill", "uname", "fcntl", "flock", "fsync", "fdatasync",
        "truncate", "ftruncate", "getdents", "getcwd", "chdir",
        "rename", "mkdir", "rmdir", "creat", "link", "unlink",
        "symlink", "readlink", "chmod", "fchmod", "chown", "fchown",
        "lchown", "umask", "gettimeofday", "getrlimit", "getrusage",
        "sysinfo", "times", "getuid", "getgid", "getgroups",
        "setuid", "setgid", "geteuid", "getegid", "setpgid",
        "getppid", "getpgrp", "setsid", "capget", "capset",
        "rt_sigsuspend", "sigaltstack", "utime", "mknod",
        "personality", "setrlimit", "sync", "gettid",
        "futex", "sched_getaffinity", "set_thread_area", "get_thread_area",
        "exit_group", "epoll_create", "epoll_ctl", "epoll_wait",
        "set_tid_address", "clock_gettime", "clock_getres", "clock_nanosleep",
        "statfs", "fstatfs", "tgkill", "openat", "getdents64",
        "newfstatat", "readlinkat", "accept4", "dup3",
        "pipe2", "epoll_create1", "prlimit64", "sendmmsg", "recvmmsg",
        "getrandom", "memfd_create", "statx"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
EOF

# Apply to a Docker container
docker run --security-opt seccomp=/tmp/custom-seccomp.json myapp:latest

# Apply with additional blocked syscalls on top of default
# Start from Docker's default profile and add extra restrictions
```

### Generating a Profile from Actual Syscall Usage (strace method)

```bash
# Step 1: Run your application under strace to capture all syscalls
strace -f -e trace=all -o /tmp/strace-output.txt docker run myapp:latest

# Step 2: Extract unique syscall names
grep -oP '(?<=^|\()[\w]+(?=\()' /tmp/strace-output.txt | sort -u > /tmp/needed-syscalls.txt
cat /tmp/needed-syscalls.txt

# Step 3: Build a whitelist profile from the list
python3 << 'PYEOF'
import json

with open('/tmp/needed-syscalls.txt') as f:
    syscalls = [line.strip() for line in f if line.strip()]

profile = {
    "defaultAction": "SCMP_ACT_ERRNO",
    "architectures": ["SCMP_ARCH_X86_64"],
    "syscalls": [
        {
            "names": syscalls,
            "action": "SCMP_ACT_ALLOW"
        }
    ]
}

with open('/tmp/generated-profile.json', 'w') as f:
    json.dump(profile, f, indent=2)

print(f"Generated profile with {len(syscalls)} allowed syscalls")
PYEOF

# Step 4: Test with the generated profile (use SCMP_ACT_LOG to monitor first)
# Change defaultAction to "SCMP_ACT_LOG" to log blocked calls without blocking
# Then switch to "SCMP_ACT_ERRNO" once verified
```

### Kubernetes seccomp Profiles

```bash
# Kubernetes seccomp profile types:
# - RuntimeDefault: use the container runtime's default profile (Docker default, containerd default)
# - Localhost: use a profile file from the node's filesystem
# - Unconfined: no seccomp restriction

# Apply RuntimeDefault to a pod (recommended baseline)
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
          - ALL
EOF

# Apply a custom Localhost profile
# First: copy profile to all nodes at /var/lib/kubelet/seccomp/profiles/custom.json
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secure-app-custom
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/custom.json
  containers:
  - name: app
    image: myapp:latest
EOF

# Check which pods have seccomp profiles configured
kubectl get pods -A -o json | jq -r '
  .items[] |
  "\(.metadata.namespace)/\(.metadata.name): " +
  (.spec.securityContext.seccompProfile.type // "NONE")'

# Pods with no seccomp profile are a security risk
kubectl get pods -A -o json | jq -r '
  .items[] |
  select(.spec.securityContext.seccompProfile == null) |
  "\(.metadata.namespace)/\(.metadata.name)"'
```

### Capabilities and seccomp Together

```bash
# Drop ALL capabilities then add only what's needed
# Combined with seccomp, this is a strong security posture

cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hardened-nginx
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
    runAsNonRoot: true
    runAsUser: 101     # nginx user
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 8080    # non-privileged port
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
        add:
          - NET_BIND_SERVICE   # only if binding to ports < 1024
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: var-run
      mountPath: /var/run
    - name: var-cache-nginx
      mountPath: /var/cache/nginx
  volumes:
  - name: tmp
    emptyDir: {}
  - name: var-run
    emptyDir: {}
  - name: var-cache-nginx
    emptyDir: {}
EOF
```

### Auditing with audit2seccomp / bpftrace

```bash
# Method 1: Use audit log to collect denied syscalls during testing
# Set profile to SCMP_ACT_LOG (kernel 4.14+) — logs without blocking

# Modify profile defaultAction to log
cat > /tmp/log-profile.json << 'EOF'
{
  "defaultAction": "SCMP_ACT_LOG",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": []
}
EOF

docker run --security-opt seccomp=/tmp/log-profile.json myapp:latest
# Then check audit log or kernel log for logged syscalls:
journalctl -k | grep "audit: type=1326"   # SECCOMP log entries

# Method 2: bpftrace to capture syscalls from a running container
# Get the container's PID
PID=$(docker inspect --format '{{.State.Pid}}' mycontainer)

# Trace all syscalls from that PID
bpftrace -e "tracepoint:raw_syscalls:sys_enter /pid == $PID/ { @[ksym(args->id)] = count(); }"
# Run for 30 seconds, then Ctrl-C to see the sorted list

# Method 3: Using oci-seccomp-bpf-hook (Podman/CRI-O integration)
# This automatically generates a profile while a container runs
podman run --annotation io.containers.trace-syscall=of:/tmp/profile.json myapp:latest
# profile.json is generated with the actual syscalls used
```

### Testing Your Profile

```bash
# Verify seccomp is blocking what it should

# Test 1: try to call a blocked syscall from inside a container
docker run --security-opt seccomp=/tmp/custom-seccomp.json alpine sh -c "
  # Try ptrace (should be blocked)
  strace ls 2>&1 || echo 'ptrace blocked as expected'
"

# Test 2: use seccomp-tools to inspect a profile
pip install seccomp-tools
seccomp-tools dump /tmp/custom-seccomp.json

# Test 3: check effective seccomp from inside a container
docker run --security-opt seccomp=/tmp/custom-seccomp.json alpine sh -c "
  cat /proc/self/status | grep Seccomp
"
# Should show: Seccomp: 2 (filter mode active)

# Test 4: verify the container cannot gain new capabilities
docker run --security-opt seccomp=/tmp/custom-seccomp.json alpine sh -c "
  capsh --print
"
```

---

## ⚠️ Gotchas & Pro Tips

- **RuntimeDefault vs custom:** `RuntimeDefault` is the containerd/Docker default profile — good baseline but not maximally restrictive. Custom profiles based on actual syscall auditing can cut the allowed set by 50-70% further.
- **SCMP_ACT_LOG before SCMP_ACT_ERRNO:** Always test new profiles with `SCMP_ACT_LOG` first (available on kernel 4.14+). Log mode records denied calls without actually blocking them, so you can identify what your application legitimately needs before switching to ERRNO.
- **Multi-arch profiles:** Always list multiple architecture entries (`SCMP_ARCH_X86_64`, `SCMP_ARCH_X86`) or your profile may not apply on 32-bit syscall paths.
- **seccomp + capabilities interact:** Some capabilities allow bypassing certain syscall restrictions. Always drop capabilities (especially `CAP_SYS_ADMIN`, `CAP_NET_ADMIN`, `CAP_SYS_PTRACE`) in addition to applying seccomp.
- **Kubernetes defaults changed in 1.27:** From Kubernetes 1.27+, `RuntimeDefault` is the default seccomp profile when `--seccomp-default` kubelet flag is enabled (it was opt-in before). Check your EKS version and node configuration.
- **Don't block signals:** Many profiles accidentally block `rt_sigaction`, `rt_sigprocmask`, or `rt_sigreturn`, which breaks signal handling. Always include the signal-related syscalls in your whitelist.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
