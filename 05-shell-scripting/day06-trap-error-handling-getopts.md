# 🐧 trap, Error Handling & getopts — Linux Mastery

> **Write bash scripts that fail safely, clean up after themselves, and handle arguments like a proper CLI tool.**

## 📖 Concept

Production bash scripts fail differently than development scripts. In dev, you notice the error and fix it. In production at 3am, the script silently half-finishes and leaves the system in an inconsistent state — a partially restored database, a mounted filesystem with no entry in fstab, an S3 upload that stalled mid-way. Robust error handling is the difference between scripts that are safe to run in automation and scripts that are ticking time bombs.

**`set` options** are the first line of defense: `set -e` exits on any error, `set -u` treats unset variables as errors, `set -o pipefail` makes pipelines fail if any command fails (not just the last), and `set -x` enables debug tracing. In production scripts, `set -euo pipefail` is the standard preamble.

**`trap`** lets you register a function to run when the script receives a signal or exits. `trap cleanup EXIT` runs your cleanup function whether the script exits normally, due to an error, or due to a signal. This is how you ensure temporary files get deleted, mounts get unmounted, and locks get released even when things go wrong.

**`getopts`** is the POSIX-compliant way to parse command-line options (like `-f filename -v --help`). It handles option parsing correctly — including combined flags (`-vf`), missing arguments, and unknown flags — without the fragile manual `if [ "$1" == "-f" ]` approach that breaks on edge cases.

---

## 💡 Real-World Use Cases

- Ensure temp files, lock files, and mounts are cleaned up even if a deployment script crashes mid-run
- Catch and log errors in CI/CD pipeline scripts so failures produce actionable messages
- Parse complex CLI flags for infrastructure automation scripts (backup scripts, deploy scripts, migration tools)
- Prevent partially-applied Terraform/Ansible runs from leaving systems in unknown states
- Build scripts that gracefully handle SIGTERM from Kubernetes pod shutdown or AWS instance termination

---

## 🔧 Commands & Examples

### set Options — Defensive Script Configuration

```bash
#!/bin/bash
# Professional bash preamble — use at the top of all production scripts

set -e          # exit immediately if a command exits with non-zero status
set -u          # treat unset variables as errors (prevents typo bugs)
set -o pipefail # fail if any command in a pipe fails (not just the last)
set -E          # ERR trap is inherited by shell functions (needed with set -e)

# Shorthand (equivalent to above)
set -euo pipefail

# Debug mode — shows each command before execution
set -x

# Enable in a function temporarily
debug_function() {
    set -x
    # ... commands ...
    set +x  # turn off debug mode
}

# Check for unset variable (without set -u)
: "${REQUIRED_VAR:?ERROR: REQUIRED_VAR must be set}"
: "${DATABASE_URL:?ERROR: DATABASE_URL environment variable is required}"

# Default value if unset (safe with set -u)
LOG_LEVEL="${LOG_LEVEL:-INFO}"
TIMEOUT="${TIMEOUT:-30}"
ENVIRONMENT="${1:-production}"
```

### trap — Cleanup on Exit and Signals

```bash
#!/bin/bash
set -euo pipefail

# ─── Cleanup function ───────────────────────────────────────────────────────
cleanup() {
    local exit_code=$?
    echo "[$(date)] Cleanup triggered (exit code: $exit_code)"
    
    # Remove temp files
    [[ -f "$TMPFILE" ]] && rm -f "$TMPFILE"
    
    # Unmount if mounted
    if mountpoint -q "${MOUNT_DIR}"; then
        umount "${MOUNT_DIR}" || echo "WARNING: Failed to unmount ${MOUNT_DIR}"
    fi
    
    # Release a lock file
    [[ -f "$LOCKFILE" ]] && rm -f "$LOCKFILE"
    
    # Send failure notification
    if [ $exit_code -ne 0 ]; then
        echo "Script FAILED with exit code $exit_code" | \
            aws sns publish --topic-arn "$SNS_TOPIC" --message file:///dev/stdin 2>/dev/null || true
    fi
    
    exit $exit_code
}

# Register cleanup to run on any exit (normal, error, or signal)
trap cleanup EXIT

# ─── Signal handlers ─────────────────────────────────────────────────────────
handle_sigterm() {
    echo "[$(date)] Received SIGTERM — shutting down gracefully..."
    SHUTDOWN_REQUESTED=true
    # Give the main loop a chance to finish the current iteration
}

handle_sigint() {
    echo "[$(date)] Received SIGINT (Ctrl+C) — aborting"
    exit 130  # 128 + SIGINT(2)
}

trap handle_sigterm SIGTERM
trap handle_sigint SIGINT

# ─── Main script ─────────────────────────────────────────────────────────────
TMPFILE=$(mktemp /tmp/deploy-XXXXXX.tmp)
LOCKFILE="/var/lock/deploy.lock"
MOUNT_DIR="/mnt/deploy"
SNS_TOPIC="arn:aws:sns:us-east-1:123456789:alerts"
SHUTDOWN_REQUESTED=false

# Acquire lock (prevents concurrent runs)
if ! mkdir "$LOCKFILE" 2>/dev/null; then
    echo "ERROR: Another deploy is already running (lock: $LOCKFILE)"
    exit 1
fi

echo "Starting deployment..."
echo "Temp file: $TMPFILE"

# ... rest of deployment ...
echo "Deployment complete"
# cleanup runs automatically here via EXIT trap
```

### Error Handling Patterns

```bash
#!/bin/bash
set -euo pipefail

# ─── Error handler with context ──────────────────────────────────────────────
error_handler() {
    local exit_code=$?
    local line_number=$1
    local command=$2
    echo "ERROR on line ${line_number}: command '${command}' exited with code ${exit_code}" >&2
    echo "Stack trace:" >&2
    # Print bash call stack
    local i=0
    while caller $i; do
        ((i++))
    done >&2
}
trap 'error_handler $LINENO "$BASH_COMMAND"' ERR

# ─── Retry function ──────────────────────────────────────────────────────────
retry() {
    local max_attempts=$1
    local delay=$2
    shift 2
    local cmd=("$@")
    
    local attempt=1
    until "${cmd[@]}"; do
        if [ $attempt -ge $max_attempts ]; then
            echo "ERROR: Command failed after $max_attempts attempts: ${cmd[*]}" >&2
            return 1
        fi
        echo "Attempt $attempt failed. Retrying in ${delay}s..." >&2
        sleep "$delay"
        ((attempt++))
    done
    return 0
}

# Usage
retry 3 5 aws s3 cp /data/backup.tar.gz s3://my-bucket/
retry 5 10 kubectl rollout status deployment/myapp -n production

# ─── check_command helper ─────────────────────────────────────────────────────
check_prereqs() {
    local missing=()
    for cmd in aws kubectl helm jq; do
        command -v "$cmd" &>/dev/null || missing+=("$cmd")
    done
    if [ ${#missing[@]} -gt 0 ]; then
        echo "ERROR: Missing required commands: ${missing[*]}" >&2
        exit 1
    fi
}

# ─── Conditional error handling (disable set -e locally) ─────────────────────
# Sometimes you expect a command to fail
check_db_exists() {
    local db_name=$1
    # Temporarily allow failure
    if mysql -u root -e "use $db_name" 2>/dev/null; then
        echo "Database $db_name exists"
        return 0
    else
        echo "Database $db_name does not exist"
        return 1
    fi
}

# Or use explicit error handling
result=$(aws s3 ls s3://my-bucket/ 2>&1) && bucket_exists=true || bucket_exists=false

# ─── Logging with timestamps ─────────────────────────────────────────────────
LOG_FILE="/var/log/deploy-$(date +%Y%m%d_%H%M%S).log"

log() {
    local level=$1
    shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $*" | tee -a "$LOG_FILE"
}

log INFO "Starting deployment to production"
log WARN "Skipping health check (override flag set)"
log ERROR "Database migration failed"
```

### getopts — Parsing Command-Line Arguments

```bash
#!/bin/bash
set -euo pipefail

# ─── Usage function ──────────────────────────────────────────────────────────
usage() {
    cat <<EOF
Usage: $(basename "$0") [OPTIONS] <environment>

Deploy the application to the specified environment.

Arguments:
  environment       Target environment (dev|staging|production)

Options:
  -f, --force       Force deployment even if health checks fail
  -i, --image TAG   Docker image tag to deploy (default: latest)
  -r, --replicas N  Number of replicas (default: 2)
  -d, --dry-run     Show what would be done without doing it
  -v, --verbose     Enable verbose output
  -h, --help        Show this help message

Examples:
  $(basename "$0") production
  $(basename "$0") -i v1.2.3 -r 3 staging
  $(basename "$0") --force --dry-run production

EOF
    exit "${1:-0}"
}

# ─── Default values ──────────────────────────────────────────────────────────
FORCE=false
IMAGE_TAG="latest"
REPLICAS=2
DRY_RUN=false
VERBOSE=false

# ─── getopts parsing ─────────────────────────────────────────────────────────
# getopts optstring name
# Optstring: list of valid option chars. ':' after a char means it takes an argument.
# Leading ':' enables silent error handling (you handle the errors).

while getopts ":fi:r:dvh" opt; do
    case $opt in
        f) FORCE=true ;;
        i) IMAGE_TAG="$OPTARG" ;;
        r) REPLICAS="$OPTARG" ;;
        d) DRY_RUN=true ;;
        v) VERBOSE=true ;;
        h) usage 0 ;;
        :) echo "ERROR: Option -$OPTARG requires an argument" >&2; usage 1 ;;
        \?) echo "ERROR: Unknown option -$OPTARG" >&2; usage 1 ;;
    esac
done

# Shift past the parsed options to get positional arguments
shift $((OPTIND - 1))

# Validate positional argument
if [ $# -lt 1 ]; then
    echo "ERROR: environment argument is required" >&2
    usage 1
fi

ENVIRONMENT="$1"
shift  # consume the environment argument

# Validate environment value
case "$ENVIRONMENT" in
    dev|staging|production) ;;
    *) echo "ERROR: Invalid environment '$ENVIRONMENT'. Must be dev, staging, or production" >&2; usage 1 ;;
esac

# Validate numeric argument
if ! [[ "$REPLICAS" =~ ^[0-9]+$ ]] || [ "$REPLICAS" -lt 1 ]; then
    echo "ERROR: --replicas must be a positive integer" >&2
    exit 1
fi

# ─── Verbose output helper ────────────────────────────────────────────────────
debug() {
    $VERBOSE && echo "[DEBUG] $*" >&2 || true
}

# ─── Main logic ──────────────────────────────────────────────────────────────
debug "Environment: $ENVIRONMENT"
debug "Image tag: $IMAGE_TAG"
debug "Replicas: $REPLICAS"
debug "Force: $FORCE"
debug "Dry run: $DRY_RUN"

if $DRY_RUN; then
    echo "[DRY RUN] Would deploy image:${IMAGE_TAG} with ${REPLICAS} replicas to ${ENVIRONMENT}"
    echo "[DRY RUN] kubectl set image deployment/app app=myrepo/app:${IMAGE_TAG} -n ${ENVIRONMENT}"
    exit 0
fi

echo "Deploying ${IMAGE_TAG} to ${ENVIRONMENT} with ${REPLICAS} replicas..."
kubectl set image deployment/app \
    app="myrepo/app:${IMAGE_TAG}" \
    -n "${ENVIRONMENT}"

kubectl scale deployment/app \
    --replicas="${REPLICAS}" \
    -n "${ENVIRONMENT}"
```

### Complete Production Script Template

```bash
#!/bin/bash
# ═══════════════════════════════════════════════════════════════════════════════
# backup.sh — Production database backup script
# Usage: ./backup.sh [-e env] [-d database] [-r retention_days] [-v]
# ═══════════════════════════════════════════════════════════════════════════════

set -euo pipefail

# ─── Constants ────────────────────────────────────────────────────────────────
readonly SCRIPT_NAME=$(basename "$0")
readonly SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)
readonly TIMESTAMP=$(date +%Y%m%d_%H%M%S)
readonly TMPDIR_BASE="/tmp"

# ─── Defaults ─────────────────────────────────────────────────────────────────
ENVIRONMENT="production"
DATABASE="appdb"
RETENTION_DAYS=30
VERBOSE=false
TMPDIR=""
LOCKFILE=""

# ─── Logging ─────────────────────────────────────────────────────────────────
log() { echo "[$(date '+%H:%M:%S')] [$1] ${*:2}" >&2; }
info() { log INFO "$@"; }
warn() { log WARN "$@"; }
error() { log ERROR "$@"; }
debug() { $VERBOSE && log DEBUG "$@" || true; }

# ─── Cleanup ─────────────────────────────────────────────────────────────────
cleanup() {
    local rc=$?
    debug "Cleanup called with exit code $rc"
    [[ -n "$TMPDIR" && -d "$TMPDIR" ]] && rm -rf "$TMPDIR"
    [[ -n "$LOCKFILE" && -f "$LOCKFILE" ]] && rm -f "$LOCKFILE"
    if [ $rc -ne 0 ]; then
        error "Script failed with exit code $rc"
    fi
    exit $rc
}
trap cleanup EXIT
trap 'error "Interrupted"; exit 130' INT TERM

# ─── Argument Parsing ─────────────────────────────────────────────────────────
usage() {
    echo "Usage: $SCRIPT_NAME [-e env] [-d database] [-r days] [-v] [-h]"
    exit "${1:-0}"
}

while getopts ":e:d:r:vh" opt; do
    case $opt in
        e) ENVIRONMENT="$OPTARG" ;;
        d) DATABASE="$OPTARG" ;;
        r) RETENTION_DAYS="$OPTARG" ;;
        v) VERBOSE=true ;;
        h) usage 0 ;;
        :) error "Option -$OPTARG requires an argument"; usage 1 ;;
        \?) error "Unknown option: -$OPTARG"; usage 1 ;;
    esac
done

# ─── Main ────────────────────────────────────────────────────────────────────
info "Starting backup: env=$ENVIRONMENT db=$DATABASE retention=${RETENTION_DAYS}d"

# Acquire lock
LOCKFILE="/var/lock/${SCRIPT_NAME}.lock"
if ! ( set -o noclobber; echo $$ > "$LOCKFILE" ) 2>/dev/null; then
    error "Already running (lock: $LOCKFILE held by PID $(cat $LOCKFILE 2>/dev/null))"
    exit 1
fi

# Create temp workspace
TMPDIR=$(mktemp -d "${TMPDIR_BASE}/${SCRIPT_NAME}-XXXXXX")
debug "Temp dir: $TMPDIR"

BACKUP_FILE="${TMPDIR}/${DATABASE}_${TIMESTAMP}.sql.gz"

# Backup
info "Dumping database..."
pg_dump "${DATABASE}" | gzip > "$BACKUP_FILE"
info "Backup size: $(du -sh "$BACKUP_FILE" | cut -f1)"

# Upload to S3
info "Uploading to S3..."
aws s3 cp "$BACKUP_FILE" \
    "s3://backups-${ENVIRONMENT}/${DATABASE}/$(basename $BACKUP_FILE)" \
    --sse aws:kms

# Clean up old backups
info "Pruning backups older than ${RETENTION_DAYS} days..."
aws s3 ls "s3://backups-${ENVIRONMENT}/${DATABASE}/" | \
    awk '{print $4}' | \
    while read key; do
        age=$(( ($(date +%s) - $(date -d "${key:(-19):8}" +%s 2>/dev/null || echo 0)) / 86400 ))
        if [ "$age" -gt "$RETENTION_DAYS" ]; then
            debug "Deleting old backup: $key (age: ${age}d)"
            aws s3 rm "s3://backups-${ENVIRONMENT}/${DATABASE}/${key}"
        fi
    done

info "Backup completed successfully"
```

---

## ⚠️ Gotchas & Pro Tips

- **`set -e` doesn't catch all failures:** Commands in `if` conditions, `||`, `&&`, and `!` expressions are explicitly exempt from `set -e`. A common bug: `set -e; result=$(failing_command)` — this WILL exit. But `set -e; if failing_command; then ...` — this will NOT exit on the failure.

- **`set -u` breaks `"${array[@]}"` on empty arrays in bash < 4.4:** An empty array with `set -u` throws "unbound variable". Use `"${array[@]+"${array[@]}"}"` for safe empty-array expansion.

- **`trap EXIT` fires even on `exit` calls:** This is intentional and correct. The `$?` in the cleanup function reflects the exit code of the `exit` command or the last failed command. Use `local exit_code=$?` as the first line of cleanup.

- **getopts vs getopt:** `getopts` (built-in, POSIX) handles short options only (`-f`, `-v`). `getopt` (external, GNU) handles long options (`--force`, `--verbose`). For long options, use `getopt` or a manual `case` loop over `"$@"`.

- **Lockfile race conditions:** `if [ ! -f "$LOCKFILE" ]; then touch $LOCKFILE; fi` has a TOCTOU race. Use `mkdir "$LOCKFILE"` or `( set -o noclobber; echo $$ > "$LOCKFILE" )` for atomic lock acquisition.

- **`set -x` leaks secrets:** If your script handles passwords, tokens, or private keys, `set -x` will print them to stderr. Either avoid `set -x` in functions that handle secrets, or temporarily disable it with `{ set +x; } 2>/dev/null` around sensitive operations.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
