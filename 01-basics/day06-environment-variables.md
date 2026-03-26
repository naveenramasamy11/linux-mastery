# 🐧 Environment Variables & Shell Expansion — Linux Mastery

> **Understanding how the shell expands variables, globs, and expressions before executing commands is the difference between scripts that work and scripts that silently misbehave.**

## 📖 Concept

Environment variables are key-value pairs that the kernel and shell expose to every running process. They define the runtime context: where to find executables (`PATH`), who the user is (`USER`, `HOME`), how the terminal behaves (`TERM`, `COLUMNS`), and custom values your applications depend on. Every process inherits its environment from its parent via `execve()` — a fact that has deep implications for how you configure services in systemd, Docker, or EC2 user-data scripts.

Shell expansion is the preprocessing the shell performs *before* executing a command. When you type `echo $HOME/*.log`, the shell first substitutes `$HOME`, then expands `*.log` into matching filenames — the `echo` binary never sees the dollar sign or the glob. Knowing the exact order of expansion (brace → tilde → parameter → command substitution → arithmetic → word splitting → pathname → quote removal) lets you predict and control behaviour precisely.

In production AWS environments, environment variables are the primary mechanism for injecting secrets, region configuration, and feature flags into EC2 instances, Lambda functions, ECS tasks, and Kubernetes pods. Misunderstanding variable scoping — especially the difference between shell variables and *exported* environment variables — is a common source of hard-to-debug issues where a script works interactively but fails when run by systemd or a CI runner.

---

## 💡 Real-World Use Cases

- Injecting AWS credentials and region into EC2 instances via `cloud-init` using environment variables rather than baking them into AMIs
- Configuring 12-factor app settings (database URLs, API keys) for ECS/K8s workloads without modifying container images
- Debugging why a cron job or systemd service can't find a binary by inspecting its inherited `PATH`
- Using shell expansion in Terraform or Ansible wrapper scripts to dynamically build resource names from `$ENVIRONMENT` and `$AWS_REGION`

---

## 🔧 Commands & Examples

### Viewing and Setting Variables

```bash
# Print all environment variables (exported, inherited by child processes)
env
printenv

# Print a specific variable
echo $HOME
printenv HOME

# Set a shell variable (NOT exported — child processes won't see it)
MY_VAR="hello"
echo $MY_VAR           # works in current shell
bash -c 'echo $MY_VAR' # empty — child process doesn't inherit it

# Export a variable so child processes inherit it
export MY_VAR="hello"
bash -c 'echo $MY_VAR' # now prints: hello

# Export and assign in one line
export AWS_REGION="us-east-1"
export DB_URL="postgres://localhost:5432/mydb"

# Export an existing variable
MY_VAR="world"
export MY_VAR

# Set a variable for just one command (doesn't affect current shell)
AWS_PROFILE=staging aws s3 ls
MY_DEBUG=1 ./my-script.sh

# Unset a variable
unset MY_VAR
echo $MY_VAR  # empty
```

### Inspecting the Environment

```bash
# Show all variables (shell + env + functions)
set

# Show only exported variables
export -p

# Show environment of a running process (great for debugging services)
cat /proc/<PID>/environ | tr '\0' '\n'

# Example: see what environment nginx is running with
cat /proc/$(pgrep nginx | head -1)/environ | tr '\0' '\n' | grep -E 'PATH|HOME|USER'

# Show environment of a systemd service
systemctl show-environment
# Or via journald context
sudo systemctl cat sshd | grep Environment
```

### Shell Parameter Expansion

```bash
# Basic substitution
NAME="naveen"
echo "Hello, ${NAME}"      # Hello, naveen

# Default value if unset or empty
echo "${REGION:-us-east-1}"    # uses us-east-1 if REGION is unset
echo "${REGION:=us-east-1}"    # also ASSIGNS us-east-1 if REGION is unset

# Error if unset
echo "${REQUIRED_VAR:?'REQUIRED_VAR must be set'}"

# Use alternate value if set
echo "${DEBUG:+--verbose}"    # outputs --verbose only if DEBUG is set

# String length
STR="hello world"
echo "${#STR}"   # 11

# Substring extraction: ${var:offset:length}
echo "${STR:6:5}"    # world
echo "${STR:0:5}"    # hello
echo "${STR: -5}"    # world (note space before -)

# Remove prefix (shortest match)
FILE="path/to/my-script.sh"
echo "${FILE#*/}"      # to/my-script.sh  (removes up to first /)
echo "${FILE##*/}"     # my-script.sh     (removes up to last /)

# Remove suffix (shortest match)
echo "${FILE%.*}"      # path/to/my-script  (removes .sh)
echo "${FILE%%.*}"     # path/to/my-script  (same here, but be careful with multiple dots)

# Replace first match
VERSION="1.2.3-beta"
echo "${VERSION/-/_}"    # 1.2.3_beta

# Replace all matches
LOG="err err err"
echo "${LOG//err/ERROR}"   # ERROR ERROR ERROR

# Convert case (bash 4+)
STR="Hello World"
echo "${STR,,}"   # hello world (all lowercase)
echo "${STR^^}"   # HELLO WORLD (all uppercase)
echo "${STR,}"    # hello World (first char lowercase)
echo "${STR^}"    # Hello World (first char uppercase)
```

### Brace Expansion

```bash
# Generate sequences
echo {1..5}            # 1 2 3 4 5
echo {a..e}            # a b c d e
echo {01..10}          # 01 02 03 04 05 06 07 08 09 10

# Step sequences
echo {0..20..5}        # 0 5 10 15 20

# Create directory structures in one line
mkdir -p app/{src,tests,docs}/{main,utils}

# Create dated backup files
cp config.yaml config.yaml.bak.{$(date +%Y%m%d)}

# AWS: create S3 prefixes across regions
for prefix in us-east-1 us-west-2 eu-west-1; do
  aws s3api put-object --bucket my-bucket --key "logs/${prefix}/"
done
# Or with brace expansion (when regions are known):
echo logs/{us-east-1,us-west-2,eu-west-1}/

# Multiple prefix/suffix combinations
echo {pre1,pre2}-{suf1,suf2}  # pre1-suf1 pre1-suf2 pre2-suf1 pre2-suf2
```

### Command Substitution

```bash
# Capture command output into a variable
DATE=$(date +%Y-%m-%d)
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)

# Use directly in a string
echo "Deploying to instance $INSTANCE_ID in $REGION on $DATE"

# Nest command substitutions
LATEST_AMI=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text)

# Old backtick style — avoid in new scripts, hard to nest
DATE=`date +%Y%m%d`
```

### Arithmetic Expansion

```bash
# Basic arithmetic
echo $((2 + 3))         # 5
echo $((10 / 3))        # 3  (integer division)
echo $((10 % 3))        # 1  (modulo)
echo $((2 ** 8))        # 256

# Increment/decrement
COUNTER=0
((COUNTER++))
echo $COUNTER   # 1
((COUNTER += 5))
echo $COUNTER   # 6

# Use in conditions
if (( COUNTER > 4 )); then
  echo "Counter exceeded threshold"
fi

# Calculate file size thresholds
MAX_SIZE=$((500 * 1024 * 1024))   # 500 MB in bytes
FILE_SIZE=$(stat -c %s /var/log/app.log)
if (( FILE_SIZE > MAX_SIZE )); then
  echo "Log file too large, rotating"
fi
```

### Arrays

```bash
# Declare and initialise an array
SERVERS=("web01" "web02" "db01" "cache01")

# Access elements
echo "${SERVERS[0]}"       # web01
echo "${SERVERS[@]}"       # all elements
echo "${#SERVERS[@]}"      # number of elements (4)
echo "${SERVERS[@]:1:2}"   # slice: web02 db01

# Iterate
for server in "${SERVERS[@]}"; do
  echo "Checking $server..."
done

# Append to array
SERVERS+=("monitor01")

# Associative arrays (bash 4+)
declare -A REGION_ENDPOINTS
REGION_ENDPOINTS[us-east-1]="api-east.example.com"
REGION_ENDPOINTS[eu-west-1]="api-eu.example.com"

echo "${REGION_ENDPOINTS[us-east-1]}"

for region in "${!REGION_ENDPOINTS[@]}"; do
  echo "$region => ${REGION_ENDPOINTS[$region]}"
done
```

### The PATH Variable

```bash
# View current PATH
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Prepend to PATH (your tools take priority)
export PATH="$HOME/.local/bin:$PATH"

# Append to PATH
export PATH="$PATH:/opt/custom-tools/bin"

# Permanently add to PATH — append to ~/.bashrc or ~/.bash_profile
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Find where a binary is coming from
which python3
type python3
command -v python3   # most portable

# See all locations of a command in PATH order
type -a python3
```

### Managing Environment in Production

```bash
# systemd: set environment for a service
# /etc/systemd/system/myapp.service
cat << 'EOF' > /etc/systemd/system/myapp.service
[Unit]
Description=My Application

[Service]
EnvironmentFile=/etc/myapp/env
ExecStart=/usr/local/bin/myapp
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# /etc/myapp/env — one KEY=VALUE per line
cat << 'EOF' > /etc/myapp/env
DB_HOST=db.internal
DB_PORT=5432
LOG_LEVEL=info
AWS_REGION=us-east-1
EOF

systemctl daemon-reload
systemctl restart myapp

# Override env for a single run without editing the unit file
systemctl set-environment DEBUG=1
systemctl restart myapp
systemctl unset-environment DEBUG

# Docker: pass env vars
docker run -e DB_HOST=db.internal -e LOG_LEVEL=debug myapp:latest
docker run --env-file .env myapp:latest

# K8s: env from ConfigMap / Secret
kubectl create configmap app-config --from-literal=LOG_LEVEL=info
kubectl create secret generic db-creds --from-literal=DB_PASSWORD=secret123
```

---

## ⚠️ Gotchas & Pro Tips

- **Quoting is critical:** Always double-quote variable expansions: `"$VAR"` and `"${ARRAY[@]}"`. Unquoted expansions undergo word splitting and pathname expansion, which breaks filenames with spaces and can expand globs unexpectedly.

- **Shell vs environment variables:** Setting `FOO=bar` in a script does *not* export it — child processes won't see it. Always `export` variables that subprocesses need. This bites people when they set `AWS_PROFILE=prod` without exporting and wonder why the AWS CLI ignores it.

- **Source vs execute:** Running `./script.sh` spawns a subshell; variables set inside don't affect the parent. Running `source ./script.sh` (or `. ./script.sh`) runs in the current shell, so exported variables persist. This is how `virtualenv activate` and `aws-vault exec` work.

- **`env -i` for clean environments:** When testing or running scripts, `env -i HOME=/tmp bash --noprofile --norc ./script.sh` gives you a minimal environment, which is great for catching missing exports or PATH assumptions.

- **Secrets should never be in `env`:** Anything in `env` is visible to all processes running as that user via `/proc/<pid>/environ`. Use secrets managers (AWS Secrets Manager, SSM Parameter Store, Vault) and inject values at the last possible moment.

- **`$()` vs backticks:** Always use `$()` for command substitution. It nests cleanly, is easier to read, and avoids escaping nightmares in backtick syntax.

- **`declare -r` for constants:** Use `declare -r CONSTANT="value"` to make variables read-only. Attempting to reassign causes an immediate error rather than a silent wrong-value bug.

- **IFS and word splitting:** The Internal Field Separator (`IFS`) defaults to space/tab/newline. Changing it (e.g., `IFS=,` to parse CSVs) must be done carefully and restored afterward to avoid breaking downstream command processing.

```bash
# Safe IFS usage — restore after
OLD_IFS="$IFS"
IFS=','
read -ra FIELDS <<< "us-east-1,us-west-2,eu-west-1"
IFS="$OLD_IFS"
for region in "${FIELDS[@]}"; do echo "$region"; done
```

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
