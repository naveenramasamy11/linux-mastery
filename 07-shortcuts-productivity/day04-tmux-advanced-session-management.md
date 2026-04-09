# 🐧 tmux Advanced Session Management — Linux Mastery

> **A tmux power user running a multi-server incident response outperforms someone without it by an order of magnitude — persistent sessions, scripted layouts, and synchronized panes change the game.**

## 📖 Concept

tmux is a terminal multiplexer — it lets you run multiple terminal sessions inside a single connection, detach and reattach without losing work, and share sessions between users. While the basics (new window, split pane, detach) are well-known, the advanced features are where real productivity lives.

The tmux object hierarchy is: **server → sessions → windows → panes**. One tmux server process manages all sessions. Sessions are independent workspaces (think: one per project or incident). Windows are tabs within a session. Panes are split regions within a window. This hierarchy lets you organise complex multi-host work: one session per environment (dev/staging/prod), windows per system tier (web/app/db), panes showing live logs and a command prompt side by side.

For DevOps and SRE work, the killer features are: `tmux new-session -s` with scripted layouts (spin up a full monitoring dashboard with one command), `synchronize-panes` (run the same command on 20 servers simultaneously — every pane in sync), named sessions that survive SSH disconnections, and `tmux send-keys` for automation scripts that need to interact with long-running processes.

---

## 💡 Real-World Use Cases

- Script a one-command incident dashboard: 4 panes showing `htop`, `dmesg -w`, app log tail, and a command prompt
- Use `synchronize-panes` to run the same Ansible ad-hoc command across a fleet of EC2 bastions simultaneously
- Keep a persistent `tmux` session on a jump host so long-running Terraform runs or k8s deployments survive network drops
- Share a tmux session with a teammate during a live incident for collaborative debugging (read-only or full access)
- Script tmux layouts for specific project environments that are instantly reproducible across team members

---

## 🔧 Commands & Examples

### Session Management

```bash
# New named session
tmux new-session -s prod-incident
tmux new -s prod-incident    # shorthand

# New session in detached mode (background)
tmux new -s monitor -d

# List sessions
tmux list-sessions
tmux ls

# Attach to a session
tmux attach -t prod-incident
tmux a -t prod-incident

# Attach and detach any other clients (steal session)
tmux attach -t prod-incident -x

# Detach from current session (inside tmux)
# Ctrl-b d

# Kill a session
tmux kill-session -t prod-incident

# Kill all sessions
tmux kill-server

# Rename current session (inside tmux)
# Ctrl-b $

# Switch between sessions (inside tmux)
# Ctrl-b s  (interactive list)
# Ctrl-b (  (previous session)
# Ctrl-b )  (next session)
```

### Window Management

```bash
# New window (inside tmux)
# Ctrl-b c

# Name a window on creation
tmux new-window -n "logs"
tmux new-window -n "app-server" -t prod-incident

# Switch windows
# Ctrl-b 0-9       (by index)
# Ctrl-b n         (next)
# Ctrl-b p         (previous)
# Ctrl-b l         (last used)
# Ctrl-b w         (interactive list)
# Ctrl-b f         (search by name)

# Rename current window
# Ctrl-b ,

# Kill current window
# Ctrl-b &

# Move window to another position
# Ctrl-b .  (enter new index)
tmux move-window -s prod-incident:3 -t prod-incident:0
```

### Pane Management

```bash
# Split vertically (left/right)
# Ctrl-b %

# Split horizontally (top/bottom)
# Ctrl-b "

# Navigate panes
# Ctrl-b arrow keys
# Ctrl-b o          (next pane, cycle)
# Ctrl-b q          (show pane numbers, press number to jump)
# Ctrl-b ;          (last active pane)

# Resize panes
# Ctrl-b Ctrl-arrow  (resize in direction)
# Ctrl-b Alt-arrow   (resize 5 cells at a time)

# Break pane out into its own window
# Ctrl-b !

# Join a window as a pane
tmux join-pane -s prod-incident:2 -t prod-incident:1

# Swap panes
# Ctrl-b {  (swap with previous)
# Ctrl-b }  (swap with next)

# Kill pane
# Ctrl-b x

# Zoom a pane to full window (toggle)
# Ctrl-b z
```

### Synchronize Panes — Run Commands on Multiple Servers

This is the killer feature for fleet management. All panes in the window type simultaneously.

```bash
# Step 1: Create a window with one pane per server
tmux new-session -s fleet -d
tmux rename-window -t fleet:0 "web-tier"

# SSH to each server in a new pane
tmux send-keys -t fleet:web-tier 'ssh ec2-user@web-01.prod' Enter
tmux split-window -t fleet:web-tier -h
tmux send-keys -t fleet:web-tier 'ssh ec2-user@web-02.prod' Enter
tmux split-window -t fleet:web-tier -v
tmux send-keys -t fleet:web-tier 'ssh ec2-user@web-03.prod' Enter
tmux split-window -t fleet:web-tier -h
tmux send-keys -t fleet:web-tier 'ssh ec2-user@web-04.prod' Enter
tmux select-layout -t fleet:web-tier tiled

# Step 2: Enable synchronize-panes
tmux set-window-option -t fleet:web-tier synchronize-panes on

# Now type in any pane — ALL panes receive the keystrokes
# Check nginx status on all 4 servers at once:
# systemctl status nginx

# Step 3: Disable when done
tmux set-window-option -t fleet:web-tier synchronize-panes off

# Bind a key to toggle sync (add to .tmux.conf)
# bind-key S set-window-option synchronize-panes
```

### Scripted Session Layouts

Create reproducible environments with shell scripts:

```bash
#!/bin/bash
# incident-dashboard.sh — one command to set up a full monitoring view

SESSION="incident-$(date +%Y%m%d-%H%M%S)"
HOST=${1:-localhost}

# Create session with first window
tmux new-session -d -s "$SESSION" -n "dashboard"

# Pane 0 (top-left): htop
tmux send-keys -t "$SESSION:dashboard.0" "ssh ${HOST} 'htop'" Enter

# Split horizontally — pane 1 (top-right): dmesg
tmux split-window -t "$SESSION:dashboard" -h
tmux send-keys -t "$SESSION:dashboard.1" "ssh ${HOST} 'dmesg -Tw'" Enter

# Split the left pane vertically — pane 2 (bottom-left): app log
tmux split-window -t "$SESSION:dashboard.0" -v
tmux send-keys -t "$SESSION:dashboard.2" "ssh ${HOST} 'tail -f /var/log/app/app.log'" Enter

# Split the right pane vertically — pane 3 (bottom-right): command prompt
tmux split-window -t "$SESSION:dashboard.1" -v
tmux send-keys -t "$SESSION:dashboard.3" "ssh ${HOST}" Enter

# Even out the layout
tmux select-layout -t "$SESSION:dashboard" tiled

# Second window: network view
tmux new-window -t "$SESSION" -n "network"
tmux send-keys -t "$SESSION:network" "ssh ${HOST} 'watch -n1 ss -tnp'" Enter
tmux split-window -t "$SESSION:network" -h
tmux send-keys -t "$SESSION:network" "ssh ${HOST} 'tcpdump -i any -n -q'" Enter

# Third window: free command prompt
tmux new-window -t "$SESSION" -n "shell"
tmux send-keys -t "$SESSION:shell" "ssh ${HOST}" Enter

# Focus on dashboard
tmux select-window -t "$SESSION:dashboard"
tmux select-pane -t "$SESSION:dashboard.3"

# Attach
tmux attach -t "$SESSION"
```

### tmux.conf — Essential Configuration

```bash
# ~/.tmux.conf

# Change prefix from Ctrl-b to Ctrl-a (screen-like)
unbind C-b
set-option -g prefix C-a
bind-key C-a send-prefix

# Reload config
bind r source-file ~/.tmux.conf \; display "Config reloaded!"

# Enable mouse support (click panes, resize, scroll)
set -g mouse on

# Start window and pane numbering at 1 (easier keyboard reach)
set -g base-index 1
set -g pane-base-index 1
set-window-option -g pane-base-index 1

# Increase history limit (default is 2000 — way too small)
set -g history-limit 50000

# Faster escape time (important for vim/neovim users)
set -sg escape-time 10

# Vi key bindings in copy mode
set-window-option -g mode-keys vi

# Vim-like pane navigation
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# Resize panes with vim-like keys
bind -r H resize-pane -L 5
bind -r J resize-pane -D 5
bind -r K resize-pane -U 5
bind -r L resize-pane -R 5

# Better split bindings (open in current directory)
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"

# New window in current directory
bind c new-window -c "#{pane_current_path}"

# Toggle synchronize-panes
bind S set-window-option synchronize-panes \; display "sync #{?synchronize-panes,ON,OFF}"

# Status bar
set -g status-position top
set -g status-bg colour234
set -g status-fg colour137
set -g status-left '#[fg=colour233,bg=colour241,bold] #S '
set -g status-right '#[fg=colour233,bg=colour241,bold] %d/%m %H:%M:%S '
set -g status-right-length 50
set -g status-left-length 20

# Highlight active window
set-window-option -g window-status-current-style fg=colour81,bg=colour238,bold

# 256 color support
set -g default-terminal "screen-256color"
set-option -sa terminal-features ',xterm-256color:RGB'

# Aggressive resize (useful when multiple clients of different sizes attach)
set-window-option -g aggressive-resize on
```

### Copy Mode & Scrollback

```bash
# Enter copy mode (to scroll up through output)
# Ctrl-b [

# In copy mode with vi bindings:
# j/k       scroll line by line
# Ctrl-d    scroll down half page
# Ctrl-u    scroll up half page
# /         search forward
# ?         search backward
# n/N       next/previous search result
# Space     start selection
# Enter     copy selection to tmux clipboard
# q         exit copy mode

# Paste from tmux clipboard
# Ctrl-b ]

# View clipboard contents
tmux show-buffer

# Save clipboard to file
tmux save-buffer /tmp/tmux-paste.txt

# Copy to system clipboard (requires xclip or pbcopy)
# Add to .tmux.conf:
# bind -T copy-mode-vi Enter send-keys -X copy-pipe-and-cancel "xclip -in -selection clipboard"
```

### Session Sharing — Pair Programming / Incident Response

```bash
# Method 1: Same user, same socket (simplest)
# User A:
tmux new -s shared

# User B (same user, any terminal):
tmux attach -t shared
# Both users see the same session and can type

# Method 2: Different users (read-only for guest)
# User A (owner):
tmux new -s shared
chmod 777 /tmp/tmux-1000/  # allow other users to access socket

# User B (guest, read-only):
tmux -S /tmp/tmux-1000/default attach -t shared -r

# Method 3: tmate (tmux with built-in sharing via SSH/web)
# yum install tmate || apt install tmate
tmate new-session -s incident
# tmate gives you a URL and SSH command to share
# Read-only URL: https://tmate.io/t/XXXXX  
# Full access: ssh session@lon1.tmate.io
```

### tmux Automation — send-keys for Scripting

```bash
# Run a command in a detached session without attaching
tmux new-session -d -s deploy -n deploy
tmux send-keys -t deploy:deploy "terraform apply -auto-approve 2>&1 | tee /tmp/tf-apply.log" Enter

# Check if it's still running
tmux list-panes -t deploy:deploy -F "#{pane_pid} #{pane_current_command}"

# Wait for completion (poll)
while tmux list-panes -t deploy:deploy -F "#{pane_current_command}" | grep -q "terraform"; do
  sleep 10
  echo "Still running..."
done
echo "Done!"

# Capture pane output (last N lines)
tmux capture-pane -t deploy:deploy -p | tail -20

# Capture with history
tmux capture-pane -t deploy:deploy -p -S -1000 > /tmp/deploy-output.txt
```

---

## ⚠️ Gotchas & Pro Tips

- **`tmux attach` vs `tmux attach -x`:** Plain attach allows multiple clients viewing the same session. `-x` detaches all other clients (steal session). When you need sole control during an incident, use `-x`.
- **Synchronize-panes is dangerous:** It's incredibly powerful and easy to accidentally leave on. Set a visual indicator in your status bar: `set -g window-status-current-format '#{?synchronize-panes,#[fg=red]SYNC ,}#W'`.
- **History limit:** The default 2000 lines is far too small for log-heavy sessions. Set `set -g history-limit 100000` in `.tmux.conf`. Memory cost is ~1MB per 10,000 lines — very cheap.
- **TERM variable:** tmux sets `$TERM=screen-256color` inside sessions. Some tools (vim, htop) need this set correctly for colour support. Add `set -g default-terminal "screen-256color"` to your `.tmux.conf`.
- **Nested tmux sessions:** If you SSH to a remote host that also runs tmux, you end up with nested sessions. The inner tmux prefix requires hitting your prefix twice. Use a different prefix on remote hosts or use `Ctrl-b :` to send commands to the outer tmux.
- **Plugin manager (tpm):** Install tmux Plugin Manager (`tpm`) for plugins like `tmux-resurrect` (save/restore sessions across reboots) and `tmux-continuum` (automatic session saving). These are invaluable for long-lived development environments.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
