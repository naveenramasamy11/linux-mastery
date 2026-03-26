# 🐧 fzf — Fuzzy Finding for Everything — Linux Mastery

> **fzf transforms every list-producing command in Linux into an interactive, searchable interface — once you wire it into your shell and workflows, navigating history, files, pods, and AWS resources becomes 10x faster.**

## 📖 Concept

`fzf` is a general-purpose command-line fuzzy finder written in Go. It reads lines from stdin (or from a file), presents them in an interactive TUI where you type to filter by fuzzy matching, and outputs the selected line(s) to stdout. Because it's just a filter between stdin and stdout, it composes with anything that produces lists.

What makes fzf exceptional is its integration with the shell. When installed properly, fzf replaces several built-in shell behaviours: `Ctrl+R` (history search), `Ctrl+T` (file path insertion), and `Alt+C` (cd into a fuzzy-matched directory) all become fast, visual, interactive experiences instead of sequential searches. The underlying fuzzy matching algorithm is extremely fast — it handles millions of lines (like a full git log or a large set of AWS resources) without noticeable lag.

For DevOps and SRE workflows on AWS, fzf is particularly powerful when combined with `kubectl`, `aws`, `git`, and log file inspection. Instead of copy-pasting pod names, instance IDs, or log file paths, you interactively select them from a live-filtered list. This is a productivity multiplier that becomes invisible once integrated — you stop thinking about it and just move faster.

---

## 💡 Real-World Use Cases

- Fuzzy-searching through thousands of lines of CloudWatch logs exported to a file without knowing the exact error message
- Interactively selecting a K8s pod by partial name to exec into, without typing the full name or copy-pasting
- Browsing git log history visually and selecting a commit hash to cherry-pick or check out
- Fuzzy-searching `~/.bash_history` to find a complex `aws` CLI command you ran three weeks ago
- Batch-selecting multiple EC2 instances from a `describe-instances` list to stop or tag them

---

## 🔧 Commands & Examples

### Installation

```bash
# Ubuntu / Debian
apt-get install fzf

# Amazon Linux 2 / RHEL
yum install fzf

# Amazon Linux 2023
dnf install fzf

# macOS
brew install fzf

# From source (latest version, any OS)
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install
# Answer: yes to shell key bindings, yes to fuzzy completion, yes to PATH update

# Verify
fzf --version   # 0.50.0+
```

### Shell Integration (add to ~/.bashrc or ~/.zshrc)

```bash
# Set up key bindings and fuzzy completion
# If installed via package manager, add:
source /usr/share/doc/fzf/examples/key-bindings.bash   # Ubuntu
source /usr/share/fzf/shell/key-bindings.bash           # Amazon Linux

# If installed via ~/.fzf:
[ -f ~/.fzf.bash ] && source ~/.fzf.bash

# After sourcing, these key bindings are active:
# Ctrl+T  — paste a fuzzy-selected file path at the cursor
# Ctrl+R  — fuzzy search history (replaces default history search)
# Alt+C   — cd into a fuzzy-selected directory

# Configure fzf defaults via FZF_DEFAULT_OPTS
export FZF_DEFAULT_OPTS='
  --height 40%
  --layout=reverse
  --border
  --info=inline
  --preview "cat {}"
  --preview-window=right:50%:wrap
  --bind "ctrl-/:toggle-preview"
'

# Use fd or ripgrep as the file source (much faster than find)
export FZF_DEFAULT_COMMAND='fd --type f --hidden --follow --exclude .git'
export FZF_CTRL_T_COMMAND="$FZF_DEFAULT_COMMAND"
```

### Basic fzf Usage

```bash
# Interactive file selector
fzf

# Pass any list to fzf
ls | fzf
cat /etc/hosts | fzf
history | fzf

# Multi-select with Tab key
ls | fzf -m

# Output to a variable
FILE=$(fzf)
echo "Selected: $FILE"

# Open selected file in vim
vim $(fzf)

# Delete a file interactively
rm $(ls *.log | fzf)

# Preview file contents while selecting
fzf --preview 'cat {}'
fzf --preview 'less {}'
fzf --preview 'head -100 {}'

# Start with a query pre-filled
fzf --query "error"
ls /var/log | fzf --query "nginx"

# Select a process to kill
kill -9 $(ps aux | fzf | awk '{print $2}')
```

### Shell History Search (Ctrl+R Enhanced)

```bash
# This replaces Ctrl+R with fzf by default when shell integration is loaded
# Type Ctrl+R and start fuzzy searching history

# Create a custom history search function for more control:
fzf-history() {
  local cmd
  cmd=$(history | sort -rn | awk '{$1=""; print substr($0,2)}' | \
    awk '!seen[$0]++' | \
    fzf --no-sort --tiebreak=index \
        --query "$READLINE_LINE" \
        --bind 'ctrl-r:toggle-sort' \
        --header 'Press CTRL-R to toggle sort by recency')
  if [ -n "$cmd" ]; then
    READLINE_LINE="$cmd"
    READLINE_POINT=${#cmd}
  fi
}
bind -x '"\C-r": fzf-history'
```

### File & Directory Navigation

```bash
# cd into a directory interactively (Alt+C default)
# Or build your own:
fcd() {
  local dir
  dir=$(find ${1:-.} -type d 2>/dev/null | fzf +m) && cd "$dir"
}

# Open a file with preview
fopen() {
  local file
  file=$(fzf --preview 'bat --color=always --line-range :200 {}' \
             --preview-window=right:60%:wrap)
  [ -n "$file" ] && ${EDITOR:-vim} "$file"
}

# Search file CONTENTS with ripgrep, then open
fgrep() {
  local file line
  read file line <<< "$(rg --line-number --column --smart-case "$1" | \
    fzf --delimiter : \
        --preview 'bat --color=always --highlight-line {2} {1}' \
        --preview-window '+{2}-10' | \
    awk -F: '{print $1, $2}')"
  [ -n "$file" ] && ${EDITOR:-vim} "+$line" "$file"
}
```

### Git Integration

```bash
# Fuzzy checkout a branch
fgco() {
  local branch
  branch=$(git branch --all | grep -v HEAD | \
    fzf --height 40% --ansi --preview "git log --oneline --graph --color=always {}" | \
    sed 's/remotes\/origin\///' | \
    xargs echo | xargs)
  git checkout "$branch"
}

# Fuzzy-select a commit and show its diff
fshow() {
  git log --oneline --graph --color=always --all | \
    fzf --ansi --no-sort --reverse --tiebreak=index \
        --preview "git show --color=always {2}" \
        --bind "enter:execute(git show {2})"
}

# Interactive git add (select files to stage)
fga() {
  local files
  files=$(git -c color.status=always status --short | \
    fzf -m --ansi --preview 'git diff --color=always {2}' | \
    awk '{print $NF}')
  [ -n "$files" ] && git add $files
}

# Cherry-pick a commit from another branch
git log --oneline --all | fzf | awk '{print $1}' | xargs git cherry-pick
```

### Kubernetes Integration

```bash
# Select a pod to describe
kd() {
  local namespace pod
  namespace=$(kubectl get namespaces -o name | sed 's/namespace\///' | fzf)
  pod=$(kubectl get pods -n "$namespace" --no-headers | fzf | awk '{print $1}')
  kubectl describe pod -n "$namespace" "$pod"
}

# Interactive pod exec
kexec() {
  local namespace pod container
  namespace=$(kubectl get namespaces -o name | sed 's/namespace\///' | fzf --prompt="Namespace> ")
  pod=$(kubectl get pods -n "$namespace" --no-headers | \
    fzf --prompt="Pod> " --preview "kubectl describe pod -n $namespace {1}" | \
    awk '{print $1}')
  container=$(kubectl get pod "$pod" -n "$namespace" \
    -o jsonpath='{.spec.containers[*].name}' | tr ' ' '\n' | fzf --prompt="Container> ")
  kubectl exec -it -n "$namespace" "$pod" -c "$container" -- bash
}

# Follow logs for a selected pod
klogs() {
  local namespace pod
  namespace=$(kubectl get namespaces -o name | sed 's/namespace\///' | fzf)
  pod=$(kubectl get pods -n "$namespace" --no-headers | fzf | awk '{print $1}')
  kubectl logs -f -n "$namespace" "$pod"
}

# Delete a pod interactively
kdel() {
  local namespace pods
  namespace=$(kubectl get namespaces -o name | sed 's/namespace\///' | fzf)
  pods=$(kubectl get pods -n "$namespace" --no-headers | fzf -m | awk '{print $1}')
  echo "$pods" | xargs kubectl delete pod -n "$namespace"
}

# Select a context
kctx() {
  kubectl config use-context "$(kubectl config get-contexts -o name | fzf)"
}
```

### AWS Integration

```bash
# Select an EC2 instance and SSH to it
faws-ssh() {
  local instance_info ip name
  instance_info=$(aws ec2 describe-instances \
    --query 'Reservations[*].Instances[*].[InstanceId,PrivateIpAddress,Tags[?Key==`Name`].Value|[0],State.Name]' \
    --output text | grep running | \
    fzf --preview "echo 'Instance: {1}\nIP: {2}\nName: {3}'")
  ip=$(echo "$instance_info" | awk '{print $2}')
  [ -n "$ip" ] && ssh "ec2-user@$ip"
}

# Select an SSM-managed instance and start a session
fssm() {
  local instance_id
  instance_id=$(aws ssm describe-instance-information \
    --query 'InstanceInformationList[*].[InstanceId,ComputerName,PlatformName]' \
    --output text | fzf | awk '{print $1}')
  [ -n "$instance_id" ] && aws ssm start-session --target "$instance_id"
}

# Select an S3 bucket and browse it
fs3() {
  local bucket prefix
  bucket=$(aws s3api list-buckets --query 'Buckets[*].Name' --output text | tr '\t' '\n' | fzf)
  while true; do
    prefix=$(aws s3 ls "s3://${bucket}/${prefix}" | fzf --prompt="${bucket}/${prefix}> ")
    if echo "$prefix" | grep -q 'PRE '; then
      prefix="${prefix#*PRE }"
    else
      aws s3 cp "s3://${bucket}/${prefix}" /tmp/ && break
    fi
  done
}

# Select AWS profile to use
faws-profile() {
  export AWS_PROFILE=$(grep '\[' ~/.aws/config | sed 's/\[profile //;s/\]//' | fzf)
  echo "Using profile: $AWS_PROFILE"
}

# Select a CloudWatch log group
faws-logs() {
  local log_group start_time
  log_group=$(aws logs describe-log-groups \
    --query 'logGroups[*].logGroupName' --output text | tr '\t' '\n' | fzf)
  aws logs tail "$log_group" --follow --format short
}
```

### Advanced fzf Techniques

```bash
# Two-pane: select an item from one list, act on related data
# Example: select a file from git status, show its diff in preview
git diff --name-only | \
  fzf --preview "git diff --color=always {}" \
      --bind "enter:execute(git add {})+reload(git diff --name-only)"

# Reload the list dynamically as you take actions
# This is the --reload pattern (fzf 0.36+):
fzf --bind "ctrl-r:reload(kubectl get pods --all-namespaces --no-headers 2>&1)"

# Batch operations with multi-select
# Example: select multiple log files to archive
find /var/log -name "*.log" | fzf -m | xargs -I{} gzip {}

# Use fzf as a menu for scripts
choose_env() {
  echo -e "development\nstaging\nproduction" | fzf --prompt="Select environment: "
}
ENVIRONMENT=$(choose_env)
echo "Deploying to $ENVIRONMENT"

# fzf with a custom header
fzf --header "Use TAB to multi-select, Enter to confirm, Ctrl+C to cancel"
```

---

## ⚠️ Gotchas & Pro Tips

- **Always source shell integration after installing:** `fzf --install` or the package manager install adds fzf to PATH, but the key bindings (`Ctrl+R`, `Ctrl+T`, `Alt+C`) require sourcing `key-bindings.bash`/`key-bindings.zsh`. Add the source line to your `~/.bashrc` and verify with a new shell.

- **`fd` and `bat` make fzf dramatically better:** Install `fd` (fast find replacement) for `FZF_DEFAULT_COMMAND` and `bat` (cat with syntax highlighting) for `--preview`. Without them, fzf still works but the file preview experience is grey and slow.

- **Use `--preview` thoughtfully in production scripts:** Preview commands run for every highlighted item. If your preview command is `kubectl describe pod {}`, that's an API call per item as you scroll. Cache where possible or use lightweight previews.

- **Multi-select (`-m`) outputs newlines between selections:** When using multi-select, fzf outputs one line per selection. If piping to `xargs`, use `xargs -d '\n'` to handle filenames with spaces correctly. Or use `fzf -m --print0` paired with `xargs -0`.

- **CTRL+T and ALT+C need `$FZF_CTRL_T_COMMAND` and `$FZF_ALT_C_COMMAND`:** The default uses `find` which can be slow on large filesystems. Set these to `fd` for dramatically faster response.

- **Dotfiles management:** Put your fzf functions in `~/.bash_functions` or `~/.zsh_functions` and source them from `.bashrc`/`.zshrc`. Using `stow` or a dotfiles repo keeps these portable across your EC2 bastion hosts, dev machines, and containers.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
