# 🐧 Linux Signals Deep Dive — Linux Mastery

> **Master SIGTERM, SIGKILL, SIGHUP, and graceful shutdown patterns—critical knowledge for containers, Kubernetes, and production reliability.**

## 📖 Concept

Signals are the primary inter-process communication mechanism in Unix. Understanding them deeply is non-negotiable for container orchestration, graceful deployments, and debugging production issues. Many engineers know signals exist but don't grasp the fundamental difference between SIGTERM and SIGKILL, or why Docker's default `stop` behavior often doesn't work as expected.

At the kernel level, signals interrupt process execution—they're asynchronous notifications. A running process can install signal handlers (via `trap` in bash, `signal()` in C) to catch most signals and execute cleanup code. SIGKILL (9) and SIGSTOP (19) are special: they cannot be caught or ignored, allowing the kernel ultimate control to forcibly stop processes.

For DevOps and SRE, this means the difference between a graceful shutdown (connections closed, state flushed, locks released) and a hard kill (data corruption, orphaned resources). Kubernetes relies entirely on this: `terminationGracePeriodSeconds` is the window for SIGTERM before SIGKILL. A 30-second grace period is worthless if your application doesn't handle SIGTERM.

---

## 💡 Real-World Use Cases

- **Kubernetes pod termination**: Understanding why your app doesn't shut down gracefully (not catching SIGTERM), leading to `timeout waiting for graceful shutdown` warnings
- **Docker container stop**: `docker stop` sends SIGTERM, waits 10 seconds, then SIGKILL. Knowing this prevents data loss in databases
- **Zero-downtime deployments**: Using SIGHUP for config reload without restart (nginx, systemd)
- **Zombie process cleanup**: Identifying why `ps aux` shows defunct processes and how parent process signal handling causes it
- **Shell script reliability**: Ensuring your deploy script's background jobs are terminated on Ctrl+C or script exit
- **Container PID 1 problem**: Why your web app doesn't respond to Ctrl+C in Docker (bash isn't forwarding signals correctly)

---

## 🔧 Commands & Examples

### Signal Types & Behaviors

```bash
# List all available signals
kill -l

# Standard signals you need to know:
# SIGTERM (15): Graceful termination. Catchable. Default for 'kill'.
# SIGKILL (9):  Immediate termination. Cannot be caught. Use as last resort.
# SIGSTOP (19): Pause process. Cannot be caught. Opposite: SIGCONT (18).
# SIGHUP (1):   Hang up. Traditionally used for config reload.
# SIGINT (2):   Interrupt (Ctrl+C). Catchable.

# Send SIGTERM to process by PID
kill 1234

# Send SIGTERM to process by name
killall nginx

# Send SIGTERM to processes matching pattern
pkill -f 'python worker'

# Send specific signal
kill -SIGTERM 1234    # Explicit signal name
kill -15 1234         # Signal number
kill -TERM 1234       # Short name

# Forcibly kill (only use after SIGTERM fails)
kill -SIGKILL 1234
kill -9 1234
```

### Differences: kill vs killall vs pkill

```bash
# kill: sends signal to process by PID
kill 1234              # SIGTERM to PID 1234
kill -9 1234           # SIGKILL to PID 1234
kill -SIGHUP 1234      # SIGHUP (config reload) to PID 1234

# killall: sends signal to all processes matching exact name
killall nginx          # SIGTERM to all processes named 'nginx'
killall -9 nginx       # SIGKILL to all named 'nginx'

# pkill: sends signal to processes matching pattern (more flexible)
pkill -f 'python.*worker'     # Matches pattern, not just name
pkill -u root nginx           # Only to processes owned by root
pkill -9 -f 'defunct'         # Kill all matching 'defunct'

# Comparison: which ones does SIGTERM reach?
pkill -SIGTERM myapp          # All processes matching 'myapp'
pkill -SIGTERM -u username    # All processes by user 'username'
pkill -SIGTERM --oldest       # Oldest matching process
pkill -SIGTERM --newest       # Newest matching process
```

### Signal Handling in Bash Scripts (trap)

```bash
#!/bin/bash
set -euo pipefail

# Graceful shutdown pattern: trap SIGTERM and SIGINT
cleanup() {
    echo "Received termination signal, cleaning up..."
    # Kill background jobs
    jobs -p | xargs -r kill 2>/dev/null || true
    # Close database connections
    # Remove temp files
    rm -f /tmp/script-$$.lock
    echo "Cleanup complete, exiting."
    exit 0
}

# Register signal handlers
trap cleanup SIGTERM SIGINT

# Lockfile pattern: ensure only one instance runs
LOCKFILE="/tmp/script-$$.lock"
if ! mkdir "${LOCKFILE}" 2>/dev/null; then
    echo "Script already running (lock exists)"
    exit 1
fi
trap "rmdir ${LOCKFILE}" EXIT

# Background work
long_running_task &
TASK_PID=$!

# Wait for background process
wait $TASK_PID
```

### SIGTERM vs SIGKILL: Why It Matters for Containers

```bash
# Container graceful shutdown scenario:
# 1. 'docker stop' sends SIGTERM
# 2. Container receives SIGTERM, should cleanup and exit
# 3. If still running after terminationGracePeriodSeconds, K8s sends SIGKILL
# 4. Process killed hard, no cleanup possible

# Wrong: application ignores SIGTERM
# Result: Kubernetes waits 30 seconds, then force-kills
# Problem: database transactions aren't committed, cache isn't flushed

# Right: application catches SIGTERM, flushes state, exits
# Result: graceful shutdown in <1 second

# Test in Dockerfile:
FROM python:3.11
COPY app.py .
CMD ["python", "app.py"]

# app.py must do this:
import signal
import sys

def handle_sigterm(sig, frame):
    print("Received SIGTERM, flushing state...")
    # flush caches, close connections
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_sigterm)
# Now 'docker stop' works gracefully
```

### SIGHUP for Config Reload (Nginx Example)

```bash
# Many daemons reload config on SIGHUP without restarting
# Benefits: zero downtime, no connection drops

# Nginx: reload config without restart
nginx -t                    # Test config syntax
nginx -s reload             # OR send SIGHUP
kill -HUP $(cat /var/run/nginx.pid)  # Same thing

# Monitor: config reloads happen without process restart
ps aux | grep nginx
# Both master and worker processes stay running

# In systemd service file:
[Unit]
Description=Nginx HTTP Server
[Service]
ExecStart=/usr/sbin/nginx
ExecReload=/bin/kill -HUP $MAINPID    # Send HUP on 'systemctl reload nginx'
Type=forking
```

### Zombie Processes: What Causes Them?

```bash
# A zombie process:
# - Has exited but parent hasn't reaped it (via wait/waitpid)
# - Shows as <defunct> in ps output
# - Occupies a process table slot, consumes no resources

# Cause: Parent process not handling SIGCHLD or not calling wait()
# Example (bad parent):
#!/bin/bash
(sleep 100 & )  # Background child
# Parent exits immediately without waiting for child to finish
# Child becomes zombie (parent is init, which reaps zombies)

# Or in long-running parent:
#!/bin/bash
while true; do
    (heavy_task &)      # Start background task
    sleep 1
    # Not calling wait! Tasks become zombies after they exit
done

# Fix: wait for background processes
#!/bin/bash
(heavy_task &)
PID=$!
wait $PID    # Reap the child when it exits

# Or use job control:
#!/bin/bash
wait            # Waits for all background jobs

# Check for zombies
ps aux | grep defunct

# Kill zombie's parent (parent will be init, which reaps it)
ps -ppid <parent_pid>  # Find children of zombie parent
kill -9 <parent_pid>
```

### Container PID 1 Problem

```bash
# Problem: Docker image runs bash as PID 1
FROM ubuntu:22.04
ENTRYPOINT ["/bin/bash", "script.sh"]

# script.sh:
#!/bin/bash
python worker.py &
wait

# Issue: Bash as PID 1 doesn't forward signals to children
# User presses Ctrl+C (SIGINT), bash doesn't propagate to python worker
# Ctrl+C only kills bash, not the worker
# docker stop sends SIGTERM to bash, bash doesn't forward to worker

# Solution 1: Use exec (replaces bash with python, python becomes PID 1)
#!/bin/bash
exec python worker.py

# Solution 2: Use tini or dumb-init (small init that forwards signals)
FROM ubuntu:22.04
RUN apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["python", "worker.py"]

# tini becomes PID 1, forwards SIGTERM to python
# Ctrl+C works, graceful shutdown works

# Solution 3: Use sh -c with exec
ENTRYPOINT ["/bin/sh", "-c", "exec python worker.py"]

# Test: Verify PID 1 is what you expect
docker run myimage ps aux | head -5
# PID 1 should be your app, not bash
```

### Signal Handling in Go Applications

```go
package main

import (
	"context"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	// Create a channel for signals
	sigChan := make(chan os.Signal, 1)
	
	// Register for SIGTERM and SIGINT
	signal.Notify(sigChan, syscall.SIGTERM, syscall.SIGINT)
	
	// Start background work
	go startServer()
	
	// Wait for signal
	sig := <-sigChan
	println("Received signal:", sig)
	
	// Graceful shutdown with timeout
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	
	// Close connections, flush state
	shutdown(ctx)
	
	println("Shutdown complete")
}

func startServer() {
	// Server code
}

func shutdown(ctx context.Context) {
	// Cleanup
	select {
	case <-ctx.Done():
		println("Shutdown timeout exceeded")
	default:
		println("Graceful shutdown succeeded")
	}
}
```

### Kubernetes Pod Termination Flow

```yaml
# deployment.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  terminationGracePeriodSeconds: 30  # Window for SIGTERM before SIGKILL
  containers:
  - name: app
    image: myapp:latest
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]  # 5sec delay before SIGTERM sent
```

```bash
# Termination sequence:
# 1. User issues 'kubectl delete pod'
# 2. Pod gets 'Terminating' status
# 3. preStop hook runs (if defined)
# 4. SIGTERM sent to container PID 1
# 5. Wait up to terminationGracePeriodSeconds
# 6. If still running, send SIGKILL
# 7. Pod removed

# What the app should do:
# - Catch SIGTERM (signal 15)
# - Stop accepting new connections
# - Finish existing requests (should be <30sec)
# - Close databases, flush caches
# - Exit cleanly
```

---

## ⚠️ Gotchas & Pro Tips

- **SIGKILL is unstoppable**: You cannot catch, block, or ignore SIGKILL (9) or SIGSTOP (19). Use them as last resort. A process stuck in uninterruptible sleep (`ps` state `D`) cannot be killed even with SIGKILL (it's waiting for I/O).

- **Bash PID 1 doesn't forward signals**: If bash runs as PID 1 in a container, it won't forward signals to child processes. Use `exec` or `tini` to fix this. This is the #1 reason Docker containers don't shut down gracefully.

- **Signal race conditions**: Between receiving SIGTERM and exiting, the process might receive multiple SIGTERMs (Docker sends SIGKILL after timeout). Your handler must be idempotent.

- **Zombie cleanup**: Init (PID 1) automatically reaps zombies. Non-init processes that spawn many children should call `wait()` or `waitpid()` to avoid zombie accumulation.

- **SIGHUP convention**: Not all daemons use SIGHUP for reload. Check the man page. Some use SIGUSR1 or SIGUSR2 for custom behavior.

- **Signal handlers in multi-threaded apps**: Signal handlers run in arbitrary threads. This is dangerous in Python, Go, Rust—use language-specific signal handling (channels, goroutines, etc.).

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
