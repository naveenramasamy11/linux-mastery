# 🐧 Bash Process Substitution & Here Documents — Linux Mastery

> **Master bash I/O redirection tricks that let you pipe to non-pipe-aware tools and embed multi-line content inline.**

## 📖 Concept

Process substitution and here documents are two of bash's most underused yet powerful I/O features. They show up constantly in production DevOps scripts, Terraform provisioners, Ansible shell tasks, and AWS CLI wrapper scripts — yet most engineers learn them by accident.

**Process Substitution** (`<(cmd)` and `>(cmd)`) lets you treat the output of a command as a temporary file, bypassing the need to write intermediate files. Under the hood, bash creates a named pipe (FIFO) or `/dev/fd/N` file descriptor and substitutes the path into the command. This is invaluable when tools only accept filenames, not stdin — like `diff`, `comm`, `join`, or configuration validators.

**Here Documents** (`<<EOF`) embed multi-line strings directly in scripts, which is critical for generating config files, SQL queries, kubectl manifests, or any templated content. **Here Strings** (`<<<`) send a single-line string to stdin without needing `echo | cmd`. Both eliminate temporary file management and keep scripts readable.

Understanding these mechanisms also makes you better at debugging — you'll recognize them in Ansible's `shell` module output, Terraform's `local-exec`, and shell-based Kubernetes init containers.

---

## 💡 Real-World Use Cases

- Compare two remote files without downloading them: `diff <(ssh host1 cat /etc/nginx/nginx.conf) <(ssh host2 cat /etc/nginx/nginx.conf)`
- Feed AWS SSM parameter output directly to an application without writing to disk
- Generate kubeconfig, deployment YAML, or nginx config on the fly from a template in a deploy script
- Pass a string to `base64`, `jq`, `bc`, or `awk` without using `echo | cmd` (which has newline quirks)
- Write multi-line Ansible `shell` tasks or cloud-init scripts cleanly

---

## 🔧 Commands & Examples

### Process Substitution — Input `<(cmd)`

```bash
# diff two command outputs as if they were files
diff <(ls /etc/nginx/conf.d/) <(ls /etc/nginx/sites-enabled/)

# diff two sorted lists without creating temp files
diff <(sort file1.txt) <(sort file2.txt)

# compare live running config vs deployed config
diff <(ssh prod-server cat /etc/app/config.yaml) ./config.yaml

# pass two API responses to diff
diff <(curl -s https://api1.example.com/config) <(curl -s https://api2.example.com/config)

# compare local and remote /etc/hosts
diff /etc/hosts <(ssh remotehost cat /etc/hosts)

# comm needs sorted input — process substitution handles it inline
comm -23 <(sort file_a.txt) <(sort file_b.txt)

# join two sorted files
join <(sort -k1 users.txt) <(sort -k1 groups.txt)

# md5sum multiple remote files
md5sum <(ssh host1 cat /var/log/app.log) <(ssh host2 cat /var/log/app.log)
```

### Process Substitution — Output `>(cmd)`

```bash
# tee to two different processing pipelines simultaneously
tee >(gzip > output.gz) >(wc -l) > /dev/null < input.txt

# send logs to both a file and a remote syslog
app_command 2>&1 | tee >(logger -t myapp) >> /var/log/myapp.log

# split a CSV and process two columns independently
cat data.csv | tee >(cut -d, -f1 | sort > col1.txt) >(cut -d, -f2 | sort > col2.txt) > /dev/null

# compress and checksum simultaneously
cat bigfile.tar | tee >(md5sum > bigfile.tar.md5) | gzip > bigfile.tar.gz
```

### Here Documents (`<<EOF`)

```bash
# basic heredoc — writes to stdout
cat <<EOF
server {
    listen 80;
    server_name example.com;
    root /var/www/html;
}
EOF

# write to a file
cat <<EOF > /etc/nginx/conf.d/myapp.conf
server {
    listen 80;
    server_name ${DOMAIN};
    proxy_pass http://localhost:${APP_PORT};
}
EOF

# pass to a command
ssh remote-host <<EOF
echo "Running on \$(hostname)"
systemctl restart nginx
journalctl -u nginx -n 20
EOF

# SQL heredoc — pass to psql
psql -U appuser -d mydb <<EOF
SELECT table_name, pg_size_pretty(pg_total_relation_size(table_name::text))
FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY pg_total_relation_size(table_name::text) DESC;
EOF

# kubectl apply from heredoc — no temp file needed
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  replicas: ${REPLICAS}
  selector:
    matchLabels:
      app: ${APP_NAME}
  template:
    metadata:
      labels:
        app: ${APP_NAME}
    spec:
      containers:
      - name: ${APP_NAME}
        image: ${IMAGE}:${TAG}
        ports:
        - containerPort: 8080
EOF

# suppress variable expansion with quoted delimiter
cat <<'EOF'
This $DOLLAR and ${SIGN} are NOT expanded.
Use this for scripts that contain shell variables meant for later.
EOF

# indented heredoc (bash 4+) — strip leading tabs with <<-
cat <<-EOF
	This line has a leading tab that gets stripped.
	Useful for indented heredocs inside functions.
	EOF

# heredoc in a function
generate_config() {
    local env=$1
    local port=$2
    cat <<EOF
APP_ENV=${env}
APP_PORT=${port}
APP_DEBUG=$([ "$env" == "dev" ] && echo "true" || echo "false")
EOF
}

generate_config production 8080 > /etc/app/.env
```

### Here Strings (`<<<`)

```bash
# send a string to a command (no echo | cmd)
base64 <<< "Hello World"
# avoids the echo | base64 pattern (which adds a newline inconsistently)

# check if a string contains a pattern
grep -q "error" <<< "$log_output" && echo "ERROR FOUND"

# pass JSON string to jq
jq '.name' <<< '{"name": "naveen", "role": "devops"}'

# pass to bc for math
bc -l <<< "scale=4; 22/7"

# use herestring with read
read first last <<< "John Doe"
echo "First: $first, Last: $last"

# aws cli output processing
region=$(aws configure get region)
jq '.Reservations[].Instances[].InstanceId' <<< "$(aws ec2 describe-instances --region $region)"

# pass multiline variable to command
heredoc_var=$(cat /etc/nginx/nginx.conf)
grep "worker_processes" <<< "$heredoc_var"
```

### Combining Techniques in Real Scripts

```bash
#!/bin/bash
# Deploy script using process substitution + heredoc + herestring

set -euo pipefail

APP_NAME="${1:?Usage: $0 <app-name> <env>}"
ENV="${2:?Usage: $0 <app-name> <env>}"
IMAGE_TAG=$(git rev-parse --short HEAD)
NAMESPACE="${ENV}-${APP_NAME}"

# Validate JSON config without temp file
jq empty <<< "$(cat config/${ENV}.json)" || { echo "Invalid JSON config"; exit 1; }

# Diff current vs new deployment
echo "=== Deployment diff ==="
diff \
  <(kubectl get deployment "${APP_NAME}" -n "${NAMESPACE}" -o yaml 2>/dev/null || echo "") \
  <(cat <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ${APP_NAME}
  template:
    metadata:
      labels:
        app: ${APP_NAME}
        version: ${IMAGE_TAG}
    spec:
      containers:
      - name: ${APP_NAME}
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/${APP_NAME}:${IMAGE_TAG}
        ports:
        - containerPort: 8080
        envFrom:
        - secretRef:
            name: ${APP_NAME}-secrets
EOF
) || true  # diff returns 1 on differences — don't fail

# Apply
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ${APP_NAME}
  template:
    metadata:
      labels:
        app: ${APP_NAME}
        version: ${IMAGE_TAG}
    spec:
      containers:
      - name: ${APP_NAME}
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/${APP_NAME}:${IMAGE_TAG}
        ports:
        - containerPort: 8080
        envFrom:
        - secretRef:
            name: ${APP_NAME}-secrets
EOF

echo "Deployed ${APP_NAME}:${IMAGE_TAG} to ${ENV}"
```

---

## ⚠️ Gotchas & Pro Tips

- **Process substitution is bash-specific:** `<()` and `>()` don't work in `sh`, `dash`, or `busybox ash`. Always use `#!/bin/bash` (not `#!/bin/sh`) in scripts that use them. Alpine-based Docker containers use `ash` — this bites people in CI pipelines.

- **`<()` creates a FIFO, not a seekable file:** Tools that need to seek (read twice) through a file will fail with process substitution. `wc`, `sort`, and `diff` work fine; tools that seek (like SQLite `.import`) may not.

- **Here document variable expansion is default:** Variables are expanded in heredocs unless you quote the delimiter (`<<'EOF'`). This causes silent bugs when embedding scripts that use `$VARIABLE` syntax meant for a later shell.

- **Trailing newline in herestrings:** `echo "text" | cmd` and `cmd <<< "text"` both add a newline. Use `printf '%s' "text" | cmd` if you need no trailing newline.

- **`<<-` strips only tabs, not spaces:** Indentation with spaces is NOT stripped. If your editor converts tabs to spaces, `<<-` won't work as expected.

- **Process substitution and pipelines don't mix exit codes:** In `diff <(cmd1) <(cmd2)`, `cmd1` and `cmd2` failures may be hidden. Wrap in `set -o pipefail` and check `${PIPESTATUS[@]}` for critical scripts.

- **AWS CLI + heredoc:** When running AWS CLI commands inside a heredoc over SSH, make sure AWS credentials are available on the remote host. Environment variables aren't inherited across SSH by default unless you pass `-o SendEnv=AWS_*`.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
