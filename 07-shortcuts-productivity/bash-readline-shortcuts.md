# ⚡ Bash Readline Shortcuts — Work at the Speed of Thought

> Stop reaching for the arrow keys. These shortcuts will make you 10x faster at the terminal.

---

## Cursor Movement

| Shortcut | Action |
|----------|--------|
| `Ctrl+a` | Jump to **beginning** of line |
| `Ctrl+e` | Jump to **end** of line |
| `Alt+b` | Move **back** one word |
| `Alt+f` | Move **forward** one word |
| `Ctrl+xx` | Toggle between current pos and beginning |

---

## Editing

| Shortcut | Action |
|----------|--------|
| `Ctrl+k` | Cut from cursor to **end** of line |
| `Ctrl+u` | Cut from cursor to **beginning** of line |
| `Ctrl+w` | Cut the **word** before cursor |
| `Alt+d` | Cut the **word** after cursor |
| `Ctrl+y` | **Yank** (paste) last cut text |
| `Alt+y` | Rotate through kill ring (after `Ctrl+y`) |
| `Ctrl+t` | **Transpose** two characters |
| `Alt+t` | **Transpose** two words |
| `Alt+u` | **Uppercase** current word |
| `Alt+l` | **Lowercase** current word |
| `Alt+c` | **Capitalize** current word |

---

## History Navigation

| Shortcut | Action |
|----------|--------|
| `Ctrl+r` | **Reverse search** through history |
| `Ctrl+s` | Forward search (may need `stty -ixon`) |
| `Ctrl+g` | Cancel history search |
| `Ctrl+p` / `↑` | Previous command |
| `Ctrl+n` / `↓` | Next command |
| `Alt+.` | Insert **last argument** of previous command |
| `Alt+_` | Same as `Alt+.` |
| `!!` | Repeat last command |
| `!$` | Last argument of last command |
| `!^` | First argument of last command |
| `!*` | All arguments of last command |
| `!-2` | Two commands ago |
| `!ssh` | Last command starting with `ssh` |
| `^old^new` | Replace `old` with `new` in last command |

---

## Control

| Shortcut | Action |
|----------|--------|
| `Ctrl+l` | **Clear screen** (keeps current line) |
| `Ctrl+c` | Cancel current command (SIGINT) |
| `Ctrl+z` | Suspend to background (SIGSTOP) |
| `Ctrl+d` | EOF / logout (on empty line) |
| `Ctrl+s` | Pause output (XOFF) |
| `Ctrl+q` | Resume output (XON) |

---

## History Tricks

```bash
# Enable timestamp in history
echo 'export HISTTIMEFORMAT="%F %T "' >> ~/.bashrc

# Bigger history
echo 'export HISTSIZE=100000' >> ~/.bashrc
echo 'export HISTFILESIZE=100000' >> ~/.bashrc

# No duplicates
echo 'export HISTCONTROL=ignoredups:erasedups' >> ~/.bashrc

# Append (don't overwrite) history on exit
echo 'shopt -s histappend' >> ~/.bashrc

# Don't record commands starting with space (private commands!)
echo 'export HISTCONTROL=ignorespace' >> ~/.bashrc
#  history  → show with timestamps
#  history | grep deploy  → find deploy commands

# Immediately sync history (across terminals)
export PROMPT_COMMAND="history -a; history -c; history -r; $PROMPT_COMMAND"
```

---

## Power User Tricks

```bash
# Re-use arguments from last command
ls /very/long/path/to/directory
cd !$                      # cd to the same path
cat !$                     # cat the same path

# Run last command as root
sudo !!

# Replace a word in last command
vim /etc/niginx/nginx.conf    # typo!
^niginx^nginx                 # fix it

# Run command from history by number
history | grep terraform
!142                          # run command #142

# Expand history before executing (safe preview!)
bind '"\C-xp": history-expand-line'   # Ctrl+x p to expand

# Ctrl+r tips
# Press Ctrl+r, start typing
# Ctrl+r again → older matches
# Ctrl+s → newer matches  (need: stty -ixon in .bashrc)
# Enter → execute
# Ctrl+g → cancel, keep typed text
# → → edit before execute
```

---

## fzf — Supercharge Your History Search

```bash
# Install
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install

# Now Ctrl+r opens fuzzy history search
# Ctrl+t — fuzzy file search
# Alt+c  — fuzzy cd
```

---

## Set vi Mode (for vim fans)

```bash
# In ~/.bashrc
set -o vi

# Now:
# Esc → switch to command mode
# i → insert mode
# v → edit command in full vim
# /pattern → search history
# k/j → navigate history
```

---

> See also: [tmux Cheatsheet](./tmux-cheatsheet.md)
