# ⌨️ tmux Cheatsheet — Terminal Multiplexer Mastery

> tmux is your best friend on remote servers. Sessions survive SSH drops, splits let you multitask like a pro.

---

## Installation & Start

```bash
# Install
apt install tmux    /    brew install tmux

# Start a named session (always name your sessions!)
tmux new -s main
tmux new -s deploy
tmux new -s monitoring

# Attach to existing
tmux attach -t main
tmux a              # attach to last session

# List sessions
tmux ls

# Kill a session
tmux kill-session -t main
```

---

## Prefix Key

Default prefix: **`Ctrl+b`**
All commands below assume you press `Ctrl+b` first, then the key.

> 💡 **Pro tip:** Many users remap prefix to `Ctrl+a` (like GNU screen) by adding to `~/.tmux.conf`:
> ```bash
> unbind C-b
> set-option -g prefix C-a
> bind-key C-a send-prefix
> ```

---

## Sessions

| Action | Keys |
|--------|------|
| New session | `Ctrl+b :new-session -s name` |
| Switch session | `Ctrl+b $` (rename) / `Ctrl+b s` (list & select) |
| Detach (session stays alive!) | `Ctrl+b d` |
| Kill session | `Ctrl+b :kill-session` |

---

## Windows (Tabs)

| Action | Keys |
|--------|------|
| New window | `Ctrl+b c` |
| Rename window | `Ctrl+b ,` |
| Next window | `Ctrl+b n` |
| Prev window | `Ctrl+b p` |
| Jump to window # | `Ctrl+b 0-9` |
| List windows | `Ctrl+b w` |
| Close window | `Ctrl+b &` |

---

## Panes (Splits)

| Action | Keys |
|--------|------|
| Split horizontal (top/bottom) | `Ctrl+b "` |
| Split vertical (left/right) | `Ctrl+b %` |
| Navigate panes | `Ctrl+b ←↑→↓` |
| Swap pane | `Ctrl+b {` / `Ctrl+b }` |
| Zoom pane (fullscreen toggle) | `Ctrl+b z` |
| Close pane | `Ctrl+b x` |
| Convert pane to window | `Ctrl+b !` |
| Resize pane | `Ctrl+b :resize-pane -D/U/L/R 5` |
| Even layout | `Ctrl+b Alt+1` (even-horizontal) |

---

## Copy Mode (scroll, search, copy)

```
Ctrl+b [         → Enter copy mode
Space            → Start selection
Enter            → Copy selection
Ctrl+b ]         → Paste

q or Esc         → Exit copy mode
/                → Search forward
?                → Search backward
n / N            → Next / prev match
```

---

## Power User Config (`~/.tmux.conf`)

```bash
# Sensible defaults
set -g default-terminal "screen-256color"
set -g history-limit 50000
set -g mouse on                          # Enable mouse support!
set -g base-index 1                      # Windows start at 1, not 0
setw -g pane-base-index 1

# Status bar
set -g status-bg colour235
set -g status-fg white
set -g status-left "#[fg=green][#S] "
set -g status-right "#[fg=yellow]%H:%M %d-%b"
set -g status-interval 1                 # Update every second

# Vi mode in copy mode
setw -g mode-keys vi
bind-key -T copy-mode-vi 'v' send -X begin-selection
bind-key -T copy-mode-vi 'y' send -X copy-selection

# Easy splits with | and -
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"

# Vim-style pane navigation
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# Reload config
bind r source-file ~/.tmux.conf \; display "Config reloaded!"
```

---

## Scripting tmux — Automate Your Workspace

```bash
#!/bin/bash
# dev-workspace.sh — launch a full dev setup

SESSION="dev"
tmux new-session -d -s $SESSION -n "editor"
tmux send-keys -t $SESSION "vim ." Enter

tmux new-window -t $SESSION -n "server"
tmux send-keys -t $SESSION "npm run dev" Enter

tmux new-window -t $SESSION -n "logs"
tmux split-window -h -t $SESSION
tmux send-keys -t $SESSION "tail -f /var/log/app.log" Enter
tmux select-pane -t 0
tmux send-keys -t $SESSION "kubectl logs -f deployment/api" Enter

tmux select-window -t $SESSION:1
tmux attach-session -t $SESSION
```

---

## Useful tmux Commands (command mode `Ctrl+b :`)

```bash
:set -g mouse on          # Enable mouse
:setw synchronize-panes   # Type in all panes simultaneously!
:kill-server              # Kill ALL sessions
:pipe-pane -o "cat >> ~/output.log"   # Log pane output to file
:join-pane -s 2           # Merge window 2 into current as pane
```

> 💡 **`synchronize-panes`** is incredibly useful when running the same command on multiple servers in different panes.

---

## tmux + SSH Workflow

```bash
# Connect to remote, start/attach session
ssh user@server "tmux new -A -s main"

# Send a command to a running remote tmux pane
tmux send-keys -t main:0 "sudo systemctl restart nginx" Enter

# Nested tmux? Use Ctrl+b b to pass prefix to inner tmux
```

---

> See also: [Bash Readline Shortcuts](./bash-readline-shortcuts.md)
