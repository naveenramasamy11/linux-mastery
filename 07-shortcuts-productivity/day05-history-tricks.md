# 🐧 History Tricks & Shell Productivity — Linux Mastery

> **Your shell history is a searchable log of everything you've ever done — treat it like a database, not a throwaway buffer, and it becomes one of your most powerful productivity tools.**

## 📖 Concept

Bash history is managed by the `readline` library and the `history` built-in. By default it's configured poorly: 500-line limit, no timestamps, duplicates clogging the list, and lost on parallel sessions. With the right `HISTCONTROL`, `HISTSIZE`, `HISTFILESIZE`, and `HISTTIMEFORMAT` settings, your history becomes a permanent, searchable audit trail of every command you've run.

The **`HISTIGNORE`** variable lets you exclude sensitive commands (passwords in CLI args) and noise (simple `ls`, `cd`). **`PROMPT_COMMAND`** with `history -a` ensures every command is immediately flushed to the history file, so parallel terminal sessions don't overwrite each other.

**fzf** (fuzzy finder) transforms `Ctrl+R` from a slow linear search into an interactive fuzzy search across your entire history — one of the highest-ROI shell improvements you can make. Combined with `zoxide` for directory jumping and shell aliases/functions for common patterns, you can dramatically cut the time spent typing repetitive commands.

---

## 💡 Real-World Use Cases

- **Incident response:** Replay exactly what commands were run during an outage with timestamps — `history | grep -A5 "iptables"` or `grep "2024-01-15" ~/.bash_history`.
- **Audit trail:** On shared bastion hosts, per-user history with timestamps is a lightweight audit mechanism.
- **Command archaeology:** Find that complex `jq` filter you wrote 3 months ago — `history | grep jq | grep -i "accounts"`.
- **Reproduce deployments:** Pull exact `helm upgrade` or `kubectl apply` flags used in previous deployments.
- **Security:** `HISTIGNORE` prevents AWS secret keys passed as CLI args from persisting in history files.

---

## 🔧 Commands & Examples

### Essential History Configuration (~/.bashrc)

```bash
# --- Add to ~/.bashrc ---

# Large history (effectively unlimited)
export HISTSIZE=1000000
export HISTFILESIZE=1000000

# Timestamps on every history entry (shows when commands ran)
export HISTTIMEFORMAT="%F %T  "
# Example output:
# 2024-01-15 14:32:01  kubectl get pods -n prod
# 2024-01-15 14:32:45  helm upgrade myapp ./chart -n prod

# Deduplication and cleanup
# ignoredups = don't record consecutive duplicates
# ignorespace = commands starting with space are NOT recorded (use for secrets)
# erasedups = erase ALL previous duplicates of a command, not just consecutive
export HISTCONTROL=ignoreboth:erasedups

# Ignore common noise
export HISTIGNORE="ls:ls *:ll:la:cd:cd *:pwd:exit:clear:history:history *:man *:cat *"

# Append to history file immediately (critical for multiple terminals)
# Without this, the last terminal to close overwrites history
export PROMPT_COMMAND="history -a; ${PROMPT_COMMAND}"

# Or for more granular control:
# history -a  = append new entries to file
# history -c  = clear in-memory history
# history -r  = reload from file
export PROMPT_COMMAND='history -a; history -c; history -r'
# Note: -c -r reloads file on every prompt, showing history from ALL terminals

# Store history in a custom location (useful for separate per-project histories)
export HISTFILE=~/.bash_history_$(hostname)

# Apply changes to current session
source ~/.bashrc
```

### Secure Commands (Space Prefix)

```bash
# HISTCONTROL=ignorespace means commands starting with space are NOT saved
# Use this for sensitive operations:

 export AWS_SECRET_ACCESS_KEY="super-secret-key"   # Leading space → not saved
 mysql -u root -pSECRETpassword mydb               # Leading space → not saved
 vault login -method=token token=s.XXXXXXXXXXXX    # Leading space → not saved

# Verify it didn't get saved
history | tail -5   # These commands should not appear
```

### History Navigation and Search

```bash
# Basic history commands
history             # Show all history with line numbers
history 20          # Show last 20 entries
history -c          # Clear in-memory history (does NOT delete file)

# Search history (standard Ctrl+R — reverse incremental search)
# Press Ctrl+R, type search term, press Ctrl+R again to cycle matches
# Press Enter to execute, Ctrl+G to cancel, → or Esc to edit

# Search history with grep
history | grep kubectl | grep "rollout"
history | grep -E "aws|terraform" | grep "prod" | tail -20

# Execute history entries
!!                    # Repeat last command
!-2                   # Repeat 2nd to last command
!k                    # Repeat last command starting with 'k'
!kubectl              # Repeat last kubectl command
!42                   # Execute history line 42
!?rollout?            # Execute last command containing "rollout"

# Show what would execute without running (add :p)
!kubectl:p            # Print the kubectl command, don't run it
!!:p                  # Print last command

# Modify and re-execute
^old^new              # Replace "old" with "new" in last command
# Example: you ran: git commit -m "fix typo"
# Now run: git push origin main
# But you want to add --force-with-lease:
^push^push --force-with-lease
```

### Word Designators (History Expansion)

```bash
# These pull parts of the previous command
# Syntax: !!:designator or !n:designator

!!             # Entire previous command
!!:0           # Command name only (first word)
!!:1           # First argument
!!:2           # Second argument
!!:$           # Last argument
!!:*           # All arguments (not command name)
!!:1-3         # Arguments 1 through 3

# Practical examples:
ls /very/long/path/to/somewhere
cd !!:$        # cd to the path you just listed (cd /very/long/path/to/somewhere)
# Shorthand: Alt+. (insert last argument of previous command)

# Another example:
vim /etc/nginx/nginx.conf
sudo systemctl reload nginx
# Oops, forgot to check config first:
nginx -t -c !!:$    # nginx -t -c /etc/nginx/nginx.conf

# Modifiers
!!:$:h     # :h = head (dirname) of last arg
!!:$:t     # :t = tail (basename) of last arg
!!:$:r     # :r = remove extension
!!:$:u     # :u = uppercase
!!:$:l     # :l = lowercase

# Example:
tar xzf /downloads/myapp-v2.1.tar.gz
cd !!:$:r:r    # Remove .gz, remove .tar → cd /downloads/myapp-v2.1
```

### fzf — Fuzzy History Search

```bash
# Install fzf
# RHEL/CentOS:
sudo dnf install fzf

# Ubuntu:
sudo apt install fzf

# Amazon Linux 2:
sudo yum install fzf  # or build from source
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install

# Enable fzf shell integration
# Add to ~/.bashrc:
[ -f ~/.fzf.bash ] && source ~/.fzf.bash

# Key bindings after installation:
# Ctrl+R  — fuzzy search history (replaces default reverse-i-search)
# Ctrl+T  — fuzzy search files in current directory (inserts path)
# Alt+C   — fuzzy cd into subdirectory

# fzf with preview (search history with context)
history | fzf --tac --no-sort --preview ''

# fzf history search function for ~/.bashrc
fh() {
    local cmd
    cmd=$(history | awk '{$1=""; print substr($0,2)}' | \
          fzf --tac --no-sort \
              --query "$*" \
              --prompt="history> " \
              --preview-window=hidden \
              --height=40%)
    [ -n "$cmd" ] && echo "$cmd"
}

# Usage: fh kubectl → fuzzy search history for kubectl commands
# Press Enter to echo the command (then you can run it)
```

### History Analytics

```bash
# Top 20 most-used commands
history | awk '{print $4}' | sort | uniq -c | sort -rn | head -20
# (adjust column number if HISTTIMEFORMAT is set — shifts columns)

# With timestamp format, command is in column 4 (date=2, time=3, cmd=4)
history | awk '{print $4}' | sort | uniq -c | sort -rn | head -20

# Commands by hour of day (find your busy hours)
history | awk '{print $2}' | cut -d: -f1 | sort | uniq -c

# Find commands that failed (exit code in PS1, not history — use another approach)
# Enable exit code logging in PROMPT_COMMAND:
export PROMPT_COMMAND='
    EXIT_CODE=$?
    [ $EXIT_CODE -ne 0 ] && echo "$(date +%F\ %T) EXIT:$EXIT_CODE $(history 1)" >> ~/.bash_errors
    history -a
'

# Most common kubectl namespaces
history | grep kubectl | grep -oP '\-n\s+\K\S+' | sort | uniq -c | sort -rn

# Find commands that took a long time (set DEBUG trap)
# Add to ~/.bashrc:
trap 'CMD_START=$SECONDS' DEBUG
PROMPT_COMMAND='
    CMD_DURATION=$((SECONDS - CMD_START))
    [ $CMD_DURATION -gt 5 ] && echo "Slow command ($CMD_DURATION s): $(history 1)"
    history -a
'
```

### Persistent History Across Sessions

```bash
# ~/.bashrc: comprehensive history setup for production use
cat >> ~/.bashrc << 'EOF'

# === HISTORY CONFIGURATION ===
HISTSIZE=1000000
HISTFILESIZE=1000000
HISTTIMEFORMAT="%F %T  "
HISTCONTROL=ignoreboth:erasedups
HISTIGNORE="ls:ll:la:cd:pwd:exit:clear:history"

# Immediate append to history file, reread for all terminals
shopt -s histappend
PROMPT_COMMAND="history -a; ${PROMPT_COMMAND}"

# Enable extended glob and multi-line commands
shopt -s cmdhist        # Save multi-line commands as single entry
shopt -s lithist        # With actual newlines (not ;)
shopt -s histverify     # Show !expansion before executing (type twice to run)
EOF

source ~/.bashrc

# Verify
echo $HISTTIMEFORMAT
echo $HISTSIZE
```

---

## ⚠️ Gotchas & Pro Tips

- **`erasedups` vs `ignoredups`:** `ignoredups` only ignores consecutive duplicates (`ls; ls; ls` → one entry). `erasedups` removes ALL previous occurrences of the same command, so history stays clean but older timestamps are lost. Choose based on whether you care about "when did I first run this" vs "my history is cluttered".
- **Parallel terminal sessions overwrite each other by default.** Without `shopt -s histappend` + `history -a` in PROMPT_COMMAND, the last shell to close wins. Always use both.
- **`Ctrl+R` only searches backwards.** For forward search, press `Ctrl+S` — but this requires `stty -ixon` to disable XON/XOFF flow control first. Add `stty -ixon` to your `~/.bashrc`.
- **`HISTIGNORE` uses glob patterns.** `"ls *"` ignores `ls -la` but NOT `ls` alone (use `"ls:ls *"` for both). Test with `echo $HISTIGNORE`.
- **History file is plaintext and world-readable by default.** Harden with `chmod 600 ~/.bash_history`. Also consider that `~/.bash_history` on shared systems is a security concern — use PAM-based logging (`pam_tty_audit`) for proper audit trails.
- **Pro tip — `HISTFILE` per project:**

```bash
# In project-specific .envrc (direnv):
export HISTFILE="$PWD/.dir_history"
# Now `history` shows only commands run in this project directory
```

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
