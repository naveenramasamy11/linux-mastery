# 🐧 awk One-Liners & Field Processing — Linux Mastery

> **awk is the Swiss Army knife of structured text processing — if your data has columns, awk can slice, filter, aggregate, and transform it in a single command that's faster than Python and doesn't require installing anything.**

## 📖 Concept

`awk` is a full programming language optimised for processing structured text. Every awk program follows the pattern `pattern { action }` — for each input line, awk checks all patterns and executes the corresponding action block for any that match. If no pattern is given, the action runs on every line. If no action is given, the default is `print`.

What makes awk so powerful in DevOps and SRE work is its automatic field splitting. By default, awk splits each input line on whitespace and makes fields available as `$1`, `$2`, ... `$NF` (last field), with `$0` being the entire line. This maps perfectly onto the output of tools like `ps`, `df`, `ss`, `kubectl get`, `aws ec2 describe`, and log files — all of which produce whitespace- or delimiter-separated columns.

awk has built-in associative arrays, arithmetic, string functions, and regex matching. For log analysis, metrics extraction, and data transformation in scripts that run on production EC2 instances or in CI pipelines, a well-crafted awk one-liner often replaces 20 lines of Python that requires importing modules and handling edge cases.

---

## 💡 Real-World Use Cases

- Parsing `aws ec2 describe-instances` tab-separated output to extract instance IDs and private IPs
- Summing request sizes from nginx access logs to calculate bandwidth per endpoint
- Extracting slow query durations from PostgreSQL logs and computing P95/P99 latencies
- Filtering `kubectl get pods` output to find all pods not in Running state
- Processing CloudWatch Insights exported CSVs to aggregate metrics by service name

---

## 🔧 Commands & Examples

### awk Basics

```bash
# Print specific fields (columns)
ps aux | awk '{print $1, $2, $11}'       # user, pid, command
df -h | awk '{print $1, $5, $6}'         # filesystem, use%, mount

# Field separator: default is whitespace, use -F for custom
cat /etc/passwd | awk -F: '{print $1, $3}'      # username, uid
cat /etc/passwd | awk -F: '{print $1, $7}'      # username, shell

# CSV processing
awk -F',' '{print $1, $3}' data.csv

# Tab-separated
awk -F'\t' '{print $1, $2}' data.tsv

# Multiple field separators (regex)
awk -F'[,;:]' '{print $1, $2}' data.txt

# Print the last field (regardless of how many there are)
echo "one two three four" | awk '{print $NF}'     # four
ls -la | awk '{print $NF}'                         # filenames

# Print second-to-last field
awk '{print $(NF-1)}' file.txt

# Print all fields from 3 onwards
awk '{for(i=3;i<=NF;i++) printf "%s ", $i; print ""}' file.txt

# Reformat output
ps aux | awk '{printf "PID: %-8s CMD: %s\n", $2, $11}'

# Print line numbers
awk '{print NR": "$0}' /etc/hosts
```

### Pattern Matching

```bash
# Print lines matching a regex pattern
awk '/ERROR/' /var/log/app.log
awk '/ERROR|WARN/' /var/log/app.log

# Print lines NOT matching a pattern
awk '!/^#/' /etc/hosts      # skip comment lines
awk '!/^$/' file.txt         # skip blank lines

# Numeric comparisons on fields
df -h | awk '$5 > 80 {print "WARN:", $0}'        # disk use% > 80

# String comparisons
ps aux | awk '$1 == "www-data" {print $2, $11}'   # only www-data processes

# Range pattern: print lines between START and STOP
awk '/BEGIN CERTIFICATE/,/END CERTIFICATE/' ssl.pem

# Process only specific line numbers
awk 'NR==5' file.txt          # only line 5
awk 'NR>=10 && NR<=20' file.txt    # lines 10-20

# Skip header line
awk 'NR>1 {print $1, $2}' file.csv

# Combine field filter with pattern
awk '$3 > 100 && /timeout/' /var/log/nginx/access.log
```

### BEGIN and END Blocks

```bash
# BEGIN: runs before any input is read
# END: runs after all input is processed

# Print a header
ps aux | awk 'BEGIN {print "USER\t\tPID\t\tCOMMAND"} NR>1 {print $1"\t\t"$2"\t\t"$11}'

# Count matching lines
awk '/ERROR/ {count++} END {print "Total errors:", count}' /var/log/app.log

# Sum a column
df -k | awk 'NR>1 {sum += $3} END {print "Total used KB:", sum}'

# Calculate average response time from access log
# Log format: ... "GET /api" 200 0.123
awk '{sum += $NF; count++} END {print "Avg response time:", sum/count, "s"}' /var/log/nginx/timing.log

# Find min and max values
awk 'NR==1 {min=max=$1} {if($1<min) min=$1; if($1>max) max=$1} END {print "min:", min, "max:", max}' values.txt

# Multiple accumulators
awk '/ERROR/ {errors++} /WARN/ {warns++} /INFO/ {infos++} END {
  print "Errors:", errors+0
  print "Warnings:", warns+0
  print "Info:", infos+0
}' /var/log/app.log
```

### Associative Arrays (the Power Feature)

```bash
# Count occurrences of a field value
# Count HTTP status codes from nginx access log
awk '{print $9}' /var/log/nginx/access.log | awk '{count[$1]++} END {for(code in count) print code, count[code]}' | sort

# Or in one awk command:
awk '{count[$9]++} END {for(code in count) print code, count[code]}' /var/log/nginx/access.log | sort -k2 -rn

# Top 10 IP addresses by request count
awk '{count[$1]++} END {for(ip in count) print count[ip], ip}' /var/log/nginx/access.log | sort -rn | head 10

# Sum bytes per HTTP status code
awk '{bytes[$9] += $10} END {for(code in bytes) print code, bytes[code]}' /var/log/nginx/access.log

# Traffic per endpoint (URL path)
awk '{count[$7]++} END {for(url in count) print count[url], url}' /var/log/nginx/access.log | sort -rn | head 20

# Count unique users per day from auth.log
awk '/Accepted/ {split($1, a, "T"); day=a[1]; users[day][$9]++}
END {for(d in users) {count=0; for(u in users[d]) count++; print d, count, "unique users"}}' /var/log/auth.log

# K8s: count pods per namespace
kubectl get pods --all-namespaces | awk 'NR>1 {count[$1]++} END {for(ns in count) print ns, count[ns]}' | sort -k2 -rn
```

### String Functions

```bash
# Length of a field
awk '{print length($0), $0}' file.txt

# Substring
awk '{print substr($1, 1, 3)}' file.txt    # first 3 chars of field 1

# Split a field into an array
awk '{n=split($1, a, "."); print a[1], a[n]}' /var/log/app.log   # first and last part

# Convert case
awk '{print toupper($1)}' file.txt
awk '{print tolower($0)}' file.txt

# String substitution (sub = first match, gsub = all matches)
awk '{gsub(/foo/, "bar"); print}' file.txt           # replace all "foo" with "bar"
awk '{sub(/^[0-9]+/, "[NUM]"); print}' file.txt      # replace leading number

# Remove trailing whitespace
awk '{gsub(/[[:space:]]+$/, ""); print}' file.txt

# Extract IP addresses from a log
awk 'match($0, /[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+/) {print substr($0, RSTART, RLENGTH)}' file.log

# Parse ISO8601 timestamps
awk '{split($2, t, ":"); hour=substr(t[1],length(t[1])-1,2); print hour, $0}' app.log
```

### Multi-File and Advanced Patterns

```bash
# Process multiple files, know which file you're in
awk '{print FILENAME": "$0}' *.log

# Join two files on a common field (like SQL JOIN)
# File 1: instance_id, private_ip
# File 2: instance_id, name_tag
awk 'NR==FNR {name[$1]=$2; next} {print $1, $2, name[$1]}' names.txt ips.txt

# Dedup lines while preserving order
awk '!seen[$0]++' file.txt

# Print unique values of field 1
awk '!seen[$1]++{print $1}' file.txt

# Print only duplicate lines
awk 'seen[$0]++ == 1' file.txt

# Compare two files: lines only in file1 (like diff but for fields)
awk 'NR==FNR {seen[$0]=1; next} !seen[$0]' file2.txt file1.txt
```

### Real-World Production Patterns

```bash
# Nginx access log: requests by status code per minute
awk '{
  split($4, t, ":");
  minute = t[1]":"t[2]":"t[3];
  count[minute][$9]++
}
END {
  for(m in count)
    for(code in count[m])
      print m, code, count[m][code]
}' /var/log/nginx/access.log | sort

# Find slowest API endpoints (nginx log with response time in last field)
awk '$NF > 1.0 {print $NF, $7}' /var/log/nginx/access.log | \
  sort -rn | head 20

# P95 response time approximation (sort and pick 95th percentile)
awk '{print $NF}' /var/log/nginx/timing.log | \
  sort -n | \
  awk 'BEGIN{c=0} {a[c++]=$0} END{print "P50:", a[int(c*0.50)], "P95:", a[int(c*0.95)], "P99:", a[int(c*0.99)]}'

# AWS: extract instance IDs and states from describe-instances JSON lines
aws ec2 describe-instances --output text | awk '/INSTANCES/ {print $8, $9}'

# K8s: find all pods not in Running state
kubectl get pods --all-namespaces | awk 'NR>1 && $4 != "Running" {print $1, $2, $4}'

# K8s: show pods with high restart counts
kubectl get pods --all-namespaces | awk 'NR>1 && $5 > 5 {print $1, $2, "restarts:", $5}'

# Parse journald output to count errors per service
journalctl --since "1 hour ago" --no-pager | \
  awk '/systemd/ && /failed/ {service=$NF; gsub(/[().]/, "", service); count[service]++}
  END {for(s in count) print count[s], s}' | sort -rn

# Process CloudWatch Insights exported CSV
awk -F',' 'NR>1 {
  gsub(/"/, "", $2);
  sum[$2] += $3;
  calls[$2]++
}
END {
  for(svc in sum)
    printf "%-30s avg_latency: %.2fms calls: %d\n", svc, sum[svc]/calls[svc], calls[svc]
}' cloudwatch-export.csv | sort -k3 -rn

# Extract failed SSH attempts with IP and count
awk '/Failed password/ {
  for(i=1;i<=NF;i++)
    if($i=="from") {ip=$(i+1); count[ip]++}
}
END {
  for(ip in count)
    if(count[ip] > 5) print count[ip], ip
}' /var/log/auth.log | sort -rn
```

### awk in Scripts (Multi-line Programs)

```bash
# For complex awk, use a script file
cat > /tmp/analyze.awk << 'EOF'
BEGIN {
    FS = " ";
    print "Starting analysis...";
    threshold = 500;
}

/ERROR/ {
    errors++;
    if ($NF > threshold) {
        slow_errors++;
        print "SLOW ERROR:", $0;
    }
}

/WARN/ {
    warns++;
}

END {
    print "---";
    print "Total errors:", errors+0;
    print "Slow errors (>" threshold "ms):", slow_errors+0;
    print "Warnings:", warns+0;
}
EOF

awk -f /tmp/analyze.awk /var/log/app.log

# Pass variables from shell to awk
THRESHOLD=1000
awk -v thresh="$THRESHOLD" '$NF > thresh {print "Slow:", $0}' /var/log/app.log

# Export shell array results to awk
IGNORE_IPS="10.0.0.1 10.0.0.2 10.0.0.3"
awk -v ignore="$IGNORE_IPS" 'BEGIN {n=split(ignore,a," "); for(i in a) skip[a[i]]=1}
  !skip[$1] {print}' /var/log/nginx/access.log
```

---

## ⚠️ Gotchas & Pro Tips

- **Initialise counters in END or they're empty strings:** If a counter variable is never incremented (no matching lines), it's an empty string in awk. Use `count+0` or `count+=""` to force numeric/string context. Or initialise in `BEGIN {count=0}`.

- **`print` vs `printf`:** `print` always adds a newline. `printf` gives you full formatting control like C's printf. Use `printf` when you need aligned columns or want to suppress newlines between fields.

- **`$0` changes when you change fields:** If you do `$2 = "newvalue"`, awk rebuilds `$0` using OFS as the separator. Set `OFS` in `BEGIN` to control how `$0` is reconstructed: `BEGIN {OFS="\t"}`.

- **awk arrays are not sorted:** When iterating `for(key in array)`, the order is implementation-defined (gawk uses hash order). If you need sorted output, pipe to `sort` or collect into an indexed array.

- **Use `gawk` for advanced features:** GNU awk (`gawk`) adds TCP/IP networking (`/inet/`), `gensub()` for non-destructive substitution, `patsplit()`, and `PROCINFO`. Most Linux systems have gawk. Check with `awk --version`.

- **Performance:** awk is extremely fast — it can process millions of lines per second. For large log files (10GB+), awk will outperform Python/Ruby equivalents by 5-10x. When processing is complex, consider `mawk` (optimised awk) which is even faster on simple field operations.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
