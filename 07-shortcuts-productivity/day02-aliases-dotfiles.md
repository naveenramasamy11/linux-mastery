# ⚡ Day 02 — Aliases, Dotfiles & Shell Superpowers

> Your shell config is your most-used tool. Most engineers treat it like an afterthought. This guide turns your `.bashrc`/`.zshrc` into a force multiplier.

---

## 🧭 Why Aliases & Dotfiles Matter

Every time you type `cd /var/log && ls -lhrt | tail -20` manually, you're burning time you'll never get back. Power users automate their own workflows just as aggressively as they automate production systems.

This guide covers:
- Alias patterns that actually survive production use
- Functions for what aliases can't do
- `.bashrc`/`.zshrc` tricks that change how you work
- Dotfile management with Git (version-controlled shell config)
- AWS-specific shortcuts every cloud engineer should have

---

## 🔤 Alias Fundamentals

### Basic alias syntax

```bash
# Permanent aliases go in ~/.bashrc or ~/.bash_aliases
alias ll='ls -lhF --color=auto'          # human-readable, type indicators
alias la='ls -lhAF --color=auto'         # includes hidden files
alias lt='ls -lhrt --color=auto'         # sort by time, newest last
alias l.='ls -d .* --color=auto'         # show only dotfiles

# Navigation shortcuts
alias ..='cd ..'
alias ...='cd ../..'
alias ....='cd ../../..'
alias -- -='cd -'                        # go back to previous dir

# Safety nets — add -i for interactive confirmation
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'
alias ln='ln -i'
```

### Reload your config without re-logging in

```bash
alias reload='source ~/.bashrc && echo "✓ Shell config reloaded"'
alias bashrc='${EDITOR:-vim} ~/.bashrc'  # edit and reload in one step
alias vimbash='vim ~/.bashrc && source ~/.bashrc'
```

### Gotcha: aliases don't work inside scripts

Aliases are shell-specific expansions — they only work interactively. For scripts, use **functions** or full paths. If you need the same shortcuts inside scripts, use a shared functions file instead.

---

## 🔧 Functions: When Aliases Aren't Enough

Aliases can't accept arguments. Functions can do anything a shell script can.

### Create a directory and immediately enter it

```bash
mkcd() {
    mkdir -p "$@" && cd "$_"
    # $_ expands to the last argument of the previous command
}
# Usage:
mkcd /tmp/project/subdir   # creates + cds in one shot
```

### Find and kill a process by name

```bash
killit() {
    local name="${1:?Usage: killit <process-name>}"
    local pids
    pids=$(pgrep -f "$name")
    if [[ -z "$pids" ]]; then
        echo "No process matching: $name"
        return 1
    fi
    echo "Killing PIDs: $pids"
    echo "$pids" | xargs kill -9
}
# Usage:
killit stuck-java-app
```

### Extract any archive type without remembering flags

```bash
extract() {
    case "$1" in
        *.tar.gz|*.tgz)   tar xzf "$1"   ;;
        *.tar.bz2|*.tbz2) tar xjf "$1"   ;;
        *.tar.xz)          tar xJf "$1"   ;;
        *.tar)             tar xf "$1"    ;;
        *.gz)              gunzip "$1"    ;;
        *.bz2)             bunzip2 "$1"   ;;
        *.zip)             unzip "$1"     ;;
        *.7z)              7z x "$1"      ;;
        *)  echo "Cannot extract: $1" ;;
    esac
}
# Usage:
extract archive.tar.gz
extract data.zip
```

### Quick HTTP server in current directory

```bash
serve() {
    local port="${1:-8000}"
    echo "Serving $(pwd) on http://0.0.0.0:${port}"
    python3 -m http.server "$port"
}
# Useful for quickly sharing files to another host in the same network
```

### Repeat a command N times

```bash
repeat() {
    local n="${1:?Usage: repeat <n> <command...>}"
    shift
    for _ in $(seq "$n"); do
        "$@"
    done
}
# Usage:
repeat 5 curl -s http://localhost:8080/healthz   # test health endpoint 5 times
```

---

## 🌍 Environment Variables That Matter

```bash
# ~/.bashrc or ~/.bash_profile

# Default editor — affects git commit, crontab -e, visudo, etc.
export EDITOR=vim
export VISUAL=vim

# Better history
export HISTSIZE=100000          # number of commands in memory
export HISTFILESIZE=200000      # number of commands on disk
export HISTCONTROL=ignoredups:erasedups  # no duplicate entries
export HISTTIMEFORMAT="%Y-%m-%d %H:%M:%S  "  # timestamps in history

# Flush history from memory to disk immediately (survives crashes)
export PROMPT_COMMAND="history -a; history -c; history -r; $PROMPT_COMMAND"

# Colored man pages
export LESS_TERMCAP_mb=$'\e[1;32m'
export LESS_TERMCAP_md=$'\e[1;32m'
export LESS_TERMCAP_me=$'\e[0m'
export LESS_TERMCAP_se=$'\e[0m'
export LESS_TERMCAP_so=$'\e[01;33m'
export LESS_TERMCAP_ue=$'\e[0m'
export LESS_TERMCAP_us=$'\e[1;4;31m'
```

---

## ☁️ AWS Shortcuts Every Cloud Engineer Needs

```bash
# ~/.bashrc — AWS power aliases

# Quick profile switcher
awsprofile() {
    export AWS_PROFILE="${1:?Usage: awsprofile <profile-name>}"
    echo "AWS profile set to: $AWS_PROFILE"
    aws sts get-caller-identity --query 'Arn' --output text
}

# Who am I right now?
alias whoaws='aws sts get-caller-identity'

# Quick EC2 instance listing with state and IP
alias ec2ls='aws ec2 describe-instances \
    --query "Reservations[].Instances[].[InstanceId,State.Name,PrivateIpAddress,Tags[?Key==\`Name\`].Value|[0]]" \
    --output table'

# Tail CloudWatch logs (requires awslogs or aws-cli v2)
cwlogs() {
    aws logs tail "${1:?Usage: cwlogs <log-group>}" --follow
}

# Get SSM Parameter Store value
getparam() {
    aws ssm get-parameter --name "${1}" --with-decryption --query 'Parameter.Value' --output text
}

# List running ECS tasks in a cluster
ecstasks() {
    local cluster="${1:?Usage: ecstasks <cluster-name>}"
    aws ecs list-tasks --cluster "$cluster" \
        | jq -r '.taskArns[]' \
        | xargs aws ecs describe-tasks --cluster "$cluster" --tasks \
        | jq -r '.tasks[] | [.taskArn, .lastStatus, .containers[0].name] | @tsv'
}

# Assume an IAM role and export credentials
assumerole() {
    local role_arn="${1:?Usage: assumerole <role-arn>}"
    local creds
    creds=$(aws sts assume-role --role-arn "$role_arn" \
        --role-session-name "cli-session-$(date +%s)" \
        --query 'Credentials' --output json)
    export AWS_ACCESS_KEY_ID=$(echo "$creds" | jq -r '.AccessKeyId')
    export AWS_SECRET_ACCESS_KEY=$(echo "$creds" | jq -r '.SecretAccessKey')
    export AWS_SESSION_TOKEN=$(echo "$creds" | jq -r '.SessionToken')
    echo "✓ Assumed role: $role_arn"
    echo "  Expires: $(echo "$creds" | jq -r '.Expiration')"
}

# Clear assumed role credentials
alias unassumerole='unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN && echo "✓ Cleared assumed role"'
```

---

## 🖥️ Prompt Engineering — PS1 That Shows What You Need

A good prompt shows: hostname, user, directory, git branch, and AWS profile.

```bash
# Lightweight git branch in prompt
git_branch() {
    git branch 2>/dev/null | grep '^*' | colrm 1 2
}

# Color codes (won't corrupt readline like raw escapes)
RED='\[\033[0;31m\]'
GREEN='\[\033[0;32m\]'
YELLOW='\[\033[1;33m\]'
BLUE='\[\033[0;34m\]'
CYAN='\[\033[0;36m\]'
RESET='\[\033[0m\]'

# Prompt: user@host:dir (git-branch) [AWS_PROFILE] $
export PS1="${GREEN}\u@\h${RESET}:${BLUE}\w${RESET}${YELLOW}\$(git_branch | sed 's/.\+/ (&)/')${RESET}${CYAN}\${AWS_PROFILE:+ [$AWS_PROFILE]}${RESET} \$ "
```

**Pro tip:** For more advanced prompts, look into `starship` (a cross-shell prompt written in Rust) — it auto-detects git, AWS, k8s context, and more with near-zero config.

---

## 📁 Organizing Your Dotfiles

Don't manage dotfiles by hand — version-control them.

### The simple Git bare repo approach (no symlink manager needed)

```bash
# First-time setup on a new machine
git init --bare $HOME/.dotfiles
alias dotfiles='/usr/bin/git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'
dotfiles config status.showUntrackedFiles no  # don't show all home dir files

# Add this alias to your .bashrc so it persists
echo "alias dotfiles='/usr/bin/git --git-dir=\$HOME/.dotfiles/ --work-tree=\$HOME'" >> ~/.bashrc

# Track files
dotfiles add ~/.bashrc
dotfiles add ~/.vimrc
dotfiles add ~/.gitconfig
dotfiles commit -m "Initial dotfiles"
dotfiles remote add origin git@github.com:youruser/dotfiles.git
dotfiles push -u origin main
```

### Restore dotfiles on a new machine

```bash
# Clone on a new machine
git clone --bare git@github.com:youruser/dotfiles.git $HOME/.dotfiles
alias dotfiles='/usr/bin/git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'
dotfiles checkout

# If there are conflicts with existing files, back them up first
mkdir -p ~/.dotfiles-backup
dotfiles checkout 2>&1 | grep "^\s" | awk '{print $1}' \
    | xargs -I{} mv {} ~/.dotfiles-backup/{}
dotfiles checkout
```

---

## 🔍 Directory & Navigation Tricks

```bash
# Jump to frequently used dirs using a bookmarks system
export BOOKMARKS_DIR="$HOME/.bookmarks"
mkdir -p "$BOOKMARKS_DIR"

bm() {   # bookmark current directory
    ln -sfn "$(pwd)" "$BOOKMARKS_DIR/${1:?Usage: bm <name>}"
    echo "Bookmarked $(pwd) as '$1'"
}

go() {   # jump to a bookmark
    local target="$BOOKMARKS_DIR/${1:?Usage: go <name>}"
    [[ -L "$target" ]] || { echo "Bookmark not found: $1"; return 1; }
    cd "$(readlink "$target")"
}

bls() {  # list all bookmarks
    ls -la "$BOOKMARKS_DIR" | awk '{print $NF, "->", $(NF-2)}' | tail -n +2
}
```

### Using `CDPATH` for instant navigation

```bash
# CDPATH lets `cd` search in specified directories first
export CDPATH=".:$HOME:$HOME/projects:/etc:/var/log"

# Now you can do:
cd nginx         # jumps to /etc/nginx if it exists
cd myapp         # jumps to ~/projects/myapp
# No need to type full paths for frequently accessed roots
```

---

## 🚦 Useful Aliases for SRE/DevOps Daily Use

```bash
# Logs
alias tailf='tail -f'
alias syslog='sudo tail -f /var/log/syslog'
alias messages='sudo tail -f /var/log/messages'
alias kernlog='sudo dmesg -T --follow'

# Network
alias ports='ss -tulnp'                         # listening ports with process
alias myip='curl -s https://ipinfo.io/ip'       # public IP
alias localip="ip route get 1 | awk '{print \$7}' | head -1"

# Disk
alias df='df -hT --exclude-type=tmpfs --exclude-type=devtmpfs'
alias du1='du -h --max-depth=1 | sort -h'      # top-level dir sizes
alias biggest='du -sh * | sort -h | tail -20'   # 20 biggest items in cwd

# Process
alias psmem='ps aux --sort=-%mem | head -15'    # top memory consumers
alias pscpu='ps aux --sort=-%cpu | head -15'    # top CPU consumers
alias psg='ps aux | grep -v grep | grep'         # usage: psg java

# Git shortcuts
alias gs='git status'
alias gl='git log --oneline --graph --decorate --all'
alias gd='git diff'
alias gds='git diff --staged'
alias gco='git checkout'
alias gcb='git checkout -b'
alias gp='git push'
alias gpu='git pull'
alias gca='git commit -a -m'

# Kubernetes
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods --all-namespaces'
alias kdp='kubectl describe pod'
alias klf='kubectl logs -f'
alias kns='kubectl config set-context --current --namespace'  # switch namespace
```

---

## 🛡️ Gotchas and Common Mistakes

**1. Alias expansion doesn't work in scripts**
```bash
# This won't use your 'll' alias inside a script:
#!/bin/bash
ll /etc   # error: ll: command not found
# Fix: use the actual command or source your aliases explicitly
```

**2. Export vs. no export**
```bash
# Without export, variable is not available to child processes
MY_VAR="hello"
bash -c 'echo $MY_VAR'  # empty — child shell doesn't see it

# With export:
export MY_VAR="hello"
bash -c 'echo $MY_VAR'  # prints: hello
```

**3. Single vs. double quotes in aliases**
```bash
# Single quotes: expanded at USE time (correct for most cases)
alias now='echo $(date)'   # date is evaluated each time you run 'now'

# Double quotes: expanded at DEFINITION time (usually wrong)
alias now="echo $(date)"   # date is captured once when .bashrc is sourced — stale!
```

**4. Order matters in `.bashrc`**
Load order: environment variables → PATH additions → functions → aliases → prompt. If your alias depends on a variable, define the variable first.

---

## 🧪 Testing Your Config Without Restarting

```bash
# Test in a subshell — doesn't affect your current session
bash --rcfile ~/.bashrc

# Or source just your aliases file
source ~/.bash_aliases

# Debug: trace every command as bashrc loads
bash -x ~/.bashrc 2>&1 | head -50
```

---

## 📦 Recommended `.bashrc` Structure

```
~/.bashrc
  ├── Environment variables (PATH, EDITOR, HISTSIZE, etc.)
  ├── Source system-wide bashrc (/etc/bashrc)
  ├── Source ~/.bash_aliases (keep aliases in a separate file)
  ├── Source ~/.bash_functions (keep functions in a separate file)
  ├── Completion scripts (kubectl, aws, terraform, etc.)
  └── Prompt (PS1)
```

Split into separate files — your main `.bashrc` becomes a clean orchestrator:

```bash
# ~/.bashrc
[[ -f ~/.bash_aliases ]]   && source ~/.bash_aliases
[[ -f ~/.bash_functions ]] && source ~/.bash_functions
[[ -f ~/.bash_aws ]]       && source ~/.bash_aws     # AWS-specific shortcuts
[[ -f ~/.bash_k8s ]]       && source ~/.bash_k8s     # K8s shortcuts
```

---

## ⚡ Enable Auto-Completions

```bash
# In ~/.bashrc — enable completions for common tools
[[ -r /usr/share/bash-completion/bash_completion ]] && \
    source /usr/share/bash-completion/bash_completion

# kubectl completion (run once, save to file)
kubectl completion bash > ~/.kubectl-completion
echo 'source ~/.kubectl-completion' >> ~/.bashrc

# AWS CLI v2 completion
complete -C aws_completer aws

# Terraform completion
complete -C terraform terraform

# Git completion (if not already enabled by system)
[[ -f /usr/share/bash-completion/completions/git ]] && \
    source /usr/share/bash-completion/completions/git
```

---

## 🔗 AWS/Cloud Tie-In

In cloud environments, dotfiles matter even more because you often jump between EC2 instances, containers, and cloud shells. Best practices:

- Store your dotfiles in a private GitHub/CodeCommit repo
- Use an EC2 user data script or SSM Run Command to bootstrap dotfiles on new instances
- In ECS/Kubernetes debug containers, having a minimal `.bashrc` in your base image saves significant troubleshooting time
- AWS CloudShell persists `$HOME` across sessions — set up your dotfiles there once

```bash
# Bootstrap script for new EC2 instances (add to user data or SSM)
#!/bin/bash
curl -sL https://raw.githubusercontent.com/youruser/dotfiles/main/install.sh | bash
```

---

## Next: [Day 03 → tmux Advanced — Session Persistence & Pair Debugging](../07-shortcuts-productivity/tmux-cheatsheet.md)

> **Repo:** [github.com/naveenramasamy11/linux-mastery](https://github.com/naveenramasamy11/linux-mastery)
