# 🐧 zsh, Plugins & Prompt Customization — Linux Mastery

> **Supercharge your shell with zsh, Starship, and plugins that make you 3× faster at the command line.**

## 📖 Concept

bash gets you there. zsh gets you there faster — with better tab completion, plugin ecosystems, spelling correction, and a prompt that gives you real-time context (git branch, Python env, K8s context, AWS profile, last command exit code) without you having to ask. The right shell setup eliminates the cognitive overhead of "what branch am I on?", "which cluster is kubectl pointing at?", and "did that last command succeed?"

**Why zsh over bash?** zsh is fully POSIX-compliant and backward-compatible with bash syntax. Key advantages: glob qualifiers (`ls **/*.py`), better tab completion (menu-style selection), `zmv` for mass renaming, `zsh-syntax-highlighting` that colors commands before you run them (red = command not found, green = valid), and `zsh-autosuggestions` that suggests completions from your history in grey as you type.

**The modern shell stack:**
- **zsh** — the shell itself (default on macOS since Catalina)
- **Oh My Zsh** or **zinit/sheldon** — plugin managers
- **Starship** — a fast, cross-shell prompt written in Rust
- **zsh-autosuggestions** — fish-like history suggestions
- **zsh-syntax-highlighting** — command syntax coloring
- **fzf** — fuzzy finder (already covered, but integrates deeply with zsh)
- **zoxide** (`z`) — smart cd that learns your frequently used directories

On production servers you won't install Oh My Zsh, but understanding your local shell setup makes you more effective, and the same concepts apply to `.bashrc` optimization.

---

## 💡 Real-World Use Cases

- Never type `cd /var/log/nginx` again — `z ng` gets you there after visiting once
- See your current K8s context and AWS profile in your prompt to avoid applying changes to the wrong cluster
- Get history-based command completion — type `kub` and press → to complete the last kubectl command you ran
- Rename 50 files with a pattern in a single `zmv` command instead of a loop
- Use `take` to create a directory and immediately cd into it

---

## 🔧 Commands & Examples

### Installing zsh and Switching

```bash
# Install zsh
apt-get install zsh         # Debian/Ubuntu
yum install zsh             # RHEL/CentOS
brew install zsh            # macOS

# Verify version
zsh --version

# Change default shell (requires logout/login to take effect)
chsh -s $(which zsh)

# Or set zsh as default via /etc/passwd
usermod -s /usr/bin/zsh naveen

# Start zsh without changing default
exec zsh

# Check current shell
echo $SHELL
```

### Oh My Zsh — Quick Setup

```bash
# Install Oh My Zsh (don't run as root on production)
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# Configuration file
~/.zshrc

# Theme (set in .zshrc)
ZSH_THEME="agnoster"          # popular powerline theme
ZSH_THEME="robbyrussell"      # simple default
ZSH_THEME=""                  # disable (use Starship instead)

# Plugins (add to .zshrc)
plugins=(
    git                       # git aliases: gst, gco, gcmsg, gp, gl
    kubectl                   # kubectl aliases: k, kgp, kgs, kdp
    aws                       # AWS profile switching helpers
    terraform                 # tf aliases
    docker                    # docker shortcuts
    history                   # better history search (h, hs, hsi)
    sudo                      # press ESC twice to add sudo to last command
    z                         # smart directory jumping (built into OMZ)
    colored-man-pages         # color man pages
    command-not-found         # suggest package to install on command not found
    fzf                       # fzf key bindings
)
```

### Key Plugins — Install Manually (No OMZ Required)

```bash
# zsh-autosuggestions — history-based suggestions in grey
git clone https://github.com/zsh-users/zsh-autosuggestions \
    ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

# Add to plugins list in .zshrc:
# plugins=(... zsh-autosuggestions)

# Accept suggestion: → (right arrow) or Ctrl+F
# Accept next word: Alt+→

# Configure suggestion strategy
ZSH_AUTOSUGGEST_STRATEGY=(history completion)
ZSH_AUTOSUGGEST_BUFFER_MAX_SIZE=20

# zsh-syntax-highlighting — colors commands as you type
git clone https://github.com/zsh-users/zsh-syntax-highlighting \
    ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

# IMPORTANT: Must be LAST plugin in the list
plugins=(... zsh-syntax-highlighting)

# Colors:
# Green = valid command
# Red = command not found
# Yellow = alias
# Bold = valid path
```

### zoxide — Smart Directory Jumping

```bash
# Install zoxide (faster, smarter alternative to z/autojump)
curl -sSfL https://raw.githubusercontent.com/ajeetdsouza/zoxide/main/install.sh | sh
# or:
apt-get install zoxide
brew install zoxide

# Add to .zshrc
eval "$(zoxide init zsh)"

# Usage — after visiting directories a few times:
z nginx              # cd to the most frecently used directory matching "nginx"
z var log            # cd to directory matching both "var" and "log"
zi                   # interactive mode with fzf — fuzzy select from history

# zi is incredibly useful — it shows all your frecent directories and lets you fuzzy search
# No more tab-completing long paths you rarely type correctly

# Reset zoxide database (if it gets polluted)
zoxide remove /old/path
zoxide edit   # edit the database directly
```

### Starship Prompt — Cross-Shell, Fast, Informative

```bash
# Install Starship
curl -sS https://starship.rs/install.sh | sh

# Add to end of .zshrc (or .bashrc for bash)
eval "$(starship init zsh)"
# or for bash:
eval "$(starship init bash)"

# Configuration file: ~/.config/starship.toml

# Example starship.toml for DevOps/cloud engineers:
cat > ~/.config/starship.toml << 'EOF'
# Prompt format — shows these modules left-to-right
format = """
$username\
$hostname\
$directory\
$git_branch\
$git_status\
$kubernetes\
$aws\
$python\
$terraform\
$cmd_duration\
$line_break\
$character"""

[username]
show_always = false
format = "[$user]($style)@"

[hostname]
ssh_only = true
format = "[$hostname]($style):"

[directory]
truncation_length = 4
truncate_to_repo = true

[git_branch]
symbol = " "
format = "[$symbol$branch]($style) "

[git_status]
format = '([\[$all_status$ahead_behind\]]($style) )'
conflicted = "⚠ "
ahead = "⇡${count}"
behind = "⇣${count}"
diverged = "⇕⇡${ahead_count}⇣${behind_count}"
modified = "✚${count}"
untracked = "?${count}"
staged = "●${count}"

[kubernetes]
disabled = false
format = '[⎈ $context\($namespace\)]($style) '
style = "cyan bold"
# Only show when context is NOT prod (prevent accidental prod operations)
contexts = [
    { context_pattern = ".*prod.*", style = "bold red", symbol = "⎈ PROD " },
    { context_pattern = ".*staging.*", style = "bold yellow" },
]

[aws]
disabled = false
format = '[$symbol($profile)(\($region\))]($style) '
symbol = " "
style = "bold yellow"

[python]
format = '[${symbol}${pyenv_prefix}(${version})(\($virtualenv\))]($style) '
symbol = " "

[terraform]
format = "[$symbol$workspace]($style) "

[cmd_duration]
min_time = 2_000
format = "took [$duration]($style) "
style = "bold yellow"

[character]
success_symbol = "[❯](bold green)"
error_symbol = "[❯](bold red)"
EOF

echo "Starship configured. Restart your shell or run: source ~/.zshrc"
```

### Useful zsh-specific Features

```bash
# Extended globbing (no need for find for simple cases)
setopt EXTENDED_GLOB   # add to .zshrc

# Match all .py files recursively
ls **/*.py

# Match files NOT ending in .py
ls ^*.py

# Match files modified in last 24 hours
ls *(.m-1)   # . = regular files, m-1 = modified within 1 day

# Files larger than 1MB
ls *(.L+1048576)

# zmv — mass rename (zsh's mv with patterns)
autoload -U zmv   # add to .zshrc

# Rename all .txt files to .md
zmv '(*).txt' '$1.md'

# Add prefix to all files
zmv '*' 'backup_$f'

# Rename files with spaces to use underscores
zmv '* *' '${f// /_}'

# take — mkdir + cd combined (comes with OMZ)
take /new/deeply/nested/directory
# equivalent to: mkdir -p /new/deeply/nested/directory && cd /new/deeply/nested/directory

# Spelling correction
setopt CORRECT   # suggest corrections for mistyped commands
setopt CORRECT_ALL  # correct arguments too

# History configuration (add to .zshrc)
HISTSIZE=50000
SAVEHIST=50000
HISTFILE=~/.zsh_history
setopt SHARE_HISTORY          # share history between sessions
setopt HIST_IGNORE_DUPS       # don't save duplicate commands
setopt HIST_IGNORE_SPACE      # don't save commands starting with space
setopt HIST_EXPIRE_DUPS_FIRST # expire duplicates first when trimming history

# Better history search
bindkey '^R' history-incremental-search-backward   # Ctrl+R = backward search
bindkey '^[[A' history-search-backward              # Up arrow = search backward

# Directory stack navigation
setopt AUTO_PUSHD
setopt PUSHD_IGNORE_DUPS
alias d='dirs -v'  # show directory stack
# Press d then number to jump to that directory in the stack
```

### DevOps-Specific Aliases and Functions

```bash
# Add to ~/.zshrc

# K8s shortcuts
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployment'
alias kgi='kubectl get ingress'
alias kgn='kubectl get nodes'
alias kdp='kubectl describe pod'
alias kl='kubectl logs -f'
alias kx='kubectx'   # switch context (requires kubectx)
alias kn='kubens'    # switch namespace (requires kubens)

# AWS shortcuts
alias awsid='aws sts get-caller-identity'
alias awsregion='aws configure get region'
function awsprofile() { export AWS_PROFILE=$1; aws sts get-caller-identity; }

# Git shortcuts
alias gs='git status'
alias gd='git diff'
alias gdc='git diff --cached'
alias gco='git checkout'
alias gcb='git checkout -b'
alias gp='git push'
alias gl='git log --oneline --graph --decorate -20'

# Docker
alias dps='docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}"'
alias dex='docker exec -it'
alias dlogs='docker logs -f'

# System
alias ll='ls -lah'
alias la='ls -la'
alias ..='cd ..'
alias ...='cd ../..'
alias ....='cd ../../..'
alias grep='grep --color=auto'
alias df='df -h'
alias du='du -sh'

# SSH agent
function sshadd() {
    eval "$(ssh-agent -s)"
    ssh-add ~/.ssh/id_rsa
}

# Quick port check
function port() {
    ss -tlnp | grep ":$1"
}

# EC2 connect by name tag
function ec2ssh() {
    local name=$1
    local ip=$(aws ec2 describe-instances \
        --filters "Name=tag:Name,Values=$name" "Name=instance-state-name,Values=running" \
        --query 'Reservations[0].Instances[0].PublicIpAddress' \
        --output text)
    echo "Connecting to $name ($ip)..."
    ssh "$ip"
}
```

---

## ⚠️ Gotchas & Pro Tips

- **Oh My Zsh slows startup:** OMZ with many plugins can add 200-500ms to shell startup. Use `time zsh -i -c exit` to measure. For faster setups, use `zinit` (lazy loading) or `sheldon` instead of OMZ. Alternatively, profile with `zsh -xvs` and comment out slow plugins.

- **zsh-syntax-highlighting must be last:** If you put zsh-syntax-highlighting before other plugins, completions break. It must be the very last plugin sourced.

- **Starship AWS module shows ALL AWS commands, not just CLI config:** The AWS profile shown is from `$AWS_PROFILE`, `$AWS_DEFAULT_PROFILE`, or the `[default]` profile. If you assume roles with `aws sts assume-role` but don't set `$AWS_PROFILE`, Starship won't show the assumed role — use a wrapper that sets the env var.

- **`setopt CORRECT` can be annoying:** Spelling correction proposes changes for every mistyped command, including intentional one-off shell tricks. Disable it with `unsetopt CORRECT` if it interrupts your workflow.

- **zoxide replaces z/autojump — don't run both:** Having both `z` (from OMZ) and `zoxide` active causes conflicts. Disable the `z` OMZ plugin if you install zoxide.

- **History sharing (`SHARE_HISTORY`) has tradeoffs:** With `SHARE_HISTORY`, every command in every terminal tab is written to history immediately. This is great for recall but means `history | tail` in one tab shows commands from all other tabs — which can be surprising.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
