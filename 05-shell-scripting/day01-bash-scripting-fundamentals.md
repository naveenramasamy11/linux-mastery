# 🐧 Bash Scripting Fundamentals — Linux Mastery

> **Master production-grade bash scripting: error handling, trap cleanup, parameter expansion, and patterns that SREs actually use.**

## 📖 Concept

Bash scripting is the glue holding DevOps infrastructure together. From CI/CD pipeline scripts to deploy automation to infrastructure-as-code wrappers, your bash skills directly impact reliability and maintainability. The difference between a fragile script that fails silently and a production-grade script that handles errors gracefully is mastery of a few key patterns.

Most bash scripts fail in predictable ways: undefined variables cause silent errors, a command fails but the script continues, intermediate artifacts aren't cleaned up on error. These aren't bash failures—they're failures of the engineer to use proper patterns. Every production script needs `set -euo pipefail`, trap handlers for cleanup, and proper error messages.

Parameter expansion is powerful but underutilized. The difference between `$var`, `${var}`, `${var:-default}`, `${var:?error}`, and `${var%%pattern}` is the difference between brittle and robust scripts. Here-documents eliminate escaping nightmares. Argument parsing with `getopts` beats manual parsing with regexes. These patterns aren't optional—they're the foundation of production automation.

---

## 💡 Real-World Use Cases

- **Zero-downtime deployment scripts**: Deploy new versions while maintaining service, with automatic rollback on failure
- **Database backup automation**: Cron-friendly scripts that compress, verify, and upload backups to S3, with cleanup on failure
- **Container health checks**: Startup scripts that verify dependencies, wait for services, and report detailed errors
- **Log rotation and archival**: Scripts that compress old logs, upload to cloud storage, and clean local copies
- **Infrastructure provisioning**: Terraform/CloudFormation wrappers that handle state, lock files, and partial failures
- **Monitoring and alerting**: Scripts that parse logs, detect anomalies, and notify teams
- **Secret management**: Scripts that rotate API keys, update configuration, and notify stakeholders

---

## 🔧 Commands & Examples

### set -euo pipefail: The Foundation

```bash
#!/bin/bash
# WRONG: Script continues even when commands fail
cp /large/file /dest
echo "Done"  # Printed even if cp fails

# RIGHT: Add set -euo pipefail to every production script
#!/bin/bash
set -euo pipefail

cp /large/file /dest
echo "Done"  # Only printed if cp succeeds

# Breaking it down:
# set -e:    Exit immediately if any command exits with non-zero
# set -u:    Treat undefined variables as errors
# set -o pipefail: If any command in a pipe fails, the whole pipe fails

# Example of why each matters:
# Without -e: missing file doesn't stop script
# Without -u: $UNDEFINED_VAR expands to empty string silently
# Without pipefail: tar | gzip fails silently if tar fails (gzip succeeds)

# Always put this at the top
#!/bin/bash
set -euo pipefail
IFS=$'\n\t'  # Also good: strict word splitting

trap cleanup EXIT    # Trap defined below
```

### trap: Graceful Cleanup and Error Handling

```bash
#!/bin/bash
set -euo pipefail

# Define cleanup function
cleanup() {
    local exit_code=$?
    
    # Cleanup actions
    echo "Cleaning up..."
    rm -f /tmp/script-$$.lock
    kill $BACKGROUND_PID 2>/dev/null || true
    
    # Report result
    if [ $exit_code -ne 0 ]; then
        echo "Script failed with exit code $exit_code" >&2
    fi
    
    exit $exit_code
}

# Register cleanup on EXIT (runs on success or failure)
trap cleanup EXIT

# Also trap on signals
trap 'echo "Interrupted"; exit 1' SIGINT SIGTERM

# Main script
BACKGROUND_PID=$$
long_running_task &
BACKGROUND_PID=$!

wait $BACKGROUND_PID
```

### Parameter Expansion: Safe and Powerful

```bash
#!/bin/bash
set -euo pipefail

# Basic expansion (dangerous if undefined)
# echo $var       # Undefined var expands to ""

# Safe expansion with defaults
CONFIG_FILE="${CONFIG_FILE:-/etc/myapp/config.conf}"
LOG_LEVEL="${LOG_LEVEL:-INFO}"
RETRIES="${RETRIES:-3}"

# Expansion with error on undefined (best for required args)
DATABASE_URL="${DATABASE_URL:?DATABASE_URL not set}"
API_KEY="${API_KEY:?API_KEY environment variable required}"

# Pattern removal (clean up paths, strip extensions)
FILENAME="archive.tar.gz"
NAME="${FILENAME%.tar.gz}"        # Remove .tar.gz suffix: archive
BASENAME="${FILENAME##*/}"        # Remove directory path
DIRNAME="${FILEPATH%/*}"          # Remove filename

# More patterns:
FILE="/path/to/file.txt"
"${FILE%%.*}"                     # Remove .*: /path/to/file
"${FILE%.*}"                      # Remove last .*: /path/to/file
"${FILE#*/}"                      # Remove leading /: path/to/file.txt

# Case conversion (bash 4+)
LOWER="${VAR,,}"                  # Convert to lowercase
UPPER="${VAR^^}"                  # Convert to uppercase
```

### Logging Patterns for Production

```bash
#!/bin/bash
set -euo pipefail

# Structured logging function
log() {
    local level=$1
    shift
    local message="$@"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    
    case "$level" in
        INFO)  echo "[INFO] $timestamp: $message" ;;
        WARN)  echo "[WARN] $timestamp: $message" >&2 ;;
        ERROR) echo "[ERROR] $timestamp: $message" >&2 ;;
        *)     echo "[DEBUG] $timestamp: $message" >&2 ;;
    esac
}

# Usage
log INFO "Starting deployment"
log WARN "Database connection slow"
log ERROR "Failed to upload artifact"

# Separate stdout and stderr
command 2> >(log ERROR)  # stderr goes through log ERROR
command > >(log INFO)     # stdout goes through log INFO
```

### Lockfile Pattern: Prevent Concurrent Runs

```bash
#!/bin/bash
set -euo pipefail

LOCKFILE="/var/run/myapp-deploy.lock"
TIMEOUT=300  # 5 minutes

# Create lockfile, fail if another instance running
acquire_lock() {
    local elapsed=0
    while [ $elapsed -lt $TIMEOUT ]; do
        if mkdir "$LOCKFILE" 2>/dev/null; then
            # Lock acquired
            trap 'rmdir "$LOCKFILE"' EXIT
            echo $$ > "$LOCKFILE/pid"
            return 0
        fi
        
        # Timeout if lock is stale (older than 1 hour)
        if [ $(($(date +%s) - $(stat -c %Y "$LOCKFILE"))) -gt 3600 ]; then
            echo "Removing stale lockfile" >&2
            rmdir "$LOCKFILE" 2>/dev/null || true
        else
            echo "Lock exists ($(cat $LOCKFILE/pid)), waiting..."
            sleep 5
            elapsed=$((elapsed + 5))
        fi
    done
    
    echo "Failed to acquire lock after $TIMEOUT seconds" >&2
    return 1
}

acquire_lock
# Now script can proceed safely
```

### Argument Parsing with getopts

```bash
#!/bin/bash
set -euo pipefail

# Define defaults
VERBOSE=0
DRY_RUN=0
ENVIRONMENT="production"

# Parse arguments
usage() {
    cat <<EOF
Usage: $0 [OPTIONS]
Options:
  -v, --verbose      Enable verbose output
  -d, --dry-run      Don't make changes, just show what would happen
  -e, --env ENV      Environment: production|staging|dev (default: production)
  -h, --help         Show this help message
EOF
    exit "${1:-0}"
}

while [[ $# -gt 0 ]]; do
    case "$1" in
        -v|--verbose)
            VERBOSE=1
            shift
            ;;
        -d|--dry-run)
            DRY_RUN=1
            shift
            ;;
        -e|--env)
            ENVIRONMENT="$2"
            shift 2
            ;;
        -h|--help)
            usage 0
            ;;
        *)
            echo "Unknown option: $1" >&2
            usage 1
            ;;
    esac
done

# Validate environment
case "$ENVIRONMENT" in
    production|staging|dev)
        ;;
    *)
        echo "Invalid environment: $ENVIRONMENT" >&2
        exit 1
        ;;
esac

echo "Verbose: $VERBOSE, DryRun: $DRY_RUN, Environment: $ENVIRONMENT"
```

### Here-Documents for Multi-line Config

```bash
#!/bin/bash
set -euo pipefail

# Generate config file without escaping nightmare
cat > /tmp/config.yaml <<'EOF'
server:
  port: 8080
  host: "0.0.0.0"
  ssl:
    enabled: true
    cert: "/etc/ssl/cert.pem"
database:
  url: "postgres://localhost/mydb"
  pool_size: 20
EOF

# With variable substitution (use quotes around EOF to disable)
DBURL="postgres://prod.example.com/db"
cat > /tmp/config.yaml <<EOF
database:
  url: "$DBURL"
  retries: 3
EOF

# For heredocs with indentation, use <<- to strip leading tabs
configure_app() {
    cat > /tmp/app.conf <<-	EOF
	[app]
	name=myapp
	version=1.0.0
	EOF
}
```

### Production Script Template

```bash
#!/bin/bash
set -euo pipefail
IFS=$'\n\t'

# Script metadata
SCRIPT_NAME="$(basename "$0")"
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
SCRIPT_VERSION="1.0.0"

# Configuration
VERBOSE="${VERBOSE:-0}"
DRY_RUN="${DRY_RUN:-0}"
LOG_FILE="${LOG_FILE:-/var/log/$(basename $0).log}"

# Logging
log() {
    local level=$1
    shift
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $*" | tee -a "$LOG_FILE"
}

# Error handling
error_exit() {
    log ERROR "$1"
    exit "${2:-1}"
}

# Cleanup on exit
cleanup() {
    local exit_code=$?
    log INFO "Cleaning up..."
    # Remove temp files, kill background jobs, etc.
    exit $exit_code
}

trap cleanup EXIT SIGINT SIGTERM

# Validation
validate_environment() {
    log INFO "Validating environment"
    
    [[ -n "${REQUIRED_VAR:-}" ]] || error_exit "REQUIRED_VAR not set"
    [[ -f /etc/required/file ]] || error_exit "/etc/required/file not found"
}

# Main execution
main() {
    log INFO "Starting $SCRIPT_NAME v$SCRIPT_VERSION"
    
    validate_environment
    
    # Main logic here
    log INFO "Execution complete"
}

# Run
main "$@"
```

### Error Messages That Help Debugging

```bash
#!/bin/bash
set -euo pipefail

# BAD: Silent failure
command_that_might_fail

# GOOD: Fail with helpful message
if ! command_that_might_fail; then
    cat >&2 <<EOF
ERROR: command_that_might_fail returned $?

Debugging information:
- Working directory: $(pwd)
- User: $(whoami)
- Host: $(hostname)

Please check logs at: $LOG_FILE
Contact: team@example.com
EOF
    exit 1
fi

# BETTER: Use explicit error messages
assert_command_exists() {
    if ! command -v "$1" &>/dev/null; then
        cat >&2 <<EOF
FATAL: Required command not found: $1

This script requires $1 to be installed and in PATH.

Installation:
  Ubuntu/Debian: sudo apt-get install $1
  RHEL/CentOS:   sudo yum install $1
  macOS:         brew install $1

Detected OS: $(uname -s)
Current PATH: $PATH
EOF
        return 1
    fi
}

assert_command_exists docker
assert_command_exists jq
```

---

## ⚠️ Gotchas & Pro Tips

- **Unquoted variables expand dangerously**: `echo $var` splits on IFS and globs. Always quote: `echo "$var"`. Exception: when you explicitly want splitting.

- **Pipefail is essential**: Without it, `tar cf - /data | gzip` succeeds even if tar fails. Always use `set -o pipefail`.

- **Subshells are expensive**: Each `$(...)` creates a subshell. In loops, this is slow. Prefer `read` instead.

- **shift vs $@**: `shift` removes arguments in-place; `$@` is safer for passing through to other commands. Use `"$@"` in function signatures.

- **Here-documents create temp files**: Each `<<EOF` block creates a temp file. In tight loops, this adds overhead. Prefer echo strings or functions.

- **Trap handlers run sequentially**: If you register multiple traps, they all run. Use a cleanup function instead of multiple trap commands.

- **exit vs return**: `exit` ends the script; `return` exits the current function. In sourced scripts, use `return` to avoid closing the shell.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
