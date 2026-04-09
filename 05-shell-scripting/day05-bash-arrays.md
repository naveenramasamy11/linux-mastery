# 🐧 Bash Arrays & Associative Arrays — Linux Mastery

> **Bash arrays transform shell scripts from line-by-line text processors into real programs — they're the key to writing loops that handle filenames with spaces, managing parallel job PIDs, and building lookup tables without spawning external tools.**

## 📖 Concept

Bash supports two types of arrays: **indexed arrays** (integer keys, `declare -a`) and **associative arrays** (string keys, `declare -A`, bash 4.0+). Both are sparse — you don't need to fill every index. Unlike Python or JavaScript, bash array elements are always strings (though arithmetic operations work on numeric strings).

The syntax is notoriously quirky. Forgetting `"${array[@]}"` (with quotes) instead of `${array[@]}` is the #1 source of subtle bugs when array elements contain spaces. The `[@]` vs `[*]` distinction matters: `[@]` expands each element as a separate word (safe with spaces), `[*]` joins all elements with the first character of `IFS`.

Associative arrays (`declare -A`) are the bash equivalent of Python dicts. They enable O(1) lookups, deduplication, and state machines in pure bash — avoiding expensive `grep`/`awk` calls in hot loops. When you find yourself building parallel job trackers, environment mappings, or host-to-IP lookups in scripts, associative arrays are the right tool.

In DevOps/SRE scripting these patterns appear constantly: iterating over a list of EC2 instance IDs, mapping region→endpoint, tracking background job PIDs, building sets of unique values from log analysis.

---

## 💡 Real-World Use Cases

- **Parallel job tracking:** Store background PIDs in an associative array keyed by job name; harvest exit codes after `wait`.
- **Region-to-endpoint mapping:** `declare -A ENDPOINTS=([us-east-1]="api.east.example.com" [eu-west-1]="api.eu.example.com")` — clean lookup without parsing config files.
- **Deduplication:** Build a `declare -A SEEN` set to skip already-processed items in log processing scripts.
- **Argument parsing:** Store flags and values in an associative array as a lightweight alternative to `getopts` for complex CLIs.
- **Batch AWS operations:** Store instance IDs in an array, iterate to tag/stop/start/describe without spawning a new aws CLI process per item.

---

## 🔧 Commands & Examples

### Indexed Arrays — Basics

```bash
# Declaration and initialization
declare -a FRUITS=("apple" "banana" "cherry")

# Or without declare
SERVERS=("web01" "web02" "db01" "cache01")

# Append elements
SERVERS+=("lb01")
SERVERS+=("lb02" "lb03")   # Append multiple

# Access by index (0-based)
echo "${SERVERS[0]}"   # web01
echo "${SERVERS[-1]}"  # lb03 (last element, bash 4.3+)

# All elements (safe with spaces)
echo "${SERVERS[@]}"   # web01 web02 db01 cache01 lb01 lb02 lb03

# All elements as single string (IFS-joined)
echo "${SERVERS[*]}"

# Array length
echo "${#SERVERS[@]}"  # 7

# Length of specific element
echo "${#SERVERS[0]}"  # 5 (length of "web01")

# All indices
echo "${!SERVERS[@]}"  # 0 1 2 3 4 5 6

# Slice: elements 2 through 4 (offset, length)
echo "${SERVERS[@]:2:3}"  # db01 cache01 lb01
```

### Indexed Arrays — Iteration

```bash
# Safe iteration (handles spaces in elements)
INSTANCES=("i-0abc123" "i-0def456" "i-0ghi789")
for instance in "${INSTANCES[@]}"; do
    echo "Describing: $instance"
    aws ec2 describe-instances --instance-ids "$instance" \
        --query 'Reservations[].Instances[].State.Name' \
        --output text
done

# Iterate with index
for i in "${!SERVERS[@]}"; do
    echo "[$i] ${SERVERS[$i]}"
done

# Files with spaces — this is WHERE arrays shine
# WRONG (breaks on filenames with spaces):
for f in $(find /data -name "*.log"); do cat "$f"; done

# RIGHT:
readarray -t LOG_FILES < <(find /data -name "*.log")
for f in "${LOG_FILES[@]}"; do
    wc -l "$f"
done

# Read lines from command into array
readarray -t RUNNING_PODS < <(kubectl get pods -o name --field-selector=status.phase=Running)
echo "Running pods: ${#RUNNING_PODS[@]}"
```

### Indexed Arrays — Manipulation

```bash
# Unset element (array becomes sparse)
unset SERVERS[2]
echo "${SERVERS[@]}"     # web01 web02 cache01 lb01 lb02 lb03
echo "${!SERVERS[@]}"    # 0 1 3 4 5 6  ← index 2 is gone

# Re-index after deletion (pack the array)
SERVERS=("${SERVERS[@]}")
echo "${!SERVERS[@]}"    # 0 1 2 3 4 5

# Sort an array
IFS=$'\n' SORTED=($(sort <<< "${SERVERS[*]}")); unset IFS

# Remove duplicates (pipe through sort -u)
IFS=$'\n' UNIQUE=($(printf '%s\n' "${SERVERS[@]}" | sort -u)); unset IFS

# Array search (check if value exists)
contains() {
    local needle="$1"; shift
    local item
    for item in "$@"; do
        [[ "$item" == "$needle" ]] && return 0
    done
    return 1
}

contains "db01" "${SERVERS[@]}" && echo "found" || echo "not found"

# String operations on all elements (parameter expansion)
FILES=("report.txt" "data.txt" "log.txt")
echo "${FILES[@]%.txt}"     # report data log  (strip .txt suffix)
echo "${FILES[@]/#/backup-}"  # backup-report.txt backup-data.txt...
```

### Associative Arrays — Basics

```bash
# Must declare explicitly
declare -A REGION_AMI
REGION_AMI["us-east-1"]="ami-0abcdef1234567890"
REGION_AMI["us-west-2"]="ami-0fedcba9876543210"
REGION_AMI["eu-west-1"]="ami-0123456789abcdef0"

# Inline initialization
declare -A DB_PORTS=(
    ["postgres"]=5432
    ["mysql"]=3306
    ["redis"]=6379
    ["mongodb"]=27017
    ["elasticsearch"]=9200
)

# Access by key
echo "${REGION_AMI["us-east-1"]}"   # ami-0abcdef...

# All keys
echo "${!DB_PORTS[@]}"   # postgres mysql redis mongodb elasticsearch

# All values
echo "${DB_PORTS[@]}"    # 5432 3306 6379 27017 9200

# Check if key exists
if [[ -v DB_PORTS["postgres"] ]]; then
    echo "postgres port: ${DB_PORTS["postgres"]}"
fi

# Unset a key
unset DB_PORTS["mongodb"]

# Number of keys
echo "${#DB_PORTS[@]}"
```

### Associative Arrays — Real Patterns

```bash
#!/bin/bash
# Pattern 1: Parallel job tracker with named jobs

declare -A JOB_PIDS
declare -A JOB_STATUS

REGIONS=("us-east-1" "us-west-2" "eu-west-1" "ap-southeast-1")

# Launch parallel jobs
for region in "${REGIONS[@]}"; do
    aws ec2 describe-instances \
        --region "$region" \
        --query 'Reservations[].Instances[].InstanceId' \
        --output text > /tmp/instances-${region}.txt &
    JOB_PIDS[$region]=$!
done

# Harvest results
for region in "${REGIONS[@]}"; do
    if wait "${JOB_PIDS[$region]}"; then
        COUNT=$(wc -l < /tmp/instances-${region}.txt)
        JOB_STATUS[$region]="OK ($COUNT instances)"
    else
        JOB_STATUS[$region]="FAILED"
    fi
done

# Print summary
for region in "${REGIONS[@]}"; do
    printf "%-20s %s\n" "$region" "${JOB_STATUS[$region]}"
done
```

```bash
#!/bin/bash
# Pattern 2: Deduplication set

declare -A SEEN_IPS

while IFS= read -r line; do
    # Extract IP from Apache access log
    IP=$(echo "$line" | awk '{print $1}')

    if [[ -v SEEN_IPS[$IP] ]]; then
        continue   # Already counted
    fi
    SEEN_IPS[$IP]=1
    echo "New unique IP: $IP"
done < /var/log/httpd/access_log

echo "Total unique IPs: ${#SEEN_IPS[@]}"
```

```bash
#!/bin/bash
# Pattern 3: Config lookup table

declare -A ENV_CONFIG=(
    ["dev,db_host"]="dev-db.internal"
    ["dev,api_url"]="https://api-dev.example.com"
    ["staging,db_host"]="staging-db.internal"
    ["staging,api_url"]="https://api-staging.example.com"
    ["prod,db_host"]="prod-db.internal"
    ["prod,api_url"]="https://api.example.com"
)

ENV="${1:-dev}"

DB_HOST="${ENV_CONFIG["$ENV,db_host"]}"
API_URL="${ENV_CONFIG["$ENV,api_url"]}"

echo "Deploying to: $ENV"
echo "DB: $DB_HOST"
echo "API: $API_URL"
```

### Reading Arrays from Files and Commands

```bash
# Read a file line-by-line into an array
readarray -t HOSTS < /etc/ansible/hosts

# Read command output into array
readarray -t EC2_IDS < <(
    aws ec2 describe-instances \
        --filter "Name=instance-state-name,Values=running" \
        --query 'Reservations[].Instances[].InstanceId' \
        --output text | tr '\t' '\n'
)

# Read CSV fields into array (split by comma)
IFS=',' read -ra FIELDS <<< "name,age,city,country"
for field in "${FIELDS[@]}"; do
    echo "Field: $field"
done

# Build associative array from key=value file
declare -A CONFIG
while IFS='=' read -r key value; do
    [[ "$key" =~ ^#.*$ ]] && continue    # Skip comments
    [[ -z "$key" ]] && continue           # Skip empty lines
    CONFIG["$key"]="${value}"
done < /etc/myapp/config

echo "DB host: ${CONFIG[db_host]}"
```

### Passing Arrays to Functions

```bash
# Bash doesn't pass arrays by value — use nameref (bash 4.3+)
print_array() {
    local -n arr=$1   # nameref — arr IS the caller's array
    local name=$1
    echo "Array '$name' has ${#arr[@]} elements:"
    for item in "${arr[@]}"; do
        echo "  - $item"
    done
}

SERVERS=("web01" "web02" "db01")
print_array SERVERS

# Or pass as serialized elements
process_hosts() {
    local hosts=("$@")
    for host in "${hosts[@]}"; do
        ping -c 1 -W 1 "$host" &>/dev/null && echo "$host: UP" || echo "$host: DOWN"
    done
}

process_hosts "${SERVERS[@]}"
```

---

## ⚠️ Gotchas & Pro Tips

- **Always quote `"${array[@]}"`** when iterating. Without quotes, elements with spaces split into multiple words. This is the single most common bash array bug.
- **`declare -A` requires bash 4.0+.** macOS ships bash 3.2 (due to GPL licensing). On macOS, either `brew install bash` and use `/usr/local/bin/bash`, or use `zsh` which has native associative arrays with similar syntax.
- **Sparse arrays break numeric loops.** `for ((i=0; i<${#arr[@]}; i++))` fails if there are gaps. Use `for i in "${!arr[@]}"` to iterate only defined indices.
- **`${#arr[@]}` counts elements, not the highest index.** If you unset `arr[5]` from a 6-element array, `${#arr[@]}` returns 5 but valid indices are 0,1,2,3,4,6 if you had appended. Re-pack with `arr=("${arr[@]}")`.
- **Associative array key order is undefined.** Don't rely on `${!assoc[@]}` returning keys in insertion order. If order matters, maintain a separate indexed array of keys.
- **Pro tip — use `mapfile` as an alias for `readarray`** (same built-in, two names). `mapfile -t LINES < file` is equivalent to `readarray -t LINES < file`. `mapfile` is more commonly seen in the wild.

```bash
# Quick test: does your script handle filenames with spaces?
TESTFILES=("file one.txt" "file two.txt" "normal.txt")
for f in "${TESTFILES[@]}"; do
    [[ -f "$f" ]] && echo "Found: $f"
done
# If this works, your array quoting is correct
```

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
