# 🐧 Cron, at & systemd Timers — Linux Mastery

> **Scheduling is the backbone of automation — master it and you own every maintenance window, backup, and pipeline trigger in your infrastructure.**

## 📖 Concept

Task scheduling on Linux has evolved through three generations: **cron** (the classic, still ubiquitous), **at** (one-shot future execution), and **systemd timers** (the modern, monitorable replacement). As a DevOps engineer or SRE, you'll interact with all three — legacy cron jobs inherited from ops teams, `at` for one-time incident-response tasks, and systemd timers for new services where you want full journald integration.

**cron** is a daemon that reads *crontabs* — tables of schedule entries — and runs commands at the specified times. The system crontab lives at `/etc/crontab` and `/etc/cron.d/`, while per-user crontabs are stored in `/var/spool/cron/crontabs/`. The scheduling syntax follows a five-field format: minute, hour, day-of-month, month, day-of-week. Every sysadmin has fat-fingered this at 2 AM on a change-freeze weekend; understanding the format cold is non-negotiable.

**at** is perfect for *incident response* scenarios: "run this cleanup script in 30 minutes if I haven't cancelled it" or "restart the service at 23:00 after the deploy completes." It's also invaluable for scheduling tasks from within automated pipelines without spinning up a persistent daemon.

**systemd timers** supersede cron for new workloads. They integrate with `journalctl` for logging, support calendar expressions (far more powerful than five-field cron syntax), can be triggered by system events (boot, network-online.target), and support monotonic timers ("10 minutes after the unit last ran"). On AWS EC2 instances, ECS containers, and Kubernetes nodes running systemd-based OSes, timers are the right choice.

Understanding when cron output goes (or doesn't go) is critical. By default, cron emails output to the crontab owner — on many cloud instances this silently drops output. Always redirect stdout/stderr explicitly and ship logs to a central aggregator.

---

## 💡 Real-World Use Cases

- **Automated backups**: nightly `pg_dump` or S3 sync via cron or systemd timer, with output piped to a log file and alerting on non-zero exit codes
- **Certificate renewal**: `certbot renew` via systemd timer (the certbot package ships its own timer on modern distros)
- **AWS credential rotation**: scheduled Lambda invocation or EC2 instance profile refresh via a cron-triggered script
- **Log rotation triggers**: custom log archival jobs on containers that don't use logrotate
- **Health check remediation**: systemd timer that runs a self-healing script every 5 minutes to restart unhealthy services
- **One-time maintenance**: `at` to schedule a service restart after a deploy, cancellable if smoke tests fail

---

## 🔧 Commands & Examples

### Cron Syntax Reference

```bash
# ┌───────────── minute        (0–59)
# │ ┌───────────── hour         (0–23)
# │ │ ┌───────────── day of month (1–31)
# │ │ │ ┌───────────── month       (1–12 or jan–dec)
# │ │ │ │ ┌───────────── day of week  (0–7, 0 and 7 = Sunday)
# │ │ │ │ │
# * * * * * command

# Run every minute
* * * * * /usr/local/bin/heartbeat.sh

# Run at 02:30 every day
30 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1

# Run every 15 minutes
*/15 * * * * /opt/scripts/check_health.sh

# Run at 08:00 on weekdays only (Mon-Fri)
0 8 * * 1-5 /opt/scripts/morning_report.sh

# Run on the 1st and 15th of every month
0 0 1,15 * * /opt/scripts/monthly_cleanup.sh

# Run every hour from 09:00 to 17:00 on weekdays
0 9-17 * * 1-5 /opt/scripts/business_hours_check.sh

# Special strings (supported by most cron implementations)
@reboot  /opt/scripts/startup.sh          # run once at boot
@hourly  /opt/scripts/hourly.sh           # 0 * * * *
@daily   /opt/scripts/daily.sh            # 0 0 * * *
@weekly  /opt/scripts/weekly.sh           # 0 0 * * 0
@monthly /opt/scripts/monthly.sh          # 0 0 1 * *
@yearly  /opt/scripts/yearly.sh           # 0 0 1 1 *
```

### Managing User Crontabs

```bash
# Edit your own crontab (opens $EDITOR)
crontab -e

# List your current crontab
crontab -l

# Remove your crontab entirely (careful!)
crontab -r

# Edit or list another user's crontab (as root)
crontab -u naveen -e
crontab -u naveen -l

# Install a crontab from a file
crontab /tmp/my-crontab.txt

# Crontab with explicit PATH (always set this in crontabs!)
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=""
30 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

### System-Wide Cron Configuration

```bash
# /etc/crontab — system crontab with username field
# minute  hour  dom  month  dow  user  command
  30      2     *    *      *    root  /opt/scripts/backup.sh

# Drop-in scripts in these dirs run at the labeled interval
ls /etc/cron.hourly/
ls /etc/cron.daily/
ls /etc/cron.weekly/
ls /etc/cron.monthly/

# Custom schedules go in /etc/cron.d/
cat > /etc/cron.d/myapp-cleanup << 'EOF'
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=""
0 3 * * * app-user /opt/myapp/bin/cleanup.sh >> /var/log/myapp/cleanup.log 2>&1
EOF

chmod 644 /etc/cron.d/myapp-cleanup

# View cron daemon logs
grep cron /var/log/syslog | tail -50
journalctl -u cron -f
```

### at — One-Shot Scheduling

```bash
# Install at if not present
apt-get install at -y    # Debian/Ubuntu
dnf install at -y        # RHEL/Amazon Linux

# Enable and start the atd daemon
systemctl enable --now atd

# Schedule a command in 30 minutes
echo "/opt/scripts/post-deploy-restart.sh" | at now + 30 minutes

# Schedule at a specific time
echo "systemctl restart nginx" | at 23:00

# Schedule for a specific date and time
echo "/opt/scripts/maintenance.sh" | at 02:00 2026-06-01

# Interactive at input (Ctrl+D to save)
at now + 1 hour
> /opt/scripts/cleanup.sh >> /var/log/cleanup.log 2>&1
> Ctrl+D

# List pending at jobs
atq
# Output: jobnumber  date  time  queue  user
#         5          Sun May  3 23:00:00 2026  a  root

# View the contents of a queued job
at -c 5

# Remove a queued job
atrm 5

# Run a job at low priority (queue 'b' = batch, run when load < 1.5)
echo "/opt/scripts/heavy-report.sh" | at -q b now + 5 minutes
```

### systemd Timer Units

```bash
# A systemd timer requires TWO unit files:
# 1. a .timer unit (the schedule)
# 2. a .service unit (what to run)

# --- /etc/systemd/system/myapp-backup.service ---
cat > /etc/systemd/system/myapp-backup.service << 'EOF'
[Unit]
Description=MyApp Database Backup
After=network.target

[Service]
Type=oneshot
User=app-user
ExecStart=/opt/myapp/scripts/backup.sh
StandardOutput=journal
StandardError=journal
EOF

# --- /etc/systemd/system/myapp-backup.timer ---
cat > /etc/systemd/system/myapp-backup.timer << 'EOF'
[Unit]
Description=Run MyApp Backup Daily at 02:30

[Timer]
# Calendar expression (man systemd.time for full syntax)
OnCalendar=*-*-* 02:30:00

# Run immediately if the timer was missed (e.g., system was down)
Persistent=true

# Add up to 60 seconds of random delay to prevent thundering herd
RandomizedDelaySec=60

[Install]
WantedBy=timers.target
EOF

# Enable and start the timer (not the service)
systemctl daemon-reload
systemctl enable --now myapp-backup.timer

# Check timer status
systemctl status myapp-backup.timer
systemctl list-timers --all

# Run the service manually (useful for testing)
systemctl start myapp-backup.service

# View logs for the service
journalctl -u myapp-backup.service -n 50
journalctl -u myapp-backup.service --since "1 hour ago"
```

### systemd Timer Calendar Expressions

```bash
# Format: DayOfWeek Year-Month-Day Hour:Minute:Second

# Every day at 02:30
OnCalendar=*-*-* 02:30:00

# Every 15 minutes
OnCalendar=*:0/15

# Weekdays at 08:00
OnCalendar=Mon..Fri 08:00:00

# First day of every month at midnight
OnCalendar=*-*-01 00:00:00

# Every Sunday at 03:00
OnCalendar=Sun 03:00:00

# Hourly
OnCalendar=hourly

# Daily (equivalent to *-*-* 00:00:00)
OnCalendar=daily

# Monotonic timers (relative to system/unit events)
OnBootSec=15min         # 15 minutes after boot
OnUnitActiveSec=1h      # 1 hour after last activation
OnUnitInactiveSec=30min # 30 minutes after last deactivation

# Validate your calendar expression before deploying
systemd-analyze calendar "*-*-* 02:30:00"
systemd-analyze calendar "Mon..Fri 08:00:00"
# Output shows: next elapse time, iterations
```

### Monitoring & Debugging Scheduled Jobs

```bash
# List all active and inactive timers with next/last run times
systemctl list-timers --all
# NEXT                         LEFT     LAST                         PASSED    UNIT
# Sun 2026-05-03 02:30:00 UTC  11h left Sat 2026-05-02 02:30:05 UTC 12h ago  myapp-backup.timer

# Check if cron ran your job (look for CMD entries)
grep "CRON\|cron" /var/log/syslog | grep "backup" | tail -20

# Test a cron command in the same environment cron uses
# (Cron runs with a minimal environment — many scripts fail because of missing PATH)
env -i HOME=/root LOGNAME=root PATH=/usr/bin:/bin /opt/scripts/backup.sh

# Check crontab syntax without editing
echo "30 25 * * * /bin/echo test" | crontab -  # invalid: minute 25 > 23 ... wait, hour 25
# Use crontab -l to verify what's installed

# Run a cron job immediately for testing
run-parts /etc/cron.daily/
run-parts --test /etc/cron.daily/   # dry run, shows what would execute

# Verify at daemon is running
systemctl status atd

# Debug a failed systemd timer service
journalctl -u myapp-backup.service -b --no-pager
systemctl show myapp-backup.service | grep -E "ExecStart|Environment|User"
```

### Cron Best Practices for Production

```bash
# Always capture output — silent failures are dangerous
* * * * * /opt/scripts/check.sh >> /var/log/check.log 2>&1

# Use flock to prevent overlapping executions
* * * * * /usr/bin/flock -n /tmp/check.lock /opt/scripts/check.sh >> /var/log/check.log 2>&1

# Use timeout to prevent runaway jobs
*/5 * * * * /usr/bin/timeout 240 /opt/scripts/long_task.sh >> /var/log/long_task.log 2>&1

# Email on failure only (using a wrapper)
#!/bin/bash
# /opt/scripts/cron-wrapper.sh
OUTPUT=$("$@" 2>&1)
EXIT_CODE=$?
if [ $EXIT_CODE -ne 0 ]; then
    echo "$OUTPUT" | mail -s "CRON FAILED: $*" ops@yourcompany.com
fi
exit $EXIT_CODE

# Use it in crontab:
0 2 * * * /opt/scripts/cron-wrapper.sh /opt/scripts/backup.sh

# On AWS: send cron failures to CloudWatch
0 2 * * * /opt/scripts/backup.sh 2>&1 | /opt/scripts/log-to-cloudwatch.sh "backup-cron"
```

---

## ⚠️ Gotchas & Pro Tips

- **Missing PATH in cron environment**: Cron runs with a bare `PATH=/usr/bin:/bin`. Always set `PATH` at the top of your crontab or use absolute paths for all commands. Scripts that work fine from your shell fail silently in cron because `/usr/local/bin` isn't in the path.

- **Silent failures with MAILTO=""**: Setting `MAILTO=""` suppresses email but also hides errors. Always redirect to a log file (`>> /var/log/myjob.log 2>&1`) and monitor that log file or ship it to CloudWatch Logs / a SIEM.

- **cron doesn't source .bashrc/.profile**: User environment variables, conda environments, rbenv, nvm, etc. are not available. Source them explicitly inside your cron scripts: `source /home/user/.bashrc` or use the full path to the interpreter.

- **systemd timer vs. cron for containers**: Inside Docker containers, systemd isn't typically running. Use cron (via `crond` in your container) or an external scheduler (Kubernetes CronJob, AWS EventBridge Scheduler). On ECS, prefer EventBridge Scheduled Tasks.

- **Persistent=true in systemd timers**: Without this, if your system is down during the scheduled time, the job is skipped entirely. Set `Persistent=true` so systemd catches up and runs the job at next boot. Critical for backup timers.

- **RandomizedDelaySec prevents thundering herd**: When you have hundreds of EC2 instances all running the same timer, they'll all fire at exactly the same time without this setting, hammering your S3/RDS/API endpoint. Add `RandomizedDelaySec=300` to spread load over 5 minutes.

- **at jobs survive reboots? No.** Queued `at` jobs are stored in `/var/spool/at/` and survive reboots of the atd daemon, but not of the host system on most distros. Don't use `at` for anything that must survive a reboot — use a systemd one-shot service with `After=network-online.target` instead.

- **Cron and DST**: Cron uses local time. Jobs scheduled at `02:30` during a Daylight Saving Time spring-forward may not run (the clock jumps from 02:00 to 03:00). Use UTC in crontabs on servers: set `CRON_TZ=UTC` or run the system in UTC (which you should be doing anyway on servers).

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
