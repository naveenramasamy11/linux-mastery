# 🐧 Shell Expansion & File Descriptors — Linux Mastery

> **Understanding how Bash interprets your command line is the difference between writing scripts that work and scripts that almost work.**

## 📖 Concept

Shell expansion is the process Bash uses to transform what you type into what actually gets executed. Before a single external binary is called, Bash runs your command line through up to eight distinct expansion phases — in a strict, deterministic order. Getting this order wrong is the source of countless subtle bugs in scripts, Ansible playbooks, and CI pipeline one-liners.

The expansion order is: **brace expansion → tilde expansion → parameter/variable expansion → command substitution → arithmetic expansion → word splitting → pathname expansion (globbing) → quote removal**. Each phase produces input for the next. Quoting at the right level controls which expansions fire and which are suppressed — mastering this is non-negotiable for anyone writing production automation.

File descriptors (FDs) are the kernel-level plumbing underneath every I/O operation. Every process inherits three standard FDs: 0 (stdin), 1 (stdout), 2 (stderr). But a process can hold hundreds more, and the shell gives you full control over opening, closing, duplicating, and redirecting them. In AWS environments you'll constantly redirect stdout and stderr separately — for CloudWatch log streams, for Ansible output capture, for tee-ing to both a log file and the terminal in a bootstrap script.

Understanding FDs at the system level also explains why `2>&1` must come *after* the stdout redirect, why subshells inherit FDs but not the other way around, and how named pipes and `/dev/fd/*` let you build powerful multi-stream pipelines without temporary files.

---

## 💡 Real-World Use Cases

- Redirect only stderr to CloudWatch agent log while stdout goes to the terminal during EC2 user-data bootstrap
- Use brace expansion to create multi-environment directory scaffolding (`mkdir -p app/{dev,staging,prod}/{config,logs,data}`) in a single line
- Safely capture both stdout and stderr of a Terraform apply into separate log files for post-run analysis
- Use process substitution `<(cmd)` to diff two live command outputs without creating temp files — great for comparing running configs vs desired state

---

## 🔧 Commands & Examples

### Brace Expansion

Brace expansion fires **first**, before any variable is evaluated. It generates strings — no filesystem check occurs.

```bash
# Create a full app directory tree in one shot
mkdir -p /opt/myapp/{bin,lib,conf,logs/{app,access,error},tmp}

# Generate date-stamped backup names
echo backup-{2025,2026}-{01..12}.tar.gz

# Zero-padded sequences
echo node{01..10}   # node01 node02 ... node10

# Step sequences
echo {0..100..10}   # 0 10 20 30 40 50 60 70 80 90 100

# Useful for rolling log rotation filenames
for i in {1..7}; do
  mv /var/log/app/app.log.$((i-1)) /var/log/app/app.log.$i 2>/dev/null
done
```

### Tilde Expansion

```bash
echo ~          # /home/naveen
echo ~root      # /root
echo ~+         # $PWD (current directory)
echo ~-         # $OLDPWD (previous directory)

# Use ~- to go back without remembering the path
cd /some/deep/path
cd /tmp
cd ~-           # back to /some/deep/path
```

### Parameter & Variable Expansion

```bash
# Basic
echo ${HOME}
echo ${USER:-defaultuser}       # use default if unset or empty
echo ${USER:=defaultuser}       # assign default if unset or empty
echo ${VAR:?'VAR must be set'}  # error and exit if unset

# Substring extraction
path="/opt/myapp/conf/app.yaml"
echo ${path#*/}        # opt/myapp/conf/app.yaml   (remove shortest prefix)
echo ${path##*/}       # app.yaml                  (remove longest prefix — basename)
echo ${path%/*}        # /opt/myapp/conf           (remove shortest suffix — dirname)
echo ${path%%.*}       # /opt/myapp/conf/app       (remove longest suffix)

# String replacement
version="v1.2.3-beta"
echo ${version/beta/rc1}        # v1.2.3-rc1
echo ${version//[-.]/_}         # v1_2_3_beta  (replace all matches)

# Case modification (Bash 4+)
str="Hello World"
echo ${str,,}   # hello world
echo ${str^^}   # HELLO WORLD

# Array length and slicing
arr=(a b c d e)
echo ${#arr[@]}       # 5
echo ${arr[@]:1:3}    # b c d  (offset 1, length 3)

# Indirect expansion — useful in dynamic config scripts
for env in DEV STAGING PROD; do
  varname="DB_HOST_${env}"
  echo "${env}: ${!varname}"
done
```

### Command Substitution

```bash
# Modern syntax (preferred — nestable)
today=$(date +%Y-%m-%d)
instance_id=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)

# Get the AWS region from instance metadata
region=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)
echo "Running in: $region"

# Capture multi-line output into an array
mapfile -t interfaces < <(ip -o link show | awk -F': ' '{print $2}')
for iface in "${interfaces[@]}"; do
  echo "Interface: $iface"
done

# Nested substitution
latest_ami=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' \
  --output text --region $(curl -s http://169.254.169.254/latest/meta-data/placement/region))
```

### Arithmetic Expansion

```bash
# $(( )) — integer arithmetic only
echo $(( 2 ** 10 ))       # 1024
echo $(( 16#FF ))         # 255  (hex to decimal)
echo $(( 8#77 ))          # 63   (octal to decimal)
echo $(( 2#1010 ))        # 10   (binary to decimal)

# Disk space check in a script
free_kb=$(df /var --output=avail | tail -1)
free_gb=$(( free_kb / 1024 / 1024 ))
echo "Free space on /var: ${free_gb}GB"

if (( free_gb < 5 )); then
  echo "WARNING: Low disk space!" >&2
fi

# Ternary-style with (( ))
count=42
(( count > 100 )) && echo "high" || echo "low"
```

### Pathname Expansion (Globbing)

```bash
# Standard globs
ls *.log
ls /etc/*.conf
rm /tmp/session_*

# Extended globs (enable with shopt -s extglob)
shopt -s extglob
ls !(*.log)         # everything except .log files
ls +(*.conf|*.yaml) # files ending in .conf or .yaml
ls @(*.sh)          # exactly matching *.sh

# Globstar — recursive matching (Bash 4+)
shopt -s globstar
find_replacement_example() {
  # Instead of find, use globstar for simple cases
  for f in /opt/app/**/*.conf; do
    echo "Config: $f"
  done
}

# Nullglob — prevent literal * when no match
shopt -s nullglob
for f in /tmp/nonexistent_*.log; do
  echo "Found: $f"   # never runs if no match
done
```

### File Descriptors & Redirection

```bash
# Standard FDs
# 0 = stdin, 1 = stdout, 2 = stderr

# Redirect stdout and stderr to different files
some-command > /tmp/stdout.log 2> /tmp/stderr.log

# Redirect stderr to stdout (ORDER MATTERS)
some-command > /tmp/all.log 2>&1    # correct: stdout→file, then stderr→where stdout now points
some-command 2>&1 > /tmp/all.log    # WRONG: stderr→terminal, stdout→file

# Append both to the same file
some-command >> /tmp/combined.log 2>&1

# Discard stderr completely
noisy-command 2>/dev/null

# Discard all output
silent-command > /dev/null 2>&1
# Or in Bash 4+:
silent-command &>/dev/null

# Redirect stdout to stderr (useful in scripts for error messages)
echo "ERROR: something broke" >&2

# Here-doc — feed multi-line string as stdin
cat <<'EOF' > /etc/app/config.ini
[database]
host = localhost
port = 5432
EOF

# Here-string — feed single string as stdin
grep "error" <<< "$log_output"

# Custom file descriptors — open a log file on FD 3
exec 3>/var/log/myscript.log
echo "Starting..." >&3
do_work
echo "Done." >&3
exec 3>&-   # close FD 3

# Read from a file on FD 4
exec 4</etc/hosts
while IFS= read -r line <&4; do
  echo "$line"
done
exec 4<&-
```

### Process Substitution

Process substitution creates a named pipe (FIFO) and gives you its path as `/dev/fd/N` — allowing commands that need *files* to consume *command output* without temp files.

```bash
# diff two live outputs
diff <(ss -tnp) <(cat /tmp/saved-connections.txt)

# diff running sshd_config vs desired state
diff <(ssh bastion 'cat /etc/ssh/sshd_config') <(cat /tmp/desired-sshd_config)

# tee to multiple destinations simultaneously
some-command | tee >(grep ERROR > /tmp/errors.log) >(wc -l > /tmp/line-count.txt) > /tmp/full.log

# Pass kubectl get output to diff without temp file
diff <(kubectl get pods -o yaml) <(cat desired-pods.yaml)

# Feed multiple command outputs into paste (column merge)
paste <(cut -d: -f1 /etc/passwd) <(cut -d: -f3 /etc/passwd)
```

### Pipe and Subshell FD Inheritance

```bash
# Variables set inside a pipe subshell are NOT visible outside
count=0
cat /etc/passwd | while read line; do
  (( count++ ))   # count is in a subshell — parent won't see it
done
echo $count   # still 0!

# Fix: use process substitution instead
count=0
while read line; do
  (( count++ ))
done < <(cat /etc/passwd)
echo $count   # correct!

# Or use lastpipe option (Bash 4.2+) — runs last pipeline cmd in current shell
shopt -s lastpipe
count=0
cat /etc/passwd | while read line; do (( count++ )); done
echo $count   # correct with lastpipe
```

---

## ⚠️ Gotchas & Pro Tips

- **Quote your variables:** `"$var"` prevents word splitting and glob expansion on the value. The rule is: always quote unless you explicitly want splitting/globbing.
- **Brace expansion vs glob:** `{a,b}` is expansion (generates strings); `[ab]` is a glob (matches filesystem). They look similar but work completely differently.
- **`2>&1` order matters:** The shell processes redirections left to right. `cmd > file 2>&1` means "stdout→file, stderr→wherever stdout now points (file)". Reversed: `cmd 2>&1 > file` means "stderr→wherever stdout currently points (terminal), stdout→file".
- **Unquoted `$()` causes word splitting:** `files=$(ls)` then `rm $files` will break on filenames with spaces. Always: `rm "$files"` or better yet, use arrays.
- **Process substitution is not POSIX:** `<(cmd)` works in Bash and Zsh but not in sh/dash. In Docker entrypoints or scripts targeting minimal containers, use temp files or mkfifo instead.
- **`exec` redirects persist:** Using `exec > logfile` redirects ALL subsequent stdout in the script to logfile, including sub-commands. This is powerful for logging entire scripts but easy to accidentally leave open.
- **Here-doc indentation:** Use `<<-EOF` (with a dash) to allow tab-indented heredocs — makes scripts more readable. Note: only *tabs* (not spaces) are stripped.

```bash
# Clean here-doc with tab indentation
if true; then
	cat <<-EOF
		This line will have leading tabs stripped.
		Only tabs, not spaces.
	EOF
fi
```

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
