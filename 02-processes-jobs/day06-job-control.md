# 🐧 Job Control — Linux Mastery

> **Shell job control lets you suspend, background, and manage multiple processes in a single terminal session — an essential survival skill during long-running migrations, builds, or live debugging sessions.**

## 📖 Concept

Every process in a Linux shell belongs to a **process group**, and each terminal session has a **foreground process group** (the one receiving keyboard input/signals) and zero or more **background process groups**. Shell job control is the mechanism for moving processes between these states and managing their lifecycle without opening new terminal windows.

When you press `Ctrl+Z`, the kernel sends `SIGTSTP` to the foreground process group, suspending it. The shell then regains control and assigns the suspended process a **job number** (e.g., `[1]`). From there you can resume it in the background with `bg`, bring it back to the foreground with `fg`, or let it die. This matters in production when you're running a long `rsync`, an Ansible playbook, or a database dump and need to temporarily do something else in the same shell.

**`nohup`** and **`disown`** solve the HUP problem. When your SSH session disconnects, the kernel sends `SIGHUP` to the session leader (your shell), which then forwards it to all child processes — killing your long-running job. `nohup` wraps the command to ignore SIGHUP before exec. `disown` removes an already-running job from the shell's job table so it won't receive HUP. In modern DevOps work, `tmux`/`screen` are preferred, but `nohup` and `disown` remain critical for quick-and-dirty detachment on servers where tmux isn't installed.

The `wait` builtin is the backbone of parallel shell scripting — spawn multiple background jobs and `wait` for all of them, harvesting exit codes and implementing proper error handling.

---

## 💡 Real-World Use Cases

- **Long-running database dump:** Start `pg_dump` in foreground, realize it'll take 2 hours, Ctrl+Z → `bg` → `disown` to detach safely without tmux.
- **Parallel AWS S3 sync:** Run 4 `aws s3 sync` commands in background simultaneously and `wait` for all to complete, then check exit codes.
- **Ansible playbook interrupted:** Suspend a running playbook with Ctrl+Z to quickly check a file, then `fg` to resume.
- **Build server script:** Run `make`, `test`, and `package` in parallel with `&` + `wait`, cutting CI pipeline time.
- **Remote script resilience:** Wrap long-running scripts with `nohup` so SSH timeouts or network drops don't abort the job.

---

## 🔧 Commands & Examples

### Basic Job Control

```bash
# Start a job in the background with &
sleep 300 &
# [1] 12345

# List all jobs in current shell session
jobs
# [1]+  Running    sleep 300 &

jobs -l   # Include PIDs
# [1]+ 12345 Running    sleep 300 &

jobs -p   # Only PIDs
# 12345

# Bring job 1 to foreground
fg %1
# sleep 300

# Suspend foreground job
# (press Ctrl+Z while it's running)
# [1]+  Stopped    sleep 300

# Resume job 1 in background
bg %1
# [1]+ sleep 300 &

# Kill job 1
kill %1
# [1]+  Terminated sleep 300
```

### Suspend and Resume a Running Process

```bash
# Practical: start a long rsync, suspend, check space, resume
rsync -avz /data/ backup-server:/data/
# ... Ctrl+Z
# [1]+  Stopped    rsync -avz /data/ backup-server:/data/

df -h /data     # Check disk space while rsync is paused

fg %1           # Resume rsync
```

### nohup — Survive SSH Disconnects

```bash
# Run command immune to SIGHUP, stdout/stderr → nohup.out
nohup long-running-script.sh &
# [1] 12345
# nohup: ignoring input and appending output to 'nohup.out'

# Redirect output explicitly
nohup ansible-playbook site.yml > /var/log/ansible-run.log 2>&1 &

# Check progress
tail -f /var/log/ansible-run.log

# nohup with explicit output capture
nohup bash -c 'for i in {1..100}; do echo $i; sleep 1; done' \
  > /tmp/counter.log 2>&1 &
echo "PID: $!"
```

### disown — Detach an Already-Running Job

```bash
# You forgot nohup but the job is already running
# Step 1: suspend it
# (Ctrl+Z)
# [1]+  Stopped    big-migration.sh

# Step 2: resume in background
bg %1

# Step 3: disown it — removes from shell job table, won't get SIGHUP
disown %1
# or by PID:
disown 12345

# disown -h: mark job to ignore SIGHUP but keep in job table
disown -h %1

# Verify it's gone from job table
jobs
# (empty)

# Verify process still runs
ps -p 12345
```

### wait — Parallel Execution with Exit Code Harvesting

```bash
#!/bin/bash
# Parallel S3 sync across 4 buckets

REGIONS=("us-east-1" "us-west-2" "eu-west-1" "ap-southeast-1")
declare -A PIDS

for region in "${REGIONS[@]}"; do
    aws s3 sync s3://mybucket-${region} /backup/${region}/ \
        > /tmp/sync-${region}.log 2>&1 &
    PIDS[$region]=$!
    echo "Started sync for $region (PID: ${PIDS[$region]})"
done

# Wait for all and check exit codes
FAILED=0
for region in "${REGIONS[@]}"; do
    wait ${PIDS[$region]}
    EXIT_CODE=$?
    if [ $EXIT_CODE -ne 0 ]; then
        echo "FAILED: $region sync exited with code $EXIT_CODE"
        cat /tmp/sync-${region}.log
        FAILED=1
    else
        echo "OK: $region sync completed"
    fi
done

[ $FAILED -eq 0 ] && echo "All syncs successful" || exit 1
```

### Advanced: Parallel with Concurrency Limit

```bash
#!/bin/bash
# Process 100 files but max 8 in parallel at a time

MAX_JOBS=8
CURRENT_JOBS=0

process_file() {
    local file="$1"
    # Simulate processing
    gzip -k "$file"
    echo "Processed: $file"
}

for file in /data/logs/*.log; do
    process_file "$file" &
    ((CURRENT_JOBS++))

    if (( CURRENT_JOBS >= MAX_JOBS )); then
        wait -n 2>/dev/null || wait   # Wait for any one job (-n requires bash 4.3+)
        ((CURRENT_JOBS--))
    fi
done

# Wait for remaining jobs
wait
echo "All files processed"
```

### Job Control with trap

```bash
#!/bin/bash
# Clean up background jobs on script exit/interrupt

PIDS=()

cleanup() {
    echo "Cleaning up background jobs..."
    for pid in "${PIDS[@]}"; do
        kill "$pid" 2>/dev/null
        wait "$pid" 2>/dev/null
    done
    echo "Done."
    exit 0
}

trap cleanup SIGINT SIGTERM EXIT

# Start background workers
for i in {1..3}; do
    sleep 100 &
    PIDS+=($!)
    echo "Started worker $i (PID: ${PIDS[-1]})"
done

echo "Press Ctrl+C to stop all workers"
wait   # Wait for all background jobs
```

### Process Group Management

```bash
# Send signal to entire process group (negative PID)
# This kills the process AND all its children
kill -- -$(ps -o pgid= -p 12345 | tr -d ' ')

# Start a command in a new process group (useful for clean teardown)
setsid long-command.sh &

# Check process group of a PID
ps -o pid,pgid,cmd -p 12345

# Kill all processes in a process group
pkill -g PGID
```

---

## ⚠️ Gotchas & Pro Tips

- **`disown` doesn't redirect stdout/stderr.** If the job is writing to the terminal, you'll still see output after disowning. Always redirect before disowning: redirect output first (`exec >file 2>&1` inside the script, or use `nohup` from the start).
- **`wait` without arguments waits for ALL background jobs** in the current shell. In long scripts spawning many jobs, track PIDs explicitly to `wait` on specific jobs and capture their exit codes.
- **`Ctrl+C` sends SIGINT to the foreground process group** — it kills the whole group, not just the parent. `Ctrl+Z` sends SIGTSTP only. This distinction matters when the parent has spawned children.
- **Job numbers reset per shell session.** `%1` in one SSH session means nothing in another. Always use PIDs for cross-session management (`ps aux | grep myprocess`).
- **`nohup.out` appends, never truncates.** On busy servers running many `nohup` jobs, `nohup.out` can grow huge. Always redirect to a named file: `nohup cmd > /tmp/cmd.log 2>&1 &`.
- **Pro tip — `wait -n` (bash 4.3+)** waits for the NEXT job to finish and returns its exit code. This is the building block for efficient parallel processing with proper error handling:

```bash
# Harvest results as they complete (bash 4.3+)
for pid in "${PIDS[@]}"; do
    wait -n
    echo "A job finished with exit: $?"
done
```

- **tmux is the production answer**, but job control is what you reach for when tmux isn't available or you're already mid-command. Know both.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
