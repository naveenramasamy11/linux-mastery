# 🐧 sed Patterns & Stream Editing — Linux Mastery

> **sed is the backbone of text transformation pipelines — from config file templating in Ansible to log parsing in CI/CD, a fluent sed practitioner moves faster than anyone reaching for Python.**

## 📖 Concept

`sed` (stream editor) processes text line by line, applying a script of editing commands. It was designed for non-interactive editing of streams — pipeline inputs, files, log output — and it excels at it. Despite being decades old, sed remains indispensable because it's universally available (it's POSIX), extremely fast for large files, and composable with every other Unix tool.

sed operates on a **pattern space** (the current line being processed) and an optional **hold space** (a persistent buffer across lines). Most users only use the substitute command (`s`), but the full sed command set includes deletion, insertion, appending, branching, and multi-line operations — enough to build a state machine for complex text transformations.

In production environments, sed shows up in: Ansible `lineinfile` replacements (which use sed under the hood), Docker `CMD` entrypoints that patch configs at startup, Terraform `templatefile` alternatives for quick substitutions, and CI pipeline scripts that modify version numbers, endpoints, or feature flags in config files before deployment.

The GNU sed on Linux (gsed) has extensions over POSIX sed — in-place editing with `-i`, `\w` word-boundary assertions, and extended regexes with `-E`. macOS ships with BSD sed which has slightly different `-i` syntax. For cross-platform scripts, always note which sed you're targeting.

---

## 💡 Real-World Use Cases

- Patch a config file's endpoint URL during Docker container startup without baking the value into the image
- Mass-replace deprecated API calls across 50 config files in a repo as part of a migration
- Strip ANSI color codes from log output before shipping to CloudWatch Logs
- Comment out or uncomment feature flags in `/etc/sysctl.conf` or `/etc/ssh/sshd_config` during Ansible runs
- Extract specific multi-line blocks from a large log file for incident analysis

---

## 🔧 Commands & Examples

### Substitution Basics (`s` command)

```bash
# Basic substitution (first occurrence per line)
sed 's/foo/bar/' file.txt

# Global substitution (all occurrences per line)
sed 's/foo/bar/g' file.txt

# Case-insensitive substitution
sed 's/foo/bar/gi' file.txt

# Print only lines where substitution was made
sed -n 's/foo/bar/p' file.txt

# Nth occurrence only (e.g., replace 2nd occurrence)
sed 's/foo/bar/2' file.txt

# Replace only on lines matching a pattern (address + substitution)
sed '/^server/s/localhost/10.0.0.1/' config.conf

# Replace using extended regex (-E for ERE, no need to escape + ? | { })
sed -E 's/[0-9]{4}-[0-9]{2}-[0-9]{2}/DATE/' log.txt

# Capture groups with back-references
sed 's/\(first\) \(second\)/\2 \1/'     # GNU basic: swap two words
sed -E 's/(first) (second)/\2 \1/'      # GNU extended: cleaner syntax

# Wrap a value in quotes (config patching)
sed -E 's/(key=)(.*)/\1"\2"/' app.conf

# Replace forward slash (use alternate delimiter)
sed 's|/old/path|/new/path|g' file.txt
sed 's#/old/path#/new/path#g' file.txt
sed 's_old_new_g' file.txt     # any char can be delimiter
```

### In-Place Editing (`-i`)

```bash
# Edit in place (GNU sed) — no backup
sed -i 's/localhost/10.0.0.1/g' /etc/app/config.conf

# Edit in place with backup (GNU sed)
sed -i.bak 's/localhost/10.0.0.1/g' /etc/app/config.conf
# Creates config.conf.bak as backup

# macOS / BSD sed (different -i syntax — backup suffix is mandatory position arg)
sed -i '' 's/localhost/10.0.0.1/g' file.conf   # BSD: empty string = no backup

# Cross-platform safe pattern (create backup explicitly)
cp config.conf config.conf.bak
sed -i 's/localhost/10.0.0.1/g' config.conf

# Mass in-place edit across multiple files
sed -i 's/v1.2.3/v1.3.0/g' configs/*.conf
find /opt/app -name "*.conf" -exec sed -i 's/old_endpoint/new_endpoint/g' {} +

# Edit in place with Ansible-style safety: write to tmp, then mv
sed 's/foo/bar/g' original.conf > /tmp/original.conf.new && mv /tmp/original.conf.new original.conf
```

### Address Ranges — Targeting Lines

```bash
# Act on specific line number
sed '5s/foo/bar/' file.txt       # line 5 only
sed '5,10s/foo/bar/' file.txt    # lines 5 through 10

# From line 5 to end of file
sed '5,$s/foo/bar/' file.txt

# Lines matching a pattern
sed '/^#/s/foo/bar/' file.txt    # only comment lines

# Between two pattern markers (inclusive)
sed '/START/,/END/s/foo/bar/' file.txt

# Between two patterns — common for patching config blocks
sed '/\[database\]/,/\[.*\]/s/host=.*/host=10.0.1.5/' app.ini

# Negate an address (lines NOT matching)
sed '/^#/!s/foo/bar/' file.txt    # all non-comment lines
sed '1!s/foo/bar/' file.txt       # all lines except line 1

# Step addresses: every Nth line (GNU sed)
sed '0~2s/^/# /' file.txt    # comment every other line (0,2,4...)
sed '1~2s/^/# /' file.txt    # every odd line (1,3,5...)
```

### Deletion, Insertion, and Appending

```bash
# Delete lines
sed '/^#/d' file.txt           # delete comment lines
sed '/^$/d' file.txt           # delete empty lines
sed '5,10d' file.txt           # delete lines 5-10
sed '/START/,/END/d' file.txt  # delete block between markers

# Delete empty lines and lines with only whitespace
sed '/^[[:space:]]*$/d' file.txt

# Insert a line BEFORE a match (i command)
sed '/^server_name/i upstream_timeout 30s;' nginx.conf

# Append a line AFTER a match (a command)
sed '/^server_name/a client_max_body_size 100m;' nginx.conf

# Change (replace entire matching line) (c command)
sed '/^timeout=/c timeout=300' app.conf

# Insert at a specific line number
sed '1i #!/bin/bash' script.sh    # prepend shebang to line 1
sed '$a # End of file' file.txt   # append to last line

# Practical: uncomment a line (remove leading #)
sed '/^#net.ipv4.ip_forward/s/^#//' /etc/sysctl.conf

# Comment out a line (add # prefix)
sed '/^PermitRootLogin yes/s/^/# /' /etc/ssh/sshd_config
```

### Printing and Suppression (`-n` with `p`)

```bash
# Default: sed prints every line. -n suppresses automatic printing.
# With -n, you must explicitly print with p.

# Print only matching lines (like grep)
sed -n '/ERROR/p' app.log

# Print lines between markers (inclusive)
sed -n '/BEGIN CERT/,/END CERT/p' server.crt

# Print specific line numbers
sed -n '10,20p' file.txt      # lines 10-20
sed -n '5p' file.txt          # line 5 only
sed -n '$p' file.txt          # last line only

# Print line number and content
sed -n '/ERROR/{=;p}' app.log

# Emulate head -n 50
sed -n '1,50p' large_file.txt
# Or (slightly faster — quit after line 50)
sed '50q' large_file.txt

# Print from pattern to end of file
sed -n '/EXCEPTION/,$p' app.log
```

### Multi-Line Operations (N, P, D)

```bash
# N: append next line to pattern space (join lines for multi-line matching)
# P: print first line of pattern space
# D: delete first line of pattern space, restart cycle

# Join continuation lines (lines ending with \)
sed -e ':a' -e 'N' -e '$!ba' -e 's/\\\n/ /g' file.txt

# Remove newlines (join all lines into one)
sed ':a;N;$!ba;s/\n/ /g' file.txt

# Replace a specific multi-line block
sed '/^Host bastion/{N;s/Port 22/Port 2222/}' ~/.ssh/config

# Find and replace across two consecutive lines
sed 'N;s/foo\nbar/baz\nqux/' file.txt

# Practical: fix broken JSON that has values split across lines
sed ':a;N;$!ba;s/,\n\s*/,/g' broken.json
```

### Hold Space Patterns

```bash
# Hold space is a secondary buffer. h copies pattern to hold, H appends.
# g copies hold to pattern, G appends hold to pattern.

# Reverse file order (tac equivalent)
sed -n '1!G;h;$p' file.txt

# Print line before a matching line (context before)
sed -n '/ERROR/{x;p;x;p}' log.txt   # print previous line then matching line

# Duplicate a line
sed 'h;p' file.txt    # print line, then pattern (h copies to hold, p prints pattern)

# Print every other line
sed -n 'n;p' file.txt  # skip first, print second, skip, print...
```

### Production Config Patching Patterns

```bash
# Patch a value in a key=value config file
KEY="database_host"
VALUE="10.0.1.50"
sed -i "s/^${KEY}=.*/${KEY}=${VALUE}/" /etc/app/database.conf

# Patch a YAML value (simple key: value)
sed -i 's/^\(  host:\s*\).*/\1'"${DB_HOST}"'/' /etc/app/config.yaml

# Replace a URL pattern anywhere in file
sed -i 's|https://old-api.example.com|https://new-api.example.com|g' app.properties

# Set a value only if line exists, add it if not (combine sed + grep)
KEY="max_connections"
VALUE="200"
if grep -q "^${KEY}" config.conf; then
  sed -i "s/^${KEY}=.*/${KEY}=${VALUE}/" config.conf
else
  echo "${KEY}=${VALUE}" >> config.conf
fi

# Remove ANSI escape codes (color codes from command output)
sed -i 's/\x1b\[[0-9;]*m//g' colored_log.txt
# Or more complete:
sed -i 's/\x1b\[[0-9;]*[mGKHFJsu]//g' colored_log.txt

# Extract value from a key=value line
DB_HOST=$(sed -n 's/^database_host=//p' config.conf)
echo "DB host: $DB_HOST"

# Template substitution in Docker entrypoint pattern
# Replace __VAR__ placeholders with env vars
sed -i "s/__DB_HOST__/${DB_HOST}/g; s/__APP_PORT__/${APP_PORT}/g" /etc/app/config.conf

# Strip trailing whitespace
sed -i 's/[[:space:]]*$//' file.txt

# Strip leading whitespace
sed -i 's/^[[:space:]]*//' file.txt

# Strip both
sed -i 's/^[[:space:]]*//;s/[[:space:]]*$//' file.txt
```

### Combining sed in Pipelines

```bash
# Filter and transform in one pipeline
journalctl -u nginx --since "1 hour ago" \
  | sed -n '/error/Ip' \
  | sed 's/.*\(nginx\[.*\]\).*/\1/' \
  | sort | uniq -c | sort -rn

# Process Terraform plan output — extract resource names
terraform plan 2>&1 \
  | sed -n 's/.*# \(.*\) will be.*/\1/p' \
  | sort

# Patch kubeconfig server URL on the fly
kubectl config view --raw \
  | sed "s|https://.*:6443|https://${NEW_API_ENDPOINT}:6443|" \
  > /tmp/new_kubeconfig

# Remove comments and blank lines from config for a clean view
sed '/^#/d; /^$/d' /etc/ssh/sshd_config

# Extract certificate from a PEM bundle
sed -n '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/p' bundle.pem
```

---

## ⚠️ Gotchas & Pro Tips

- **GNU sed vs BSD sed:** The `-i` flag works differently. GNU sed: `sed -i.bak` (backup suffix attached to flag). BSD sed: `sed -i .bak` (backup suffix is a separate argument). For scripts that must work on both, use `sed -i'' -e` (works on GNU) or test for the platform. In CI/CD targeting Amazon Linux or Ubuntu, you always have GNU sed.
- **Regex escaping:** In basic regex (default), `+`, `?`, `|`, `(`, `)` must be backslash-escaped. Use `-E` for extended regex to avoid the backslashes. When in doubt, use `-E`.
- **& in replacement:** `&` represents the entire matched string. `sed 's/[0-9]*/(&)/'` wraps the match in parentheses without needing capture groups.
- **Delimiter choice:** The default delimiter `/` breaks when paths are involved. Use `|`, `#`, or `_` as the delimiter by writing `s|from|to|`. Any character immediately after `s` becomes the delimiter.
- **`-i` modifies the file directly:** There's no undo. For production automation, always take a backup or use version control. Test your sed expression on a copy first.
- **Empty pattern reuse:** An empty pattern `//` reuses the last pattern. `sed '/foo/{s//bar/g}'` is equivalent to `sed 's/foo/bar/g'` — useful in complex scripts to avoid repetition.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
