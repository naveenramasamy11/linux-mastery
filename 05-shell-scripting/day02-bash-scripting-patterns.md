# 🔧 Day 02 — Bash Scripting Patterns That Scale

> Most Bash scripts break the moment something goes wrong. These patterns make yours production-ready.

If you've ever had a script silently skip errors, leave temp files behind after a crash, or become impossible to maintain at 200 lines — this guide is for you. We'll cover the patterns that distinguish a one-off hack from a script you'd actually put in production or a CI/CD pipeline.

---

## 1. The Reliable Script Header

Every serious Bash script should start with the same four lines:

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

# Usage: ./deploy.sh [--env prod|staging] [--dry-run]
```

### What each option does:

```bash
set -e        # Exit immediately if any command returns non-zero
set -u        # Treat unset variables as errors (no silent empty strings)
set -o pipefail  # Pipe failures propagate (without this: ls /noexist | wc -l returns 0)

# Combined shorthand:
set -euo pipefail
```

### Why `pipefail` matters so much:

```bash
# Without pipefail — this exits 0 (!!!)
cat /var/log/missing.log | grep "ERROR" | wc -l

# With pipefail — this correctly exits non-zero
set -o pipefail
cat /var/log/missing.log | grep "ERROR" | wc -l
# bash: /var/log/missing.log: No such file or directory
```

### The IFS change:

```bash
# Default IFS splits on space, tab, newline — causes bugs with filenames containing spaces
IFS=$'\n\t'   # Only split on newline and tab — much safer for loops over file lists
```

---

## 2. Functions: Structure and Conventions

### Basic function anatomy:

```bash
# Prefer function keyword for clarity; parens are optional but conventional
function check_dependencies() {
    local required_tools=("aws" "jq" "curl" "git")

    for tool in "${required_tools[@]}"; do
        if ! command -v "$tool" &>/dev/null; then
            echo "ERROR: Required tool '$tool' is not installed" >&2
            return 1   # Use return inside functions, not exit
        fi
    done

    echo "All dependencies satisfied."
}
```

### Return values — the right way:

```bash
# BAD: using echo to return data AND mixing with error messages
function get_instance_id() {
    echo "Fetching instance ID..."  # This becomes part of the return value!
    echo "i-1234567890abcdef0"
}

# GOOD: separate data output from log output
function get_instance_id() {
    echo "Fetching instance ID..." >&2  # Logs go to stderr
    local instance_id
    instance_id=$(aws ec2 describe-instances \
        --filters "Name=tag:Name,Values=${1}" \
        --query 'Reservations[0].Instances[0].InstanceId' \
        --output text)

    if [[ "$instance_id" == "None" || -z "$instance_id" ]]; then
        echo "ERROR: No instance found with name '$1'" >&2
        return 1
    fi

    echo "$instance_id"  # Only actual data goes to stdout
}

# Calling it:
INSTANCE_ID=$(get_instance_id "web-prod-01") || exit 1
echo "Found: $INSTANCE_ID"
```

### Passing arrays to functions:

```bash
# Arrays can't be passed directly — use nameref (Bash 4.3+)
function process_servers() {
    local -n servers=$1   # nameref: $1 is the variable NAME, not value

    for server in "${servers[@]}"; do
        echo "Processing: $server"
    done
}

PROD_SERVERS=("web-01" "web-02" "db-01")
process_servers PROD_SERVERS   # Pass the variable name, not the value
```

---

## 3. Error Handling with Traps

`trap` lets you register cleanup code that runs on specific signals or script exit — even on crashes.

### The cleanup trap pattern:

```bash
#!/usr/bin/env bash
set -euo pipefail

TMPDIR_WORK=$(mktemp -d)
LOCKFILE="/var/run/my-script.lock"

# Register cleanup BEFORE creating any resources
function cleanup() {
    local exit_code=$?   # Capture exit code before it gets overwritten

    echo "Cleaning up..." >&2
    rm -rf "$TMPDIR_WORK"
    rm -f "$LOCKFILE"

    if [[ $exit_code -ne 0 ]]; then
        echo "Script FAILED with exit code $exit_code" >&2
    fi

    exit $exit_code   # Re-exit with original code
}

trap cleanup EXIT    # Runs on ANY exit — normal, error, or signal
trap cleanup INT     # Ctrl+C
trap cleanup TERM    # kill command

# Now do your work — cleanup runs no matter what
touch "$LOCKFILE"
echo "Working in $TMPDIR_WORK..."
```

### Trapping for debugging:

```bash
# ERR trap fires on every failed command — useful for debugging
function on_error() {
    local exit_code=$?
    local line_number=$1
    echo "ERROR: Command failed with exit code $exit_code at line $line_number" >&2
    echo "  Script: $0" >&2
    echo "  Function stack: ${FUNCNAME[*]}" >&2
}

trap 'on_error $LINENO' ERR
```

### Available trap signals:

```bash
trap cmd EXIT    # Script exits (any reason)
trap cmd ERR     # Any command fails (with set -e)
trap cmd INT     # Ctrl+C (SIGINT)
trap cmd TERM    # kill / SIGTERM
trap cmd HUP     # Terminal closed / SIGHUP
trap cmd DEBUG   # Before every command (very verbose — use for debugging only)
```

---

## 4. getopts — Proper Argument Parsing

Don't parse `$1`, `$2` manually for anything non-trivial. Use `getopts`.

```bash
#!/usr/bin/env bash
set -euo pipefail

function usage() {
    cat <<EOF
Usage: $(basename "$0") [OPTIONS]

Deploy application to target environment.

Options:
  -e ENVIRONMENT  Target environment (dev|staging|prod)  [required]
  -r REGION       AWS region                             [default: us-east-1]
  -d              Dry run — print actions without executing
  -v              Verbose output
  -h              Show this help

Examples:
  $(basename "$0") -e prod -r us-west-2
  $(basename "$0") -e staging -d
EOF
}

# Defaults
ENV=""
REGION="us-east-1"
DRY_RUN=false
VERBOSE=false

# getopts: colon after letter = argument required; leading colon = silent error handling
while getopts ":e:r:dvh" opt; do
    case $opt in
        e) ENV="$OPTARG" ;;
        r) REGION="$OPTARG" ;;
        d) DRY_RUN=true ;;
        v) VERBOSE=true ;;
        h) usage; exit 0 ;;
        :) echo "ERROR: Option -$OPTARG requires an argument" >&2; usage; exit 1 ;;
        \?) echo "ERROR: Unknown option -$OPTARG" >&2; usage; exit 1 ;;
    esac
done

# Shift parsed options away, leaving positional args in $@
shift $((OPTIND - 1))

# Validate required args
if [[ -z "$ENV" ]]; then
    echo "ERROR: -e ENVIRONMENT is required" >&2
    usage
    exit 1
fi

if [[ ! "$ENV" =~ ^(dev|staging|prod)$ ]]; then
    echo "ERROR: Environment must be dev, staging, or prod" >&2
    exit 1
fi

$VERBOSE && echo "Deploying to $ENV in $REGION (dry_run=$DRY_RUN)"
```

---

## 5. Logging That Doesn't Suck

```bash
# Levelled logging with timestamps
LOG_LEVEL="${LOG_LEVEL:-INFO}"  # Can be overridden: LOG_LEVEL=DEBUG ./script.sh

function log() {
    local level=$1
    shift
    local message="$*"
    local timestamp
    timestamp=$(date '+%Y-%m-%dT%H:%M:%S')

    # Only print if level is high enough
    case "$LOG_LEVEL:$level" in
        DEBUG:*|INFO:INFO|INFO:WARN|INFO:ERROR|WARN:WARN|WARN:ERROR|ERROR:ERROR)
            printf '[%s] [%s] %s\n' "$timestamp" "$level" "$message" >&2
            ;;
    esac
}

# Usage:
log INFO "Starting deployment..."
log DEBUG "Connecting to $HOST:$PORT"
log WARN "Retry attempt $attempt of $max_retries"
log ERROR "Failed to connect — aborting"
```

### Log to file AND terminal:

```bash
# Tee output to both terminal and a log file
LOGFILE="/var/log/deploy-$(date +%Y%m%d-%H%M%S).log"
exec > >(tee -a "$LOGFILE") 2>&1
# Everything after this line goes to both stdout/stderr AND the logfile
```

---

## 6. Retry Logic for Flaky Commands

Essential for anything hitting a network or external API (hello, AWS API rate limits):

```bash
function retry() {
    local max_attempts=$1
    local delay=$2
    local cmd=("${@:3}")   # All remaining args are the command

    local attempt=1
    until "${cmd[@]}"; do
        if (( attempt >= max_attempts )); then
            echo "ERROR: Command failed after $max_attempts attempts: ${cmd[*]}" >&2
            return 1
        fi
        echo "Attempt $attempt failed. Retrying in ${delay}s..." >&2
        sleep "$delay"
        ((attempt++))
        delay=$((delay * 2))   # Exponential backoff
    done
}

# Usage:
retry 5 2 aws s3 cp ./artifact.zip s3://my-bucket/
retry 3 1 curl -sf https://my-service/health
```

---

## 7. Input Validation Patterns

```bash
# Check if running as root
function require_root() {
    if [[ $EUID -ne 0 ]]; then
        echo "ERROR: This script must be run as root" >&2
        exit 1
    fi
}

# Validate environment variables exist
function require_env() {
    local missing=()
    for var in "$@"; do
        if [[ -z "${!var:-}" ]]; then   # indirect variable reference
            missing+=("$var")
        fi
    done

    if [[ ${#missing[@]} -gt 0 ]]; then
        echo "ERROR: Required environment variables not set: ${missing[*]}" >&2
        exit 1
    fi
}

# Usage — fails fast if any of these aren't set:
require_env AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_DEFAULT_REGION

# Validate a file exists and is readable
function require_file() {
    if [[ ! -f "$1" ]]; then
        echo "ERROR: Required file not found: $1" >&2
        exit 1
    fi
    if [[ ! -r "$1" ]]; then
        echo "ERROR: Cannot read file: $1" >&2
        exit 1
    fi
}
```

---

## 8. Here Documents for Multi-line Content

```bash
# Generate config files inline — no escaping nightmare
cat > /etc/myapp/config.yaml <<EOF
environment: ${ENVIRONMENT}
database:
  host: ${DB_HOST}
  port: ${DB_PORT:-5432}
  name: ${DB_NAME}
region: ${AWS_DEFAULT_REGION}
EOF

# Use 'EOF (quoted) to disable variable expansion:
cat > /etc/myapp/template.sh <<'EOF'
#!/bin/bash
echo "My hostname is: $(hostname)"
echo "My IP is: ${MY_IP}"   # Literal — not expanded at heredoc creation
EOF
```

---

## 9. Script Locking (Prevent Concurrent Runs)

Critical for cron jobs or any script that shouldn't run twice simultaneously:

```bash
LOCKFILE="/var/run/$(basename "$0").lock"

# flock: atomic locking — no race conditions
(
    flock -n 9 || {
        echo "ERROR: Another instance is already running (lock: $LOCKFILE)" >&2
        exit 1
    }

    # Everything inside this block is under the lock
    echo "Doing exclusive work..."
    sleep 5

) 9>"$LOCKFILE"

# Alternatively, use a subshell-free approach with flock -c:
# flock -n "$LOCKFILE" bash -c 'echo doing exclusive work'
```

---

## 10. Putting It All Together — A Real-World Example

```bash
#!/usr/bin/env bash
# rotate-logs.sh — Rotate application logs on a running EC2 instance
set -euo pipefail
IFS=$'\n\t'

readonly SCRIPT_NAME=$(basename "$0")
readonly LOG_DIR="/var/log/myapp"
readonly ARCHIVE_BUCKET="${ARCHIVE_BUCKET:-}"
readonly MAX_AGE_DAYS=7

TMPWORK=$(mktemp -d)

function cleanup() { rm -rf "$TMPWORK"; }
trap cleanup EXIT

function log() { echo "[$(date +%H:%M:%S)] [$1] ${*:2}" >&2; }

function usage() {
    echo "Usage: $SCRIPT_NAME -b S3_BUCKET [-d DAYS] [-h]"
}

while getopts ":b:d:h" opt; do
    case $opt in
        b) BUCKET="$OPTARG" ;;
        d) MAX_AGE_DAYS="$OPTARG" ;;
        h) usage; exit 0 ;;
        :) echo "Option -$OPTARG requires argument" >&2; exit 1 ;;
        \?) echo "Unknown option: -$OPTARG" >&2; exit 1 ;;
    esac
done

BUCKET="${BUCKET:-$ARCHIVE_BUCKET}"
[[ -z "$BUCKET" ]] && { echo "ERROR: -b S3_BUCKET required" >&2; exit 1; }

log INFO "Rotating logs older than ${MAX_AGE_DAYS} days from $LOG_DIR"

find "$LOG_DIR" -name "*.log" -mtime +"$MAX_AGE_DAYS" | while read -r logfile; do
    archive="$TMPWORK/$(basename "$logfile").gz"
    log INFO "Archiving: $logfile"
    gzip -c "$logfile" > "$archive"
    retry 3 2 aws s3 cp "$archive" "s3://${BUCKET}/logs/$(date +%Y/%m/%d)/$(basename "$archive")"
    rm "$logfile"
    log INFO "Archived and removed: $logfile"
done

log INFO "Log rotation complete."
```

---

## Pro Tips and Gotchas

```bash
# GOTCHA: [[ vs [
# Always use [[ ]] in Bash — it handles spaces, empty strings, and patterns better
[[ -z "$VAR" ]]        # Safe
[ -z "$VAR" ]          # Works but older, avoid in Bash scripts

# GOTCHA: Quoting arrays
FILES=("file one.txt" "file two.txt")
cp "${FILES[@]}" /dest/   # Correct — preserves spaces
cp ${FILES[@]} /dest/     # WRONG — word-splits on spaces

# PRO TIP: Use declare -r for constants
declare -r MAX_RETRIES=5  # Bash will error if you try to reassign

# PRO TIP: Check if script is sourced or executed
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"   # Only run main() when executed directly, not when sourced
fi

# PRO TIP: Use shellcheck to catch bugs before they bite you
# shellcheck ./my-script.sh
# Install: apt install shellcheck / brew install shellcheck
# Also available as VS Code extension and GitHub Action
```

### AWS-Specific Patterns

```bash
# Wait for AWS resource with retry and timeout
function wait_for_instance() {
    local instance_id=$1
    local timeout=${2:-300}
    local start=$SECONDS

    log INFO "Waiting for $instance_id to be running..."
    while true; do
        local state
        state=$(aws ec2 describe-instances \
            --instance-ids "$instance_id" \
            --query 'Reservations[0].Instances[0].State.Name' \
            --output text)

        [[ "$state" == "running" ]] && break

        if (( SECONDS - start > timeout )); then
            log ERROR "Timed out waiting for $instance_id (state: $state)"
            return 1
        fi

        log INFO "State: $state. Waiting..."
        sleep 10
    done

    log INFO "$instance_id is running."
}

# Get EC2 metadata inside an instance (IMDSv2)
TOKEN=$(curl -sf -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -sf -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -sf -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/placement/availability-zone)
```

---

## Quick Reference

| Pattern | Why It Matters |
|---|---|
| `set -euo pipefail` | Fail fast on any error |
| `trap cleanup EXIT` | Always clean up temp files |
| `local` in functions | Avoid polluting global scope |
| `>&2` for log messages | Keep stdout clean for piping |
| `"${array[@]}"` | Always quote array expansion |
| `getopts` | Proper argument parsing |
| `flock` | Prevent concurrent runs |
| `shellcheck` | Static analysis before shipping |

---

**Next:** [Day 03 — awk & sed Mastery](day03-awk-sed-mastery.md) — transform data at the speed of thought with real-world one-liners.
