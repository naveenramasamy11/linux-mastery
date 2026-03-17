# 🐧 Vim Essentials — Linux Mastery

> **Master Vim for DevOps: edit configs, code, and YAML at speed without leaving the terminal.**

## 📖 Concept

Vim is the de facto standard editor for systems engineers and DevOps professionals. Whether you're tweaking `/etc/nginx/nginx.conf` on a production server, editing Kubernetes manifests in a distro-less container, or writing deployment scripts in a remote shell, Vim is there. Most modern editors don't work over SSH without serious setup, but Vim is ubiquitous, lightweight, and powerful.

The learning curve is steep: modal editing (insert vs normal vs visual mode) is fundamentally different from modern editors. But once muscle memory develops, Vim's keystroke efficiency becomes obvious. Moving cursor to word 5 on line 20, deleting to the end, and indenting takes 3 keystrokes in Vim versus a dozen clicks and modifier keys in a GUI editor. This efficiency matters when you're editing hundreds of files or fixing urgent production issues over a slow connection.

For DevOps specifically, Vim excels at YAML editing (visual block selection for column operations), config file manipulation (find-replace across sections), and scripting. Combined with command-line tools (grep, sed, awk), Vim is part of a powerful triumvirate for infrastructure work.

---

## 💡 Real-World Use Cases

- **Production config editing**: SSH into a server, `vim /etc/nginx/nginx.conf`, edit, save, reload—all without leaving terminal
- **YAML manipulation**: Edit Kubernetes manifests, Ansible playbooks, Terraform files with efficient YAML-aware editing
- **Diff and merge conflict resolution**: Vim's diff mode shows side-by-side comparisons and makes conflict resolution fast
- **Log file analysis**: Open massive log files, search, filter, and extract data without loading into memory
- **Scripting and development**: Write shell scripts, Python, Go—Vim's extensibility and speed make it viable for serious coding
- **Remote container editing**: In distro-less containers with minimal tools, Vim is often the only editor available

---

## 🔧 Commands & Examples

### Modes: Normal, Insert, Visual, Command

```bash
# Vim has four main modes:

# NORMAL (default, opened with Vim)
# - Navigate, delete, copy, paste, search
# - Every keystroke is a command
# - Press ESC from other modes to return here

# INSERT
# - Type text like a normal editor
# - Enter with 'i' (before cursor) or 'a' (after cursor)
# - Exit with ESC

# VISUAL
# - Select text for bulk operations
# - Enter with 'v' (character), 'V' (line), or Ctrl+v (block)
# - Exit with ESC

# COMMAND (execute ex commands)
# - Type ':' to enter command mode
# - Run commands like :w (write), :q (quit), :s (substitute)
# - Exit with ESC or ENTER

# Typical workflow:
# 1. Start in normal mode (ESC to ensure)
# 2. Navigate to location
# 3. Enter insert mode (i/a) and type
# 4. Exit to normal (ESC)
# 5. Save with :w
```

### Navigation in Normal Mode

```bash
# Arrow keys work but are slow (hands off home row)
# Better: use hjkl (vim way, on home row)
h   # Move left
j   # Move down
k   # Move up
l   # Move right

# Word movement (much faster)
w   # Start of next word
e   # End of current word
b   # Start of previous word
W   # Start of next WORD (space-delimited)
E   # End of current WORD
B   # Start of previous WORD

# Line movement
^   # Start of line (first non-whitespace)
0   # Start of line (column 0)
$   # End of line
g_  # Last non-whitespace character

# File movement
gg  # Start of file
G   # End of file
:20 # Jump to line 20
Ctrl+g  # Show current line number

# Search and jump
/pattern      # Find 'pattern' forward
?pattern      # Find 'pattern' backward
n             # Next match
N             # Previous match
*             # Find word under cursor
#             # Find word under cursor (backward)

# Screen movement (without scrolling cursor position much)
Ctrl+f  # Page forward
Ctrl+b  # Page backward
Ctrl+d  # Half-page down
Ctrl+u  # Half-page up
zz      # Center current line on screen
```

### Editing: Operators and Motions

```bash
# Vim's power: operators + motions
# Delete operator 'd' + motion = delete up to motion
# Copy operator 'y' + motion = copy up to motion
# Change operator 'c' + motion = delete and insert

# Delete examples:
dw      # Delete word
d$      # Delete to end of line
d^      # Delete to start of line
dd      # Delete entire line
3d      # Delete 3 lines (d3d or ddd)
d/pattern   # Delete until pattern
dt)     # Delete until ')'

# Copy (yank) examples:
yw      # Copy word
y$      # Copy to end of line
yy      # Copy entire line
3yy     # Copy 3 lines

# Change (delete and insert) examples:
cw      # Delete word, enter insert mode
c$      # Delete to end, insert mode
ciw     # Change inner word (useful for replacing variable names)
ci"     # Change inside quotes
ca"     # Change around quotes (including quotes)

# Paste
p       # Paste after cursor
P       # Paste before cursor
]p      # Paste with auto-indent

# Combine with counts:
2dw     # Delete 2 words
10dd    # Delete 10 lines
5j      # Move down 5 lines (d + 5j = delete 5 lines)
```

### Visual Mode: Selection and Bulk Operations

```bash
# Enter visual mode
v       # Character-wise selection
V       # Line-wise selection
Ctrl+v  # Block-wise selection (column selection)

# Select and operate
# 1. Enter visual mode (v/V/Ctrl+v)
# 2. Extend selection with hjkl or search
# 3. Press operator (d delete, y copy, c change, : command)

# Example: delete multiple lines
V       # Start line selection
3j      # Extend down 3 lines
d       # Delete all selected lines

# Example: indent multiple lines
V       # Select lines
3j      # Extend selection
>       # Indent right
<       # Indent left

# Block selection for column editing (YAML, config files)
Ctrl+v  # Start block selection
3j      # Extend down 3 lines
2l      # Extend right 2 columns
d       # Delete block
c       # Change block (useful for renaming columns)

# Replace in selection
V       # Select lines
:s/old/new/g    # Replace in selected lines only
```

### Search and Replace (:s)

```bash
# Basic substitution
:s/old/new      # Replace first occurrence on current line
:s/old/new/g    # Replace all occurrences on current line

# Range substitution
:%s/old/new/g   # Replace all in entire file
:10,20s/old/new/g   # Replace in lines 10-20

# Confirm each replacement
:%s/old/new/gc  # 'c' flag prompts for each replacement

# With regex patterns
:%s/^#//g       # Remove all leading '#'
:%s/\s+$//g     # Remove trailing whitespace
:%s/\(.*\) -> \(.*\)/\2: \1/   # Swap captured groups

# Case-insensitive
:%s/old/new/gi  # 'i' flag ignores case

# Use current search pattern
:%s//new/g      # Replace last search pattern with 'new'

# Interactive replacement (useful for review)
:%s/old/new/gce # Confirm each and show context

# Replace in visual selection only
# 1. Enter visual mode, select text
# 2. Type ':s/old/new/g' (only affects selection)
```

### Multiple Files and Buffers

```bash
# Open multiple files at startup
vim file1.txt file2.txt file3.txt

# Navigate between buffers
:bn             # Next buffer
:bp             # Previous buffer
:bl             # Last buffer
:b2             # Jump to buffer 2
:buffers        # List all buffers

# Edit another file from within Vim
:e /path/to/file    # Edit file
:e %:h/other.txt    # Edit another file in same directory

# Split windows
:sp             # Horizontal split
:vsp            # Vertical split
Ctrl+w w        # Switch between splits
Ctrl+w q        # Close current split

# View multiple files side-by-side
:vsp other.txt
Ctrl+w >        # Resize current split larger
Ctrl+w <        # Resize current split smaller

# Tab support (modern Vim)
:tabnew file    # Open file in new tab
gt              # Next tab
gT              # Previous tab
:tabclose       # Close current tab
```

### Macros: Record and Replay

```bash
# Macros: record keystrokes, replay them
q<letter>       # Start recording macro in register <letter>
q               # Stop recording

# Execute macro
@<letter>       # Execute macro once
@@              # Repeat last macro
N@<letter>      # Repeat macro N times

# Practical example: format JSON lines
# Initial line: {"name":"John","age":30}
# Want: one key-value per line

# 1. Position cursor at start of line
# 2. qa (start recording in register 'a')
# 3. Manually format first line:
#    - Place cursor after '{', then 'i' ENTER
#    - After each ',', 'i' ENTER
#    - Format manually
# 4. q (stop recording)
# 5. Go to next line: j
# 6. Repeat: @a (or 5@a for 5 times)

# More efficient: use macro for repetitive edits
# Record once, apply to dozens of lines
```

### .vimrc Essentials for DevOps

```bash
# Example .vimrc configuration
" ~/.vimrc - Essential settings for DevOps work

" Use space as leader (not default \)
let mapleader = " "

" UI improvements
set number              " Line numbers
set relativenumber      " Relative line numbers (jump 5 lines = '5j')
set cursorline          " Highlight current line
set colorscheme desert  " Simple color scheme
set background=dark

" Indentation (crucial for YAML)
set expandtab           " Use spaces instead of tabs
set tabstop=2           " Tabs are 2 spaces
set shiftwidth=2        " Indent by 2
set autoindent          " Auto-indent new lines
set smartindent         " Smart indentation

" Search behavior
set incsearch           " Search as you type
set hlsearch            " Highlight matches
set ignorecase          " Case-insensitive search
set smartcase           " Unless pattern has uppercase

" Editor behavior
set mouse=a             " Mouse support
set clipboard=unnamedplus   " Sync with system clipboard
set showcmd             " Show partial commands
set wildmenu            " Better command-line completion

" Performance
set lazyredraw          " Faster scrolling
set noswapfile          " Disable swap files
set nobackup            " Disable backups

" Useful key mappings
nnoremap <leader>w :w<CR>           " Space+w to save
nnoremap <leader>q :q<CR>           " Space+q to quit
nnoremap <leader>x :x<CR>           " Space+x to save and quit

" Highlight trailing whitespace
match Error /\s\+$/
" Remove trailing whitespace (for cleaning configs)
nnoremap <leader>s :%s/\s\+$//g<CR>

" Toggle paste mode (for pasting without auto-indent)
set pastetoggle=<leader>p

" Easy split navigation
nnoremap <C-h> <C-w>h
nnoremap <C-j> <C-w>j
nnoremap <C-k> <C-w>k
nnoremap <C-l> <C-w>l
```

### Vim Tricks for YAML and Config Editing

```bash
# YAML: maintain indentation
" Add to .vimrc
au BufRead,BufNewFile *.yml,*.yaml setlocal shiftwidth=2 tabstop=2

# Copy entire section (useful for duplicating config blocks)
V       # Select lines
y       # Copy
:20     # Jump to line 20
p       # Paste

# Replace variables throughout config
:%s/old_value/new_value/g      # Replace all instances

# Auto-format JSON (assuming jq installed)
:%!jq .

# Auto-format YAML (assuming yq installed)
:%!yq .

# Sort lines
:sort       # Sort selected/all lines

# Check line endings (important for Windows/Linux differences)
:set fileformat?    # Show current (unix or dos)
:set fileformat=unix    # Convert to Unix (LF)
:set fileformat=dos     # Convert to Windows (CRLF)
```

### The sudo Trick: Editing Protected Files

```bash
# Problem: opened file with vim, forgot sudo, now can't save
# (without closing and reopening)

# Solution: :w !sudo tee %
# This writes current buffer through sudo tee to file

:w !sudo tee %      # Save protected file without reopen
:e!                 # Reload file (refresh after save)

# Add to .vimrc for convenience:
" Allow sudoing write
command! Sudo execute 'write !sudo tee % >/dev/null' | edit!
" Usage: :Sudo (instead of :w !sudo tee %)
```

---

## ⚠️ Gotchas & Pro Tips

- **Hjkl feels slow initially**: It is. But arrow keys take you off home row. After a week, hjkl is faster. Resist the urge to go back.

- **Visual mode for complex edits**: If you're struggling to construct the right motion command, use visual mode instead. Select what you want, then operate.

- **Search highlighting persists**: After search, matches stay highlighted. Clear with `:nohlsearch` or remap to `<leader>/`.

- **Undo/redo**: `u` to undo, `Ctrl+r` to redo. Vim remembers your entire session's undo history (unlike editors that lose undo after close).

- **Register power**: Vim has 26 named registers. `"ay` yanks to register 'a', `"ap` pastes from 'a'. Useful for preserving multiple copy-pastes.

- **Marks for quick jumping**: `ma` marks current position as 'a', ``a` jumps back. Useful in large files for jumping between sections.

- **Block indentation in YAML**: Use `Ctrl+v`, select block, then `>` or `<` to indent/dedent. Faster than manual space-insertion.

- **Vim differences**: Vi is POSIX, Vim extends it. Some distros have minimal Vi that lacks features. Check `vim --version` for features. If missing features, compile from source.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
