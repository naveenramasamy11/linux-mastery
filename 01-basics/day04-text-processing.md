# 🔍 Day 04 — Text Processing Powertools: grep, awk, sed, cut, sort, uniq, tr

> Master these 7 commands and you'll slice through log files, CSVs, and config data like a surgeon. Every SRE and DevOps engineer reaches for these daily — often chained together in ways that replace entire Python scripts.

---

## Table of Contents
1. [grep — Find the Signal in the Noise](#grep)
2. [awk — The Mini Programming Language](#awk)
3. [sed — Stream Edit Without Opening a File](#sed)
4. [cut — Extract Columns in Seconds](#cut)
5. [sort — Order Matters](#sort)
6. [uniq — Deduplicate and Count](#uniq)
7. [tr — Translate Characters](#tr)
8. [Pipeline Combos — Real World Recipes](#pipeline-combos)
9. [AWS / Cloud Tie-ins](#aws-cloud-tie-ins)

---

## grep — Find the Signal in the Noise <a name="grep"></a>

`grep` searches for patterns using regular expressions. It's your first line of triage on any log file.

### Basic Usage

```bash
# Find all lines containing "ERROR"
grep "ERROR" /var/log/app.log

# Case-insensitive match
grep -i "error" /var/log/app.log

# Show 3 lines of context BEFORE and AFTER each match (great for log analysis)
grep -C 3 "OOMKilled" /var/log/syslog

# Show only 5 lines AFTER the match (e.g., stack trace)
grep -A 5 "Exception" app.log

# Show only 2 lines BEFORE the match (e.g., request that caused the error)
grep -B 2 "500 Internal Server Error" nginx/access.log
```

### Recursive and Multi-file Grep

```bash
# Search recursively in all .log files under /var/log
grep -r "segfault" /var/log/ --include="*.log"

# Print filename AND line number for each match
grep -rn "FATAL" /var/log/ --include="*.log"

# Count occurrences per file
grep -rc "WARN" /var/log/ --include="*.log"

# List only filenames that contain the pattern
grep -rl "database connection" /etc/
```

### Invert and Extended Patterns

```bash
# Show lines that DON'T contain "DEBUG" (filter noise)
grep -v "DEBUG" /var/log/app.log

# Extended regex: match ERROR or WARN on same line
grep -E "ERROR|WARN" /var/log/app.log

# Match lines starting with a timestamp pattern (ISO 8601)
grep -E "^[0-9]{4}-[0-9]{2}-[0-9]{2}" /var/log/app.log

# Find IPs in a log file
grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" /var/log/nginx/access.log | sort | uniq -c | sort -rn
```

### Pro Tips
- **`grep -P`** enables Perl-compatible regex (PCRE) — supports `\d`, `\s`, lookaheads, etc.
- **`grep -F`** treats the pattern as a fixed string (no regex). Much faster for plain-string searches on huge files.
- **Gotcha:** `grep` on a binary file outputs garbage. Use `strings <file> | grep <pattern>` to extract readable text first.

---

## awk — The Mini Programming Language <a name="awk"></a>

`awk` processes text line-by-line, splitting each line into fields. Think of it as Excel for the command line. `$1` is the first field, `$2` second, `$NF` is the last field, `NF` is the number of fields.

### Column Extraction

```bash
# Print the 1st and 5th columns of a space-delimited file
awk '{print $1, $5}' file.txt

# Use a custom delimiter (CSV-like): -F sets the field separator
awk -F',' '{print $1, $3}' data.csv

# Print the LAST column regardless of how many columns there are
awk '{print $NF}' file.txt

# Print second-to-last column
awk '{print $(NF-1)}' file.txt
```

### Filtering with Conditions

```bash
# Print lines where the 3rd column is greater than 100 (numeric)
awk '$3 > 100' metrics.txt

# Print lines where the status field (column 9 in nginx logs) is 500
awk '$9 == 500 {print $1, $7, $9}' /var/log/nginx/access.log

# Print lines containing a pattern in a specific column
awk '$1 ~ /^10\.0\./' /var/log/nginx/access.log   # Lines where IP starts with 10.0.

# Skip the header line and process the rest
awk 'NR > 1 {print $2, $4}' report.csv
```

### Aggregation and Arithmetic

```bash
# Sum all values in column 5 (e.g., response sizes)
awk '{sum += $5} END {print "Total bytes:", sum}' /var/log/nginx/access.log

# Count lines matching a pattern
awk '/ERROR/ {count++} END {print count, "errors found"}' app.log

# Average of column 3
awk '{sum += $3; count++} END {print "Average:", sum/count}' metrics.txt

# Print unique values in column 1 (like uniq but per-field)
awk '!seen[$1]++' file.txt
```

### Pro Tips
- **Gotcha:** awk field indexing starts at `$1`, not `$0`. `$0` is the entire line.
- For large files, awk is often **faster than Python pandas** for simple aggregations because it streams.
- Use `awk -v var=value` to pass shell variables into awk: `awk -v threshold=500 '$3 > threshold' file`

---

## sed — Stream Edit Without Opening a File <a name="sed"></a>

`sed` edits text streams in-place or as a pipeline. Perfect for find-and-replace in configs, log sanitization, and bulk edits.

### Find and Replace

```bash
# Replace first occurrence of "foo" with "bar" on each line
sed 's/foo/bar/' file.txt

# Replace ALL occurrences on each line (global flag)
sed 's/foo/bar/g' file.txt

# Case-insensitive replace
sed 's/error/ERROR/gi' file.txt

# Edit the file IN-PLACE (modifies the file, no temp file)
sed -i 's/localhost/db.internal/g' /etc/app/config.ini

# In-place edit with a backup: creates config.ini.bak before modifying
sed -i.bak 's/localhost/db.internal/g' /etc/app/config.ini
```

### Delete Lines

```bash
# Delete blank lines
sed '/^$/d' file.txt

# Delete lines containing "DEBUG"
sed '/DEBUG/d' /var/log/app.log

# Delete lines 1 through 5 (e.g., skip a file header)
sed '1,5d' file.txt

# Delete the last line
sed '$d' file.txt
```

### Pro Tips
- **Gotcha:** On macOS, `sed -i` requires an explicit backup suffix even if you don't want one: `sed -i '' 's/foo/bar/g' file`. On Linux, `sed -i` works without a suffix.
- `sed` uses BRE (basic regex) by default. Use `sed -E` for ERE (extended regex) which supports `+`, `?`, `|`, `()` without backslashes.

---

## cut — Extract Columns in Seconds <a name="cut"></a>

`cut` is simpler than awk but extremely fast for extracting fixed columns or byte ranges.

```bash
# Extract the 1st and 3rd colon-delimited fields from /etc/passwd
cut -d: -f1,3 /etc/passwd           # username and UID

# Extract columns 1 through 5
cut -d, -f1-5 data.csv

# Extract by byte position (useful for fixed-width files)
cut -b1-10 fixed-width.txt

# Get just the hostname from a fully-qualified domain name
echo "db01.prod.us-east-1.internal" | cut -d. -f1   # → db01

# Extract AWS instance IDs from a list (assuming tab-separated output)
aws ec2 describe-instances --query '...' | cut -f2
```

### Pro Tips
- **Gotcha:** `cut` doesn't handle multiple consecutive delimiters as one (e.g., double spaces). Use `awk` when your delimiter is inconsistent whitespace.
- For CSV files with quoted fields containing commas, use `awk -F','` or a proper CSV parser — `cut` will break.

---

## sort — Order Matters <a name="sort"></a>

```bash
# Alphabetical sort (default)
sort names.txt

# Reverse sort
sort -r names.txt

# Numeric sort (important! without -n, "10" sorts before "2")
sort -n numbers.txt

# Sort by the 3rd field, numerically, descending
sort -t',' -k3 -rn data.csv

# Sort by multiple keys: first by column 2, then by column 3
sort -k2,2 -k3,3n file.txt

# Sort and remove duplicates in one step (equivalent to sort | uniq)
sort -u file.txt

# Human-readable size sort (e.g., for du output: 1K, 2M, 3G)
du -sh /var/log/* | sort -h

# Find the top 10 largest files in a directory
du -sh /var/log/* 2>/dev/null | sort -rh | head -10
```

### Pro Tips
- **Gotcha:** Always use `-n` for numeric sorting. `sort` without `-n` is lexicographic — `100` comes before `20`.
- `sort` is stable with `-s` flag, meaning equal elements retain their original order.
- For massive files, `sort` uses disk-based merge sort and can handle files larger than RAM with `--buffer-size` and `--temporary-directory`.

---

## uniq — Deduplicate and Count <a name="uniq"></a>

`uniq` collapses **adjacent** duplicate lines. **Always sort first** unless you specifically want to detect consecutive duplicates only.

```bash
# Remove duplicate lines (file must be sorted)
sort file.txt | uniq

# Count occurrences of each unique line
sort file.txt | uniq -c

# Show only lines that appear MORE than once (duplicates)
sort file.txt | uniq -d

# Show only UNIQUE lines (no duplicates at all)
sort file.txt | uniq -u

# Count and sort by frequency (most common first)
sort file.txt | uniq -c | sort -rn

# Find the top 5 most frequent HTTP status codes
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -5

# Find the top 10 IPs hitting your server
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10
```

### Pro Tips
- **Gotcha:** `uniq` without `sort` only removes **consecutive** duplicates. `aababc` becomes `ababc` — it does NOT fully deduplicate. Always `sort | uniq`.
- `uniq -c` output has leading spaces — pipe through `awk '{print $2, $1}'` to swap count/value if feeding to downstream tools.

---

## tr — Translate Characters <a name="tr"></a>

`tr` translates, squeezes, or deletes characters. Operates on **characters**, not patterns.

```bash
# Convert lowercase to uppercase
echo "hello world" | tr 'a-z' 'A-Z'

# Convert uppercase to lowercase
cat file.txt | tr 'A-Z' 'a-z'

# Replace colons with newlines (useful for parsing $PATH)
echo $PATH | tr ':' '\n'

# Delete all digits from a string
echo "abc123def456" | tr -d '0-9'           # → abcdef

# Squeeze multiple consecutive spaces into one
echo "too   many   spaces" | tr -s ' '      # → too many spaces

# Remove non-printable/control characters (e.g., Windows line endings)
cat windows-file.txt | tr -d '\r'           # Strip carriage returns

# Replace newlines with spaces (join lines into one)
cat file.txt | tr '\n' ' '
```

### Pro Tips
- **Gotcha:** `tr` does NOT support regex. It works on character-by-character translation. For pattern-based replacement use `sed`.
- `tr -s '\n'` squeezes multiple blank lines into one — useful for cleaning up verbose output.
- To count word frequencies, a classic one-liner: `cat file.txt | tr -s ' ' '\n' | sort | uniq -c | sort -rn`

---

## Pipeline Combos — Real World Recipes <a name="pipeline-combos"></a>

This is where it all comes together. These are the kinds of one-liners you'll actually use at 2 AM during an incident.

### Find the Top 10 Slowest API Endpoints

```bash
# Nginx log format: $remote_addr - $remote_user [$time] "$request" $status $bytes $referrer $ua $request_time
awk '{print $NF, $7}' /var/log/nginx/access.log \
  | sort -rn \
  | head -20 \
  | awk '{printf "%.3fs  %s\n", $1, $2}'
```

### Count Errors by Hour

```bash
# Extract hour from timestamp and count ERROR occurrences per hour
grep "ERROR" /var/log/app.log \
  | awk '{print $1}' \
  | cut -dT -f2 \
  | cut -d: -f1 \
  | sort | uniq -c | sort -k2n
```

### Parse /etc/passwd for Human Users (UID >= 1000)

```bash
awk -F: '$3 >= 1000 && $3 < 65534 {print $1, $3, $6, $7}' /etc/passwd \
  | column -t
```

### Find All Unique 4xx Errors and Their Counts

```bash
awk '$9 ~ /^4/ {print $9}' /var/log/nginx/access.log \
  | sort | uniq -c | sort -rn
```

### Strip Comments and Blank Lines from a Config File

```bash
grep -Ev '^\s*(#|$)' /etc/ssh/sshd_config
```

### Extract Failed SSH Login IPs from syslog

```bash
grep "Failed password" /var/log/auth.log \
  | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" \
  | sort | uniq -c | sort -rn \
  | awk '$1 > 5 {print "⚠️  " $1 " attempts from " $2}'
```

---

## AWS / Cloud Tie-ins <a name="aws-cloud-tie-ins"></a>

Text processing shines when working with AWS CLI output and cloud logs.

### Parse AWS CLI Output Without jq

```bash
# List instance IDs and their state
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name]' \
  --output text \
  | awk '{printf "%-20s %s\n", $1, $2}'

# Find all stopped instances
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name]' \
  --output text \
  | awk '$2 == "stopped" {print $1}'
```

### Monitor ALB Access Logs in Real Time

```bash
# ALB logs are space-separated; field 12 is the target response time
tail -f /var/log/alb-access.log \
  | awk '$12 > 1.0 {print "SLOW:", $12"s", $13}'
# Print requests taking over 1 second with the target:port
```

---

## Next:

➡️ [Day 05 — Processes & Jobs: systemd Services, systemctl, journalctl, and Unit Files](../02-processes-jobs/day02-systemd-services.md)
