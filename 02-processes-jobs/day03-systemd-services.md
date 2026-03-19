# 🔧 Day 03 — Systemd Mastery: Services, Timers, Journals & Unit Files

> Systemd runs on every modern Linux distro. If you manage EC2 instances, containers, or bare metal — understanding it deeply will save you hours of debugging and unlock powerful automation you didn't know existed.

---

## Unit File Anatomy

Unit files live in these locations (highest priority first):

```bash
/etc/systemd/system/       # Admin overrides — always wins
/run/systemd/system/       # Runtime (ephemeral, lost on reboot)
/usr/lib/systemd/system/   # Package-installed — don't edit directly
```

```bash
systemctl show nginx --property=FragmentPath   # Find where nginx's unit lives
systemctl cat nginx                             # Print effective unit (with overrides)
```

A typical service unit has three sections:

```ini
[Unit]
Description=My Web Application
After=network-online.target postgresql.service   # Start ordering
Wants=network-online.target                      # Soft dependency
Requires=postgresql.service                      # Hard dependency (fail if postgres fails)

[Service]
Type=notify                     # simple | forking | oneshot | notify | idle
User=myapp
WorkingDirectory=/opt/myapp
EnvironmentFile=/etc/myapp/env  # Load secrets from file, not unit
ExecStart=/opt/myapp/bin/server --config /etc/myapp/config.yaml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=10s
StartLimitBurst=3               # Only 3 restarts...
StartLimitIntervalSec=60s       # ...within 60 seconds

[Install]
WantedBy=multi-user.target
```

**Type= gotcha — pick the right one:**
- `Type=simple` — process stays in foreground; systemd considers it "started" immediately
- `Type=forking` — old-style daemon that double-forks; **must** also set `PIDFile=`
- `Type=notify` — service calls `sd_notify()` when ready; best for production daemons
- `Type=oneshot` — run once and exit; perfect for scripts triggered by timers

---

## Essential systemctl Commands

```bash
# Lifecycle
systemctl start|stop|restart nginx
systemctl reload nginx              # Send SIGHUP — reload config without restart
systemctl reload-or-restart nginx   # Reload if supported, restart if not

# Enable at boot
systemctl enable --now nginx        # Enable AND start in one command
systemctl disable --now nginx       # Disable AND stop in one command

# Status checks (great for scripting)
systemctl is-active --quiet nginx && echo "up" || echo "down"
systemctl is-enabled nginx          # Returns enabled/disabled/static/masked
systemctl is-failed nginx           # Returns 0 if in failed state

# List and inspect
systemctl list-units --state=failed         # All failed units — first thing to check
systemctl list-units --type=service         # All services
systemctl list-unit-files --state=enabled   # What starts at boot
systemctl list-dependencies nginx           # What nginx depends on
systemctl list-dependencies --reverse nginx # What depends on nginx

# Boot performance
systemd-analyze blame                       # Which units slowed boot
systemd-analyze critical-chain              # Critical path through boot sequence
systemd-analyze security myapp.service      # Security score (0=unsafe, 10=hardened)
```

**After editing any unit file manually:**
```bash
systemctl daemon-reload    # Always required — reloads unit definitions into memory
```

---

## Production-Grade Service Units

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Python Web Application
After=network-online.target postgresql.service
Wants=network-online.target
Requires=postgresql.service

[Service]
Type=notify
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
EnvironmentFile=/etc/myapp/env       # chmod 640, owned root:myapp
ExecStart=/opt/myapp/venv/bin/gunicorn \
    --workers 4 \
    --bind 0.0.0.0:8000 \
    --timeout 120 \
    myapp.wsgi:application
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=10s
StartLimitBurst=3
StartLimitIntervalSec=60s
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp               # Tag all logs as "myapp" in journald

# Hardening
PrivateTmp=true
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/var/lib/myapp /var/log/myapp

[Install]
WantedBy=multi-user.target
```

---

## Drop-in Overrides

**Never edit vendor unit files directly** — your changes get wiped on package upgrade. Use drop-ins:

```bash
systemctl edit nginx
# Creates /etc/systemd/system/nginx.service.d/override.conf
# Only override what you need — everything else inherits from the original
```

```ini
# /etc/systemd/system/nginx.service.d/override.conf
[Service]
LimitNOFILE=65535     # Increase file descriptor limit
Restart=always        # More aggressive restart policy
```

```bash
systemctl daemon-reload && systemctl restart nginx
systemctl cat nginx    # Verify the merged result
```

---

## Systemd Timers — Cron's Smarter Replacement

Timers get logging, dependency management, catch-up after downtime, and jitter — none of which cron offers.

Every timer needs a matching `.service` unit:

```ini
# /etc/systemd/system/db-backup.service
[Unit]
Description=Daily Database Backup

[Service]
Type=oneshot
User=backup
ExecStart=/usr/local/bin/backup-db.sh
```

```ini
# /etc/systemd/system/db-backup.timer
[Unit]
Description=Trigger daily DB backup at 2:30 AM

[Timer]
OnCalendar=*-*-* 02:30:00       # Every day at 02:30
RandomizedDelaySec=300          # Jitter up to 5 min (avoids thundering herd)
Persistent=true                 # Run missed job after downtime — cron can't do this!

[Install]
WantedBy=timers.target
```

```bash
systemctl enable --now db-backup.timer    # Enable and start the timer
systemctl list-timers                     # All timers + next run time

# Validate your calendar expression before deploying
systemd-analyze calendar "Mon..Fri 09:00:00"
```

**OnCalendar examples:**
```bash
OnCalendar=hourly                      # Every hour
OnCalendar=Mon..Fri 09:00:00           # Weekdays at 9 AM
OnCalendar=*-*-* 02:00:00/6:00:00     # Every 6 hours starting at 2 AM
```

**Monotonic timers (relative to boot or last run):**
```ini
OnBootSec=5min          # 5 minutes after boot
OnUnitActiveSec=1h      # 1 hour after last successful run (rolling)
```

---

## journalctl — Log Mastery

```bash
# Service logs
journalctl -u nginx                       # All logs for nginx
journalctl -u nginx -f                    # Follow in real time
journalctl -u nginx -n 50                 # Last 50 lines
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx --since "2026-03-19 08:00" --until "2026-03-19 09:00"

# Multiple services interleaved
journalctl -u nginx -u php-fpm -f

# Filter by severity (0=emerg → 7=debug)
journalctl -u myapp -p err               # Errors and above only
journalctl -u myapp -p warning..err      # Warnings through errors

# Boot-specific (gold for crash analysis)
journalctl -b                            # Current boot
journalctl -b -1                         # Previous boot
journalctl --list-boots                  # All recorded boots

# Output formats for parsing/pipelines
journalctl -u myapp -o json-pretty       # JSON
journalctl -u myapp -o cat               # Plain messages only

# Disk management
journalctl --disk-usage
journalctl --vacuum-size=500M            # Trim to 500MB
journalctl --vacuum-time=7d             # Delete logs older than 7 days
```

**Make journals persist across reboots (often not default):**
```bash
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald
```

---

## Hardening Services

```ini
[Service]
PrivateTmp=true                   # Isolated /tmp — prevents temp file hijacking
ProtectSystem=strict              # Read-only /usr, /etc, /boot
ProtectHome=true                  # No access to /home, /root
ReadWritePaths=/var/lib/myapp     # Explicit write allowlist
NoNewPrivileges=true              # Cannot escalate via setuid
CapabilityBoundingSet=            # Drop ALL capabilities
AmbientCapabilities=CAP_NET_BIND_SERVICE  # Add back only what's needed
ProtectKernelTunables=true        # Read-only /proc/sys, /sys
ProtectKernelModules=true         # Can't load kernel modules
SystemCallFilter=@system-service  # Only allow syscalls typical services need
SystemCallErrorNumber=EPERM       # Return EPERM (not kill) on violation
```

```bash
# Audit all running services' security scores at once
systemd-analyze security --no-pager | sort -k 4
```

---

## Resource Control via cgroups

```ini
[Service]
CPUWeight=100       # Relative CPU weight (1–10000)
CPUQuota=50%        # Hard cap: max 50% of one CPU core
MemoryHigh=512M     # Soft limit: start throttling
MemoryMax=1G        # Hard limit: OOM kill above this
MemorySwapMax=0     # Disable swap (good for latency-sensitive services)
IOReadBandwidthMax=/dev/nvme0n1 100M
IOWriteBandwidthMax=/dev/nvme0n1 50M
```

```bash
systemd-cgtop       # Live per-service CPU/memory/IO (like top for cgroups)
```

---

## Debugging Failed Services

```bash
# Step 1: Get the summary
systemctl status myapp.service           # Exit code + last log snippet

# Step 2: Read full logs for this boot
journalctl -u myapp.service -b           # Everything since last reboot
journalctl -u myapp.service -p err -b    # Errors only

# Step 3: Check if dependencies are healthy
systemctl list-dependencies myapp.service

# Step 4: Run ExecStart manually as the service user — reveals most issues
sudo -u myapp /opt/myapp/venv/bin/gunicorn myapp.wsgi:application

# Step 5: Test in systemd context (same env vars, cgroups, user)
systemd-run --uid=myapp --gid=myapp /opt/myapp/venv/bin/python -c "import myapp; print('OK')"

# Step 6: Reset failed state and retry
systemctl reset-failed myapp
systemctl start myapp
```

---

## Real-World AWS Patterns

**Packer AMI — ensure service is enabled before baking:**
```bash
systemctl enable myapp
systemctl is-enabled myapp || exit 1    # Fail the Packer build if not enabled
```

**EC2 UserData — install and start a service:**
```bash
#!/bin/bash
cat > /etc/systemd/system/myapp.service <<'EOF'
[Unit]
Description=My App
After=network-online.target
[Service]
Type=simple
ExecStart=/usr/local/bin/myapp
Restart=always
User=ec2-user
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now myapp
```

**Ansible + systemd module (for Packer provisioners):**
```yaml
- name: Enable and start myapp
  ansible.builtin.systemd:
    name: myapp
    state: started
    enabled: yes
    daemon_reload: yes
```

---

## Pro Tips and Gotchas

**`Requires=` doesn't imply `After=` — you need both for hard deps:**
```ini
Requires=postgresql.service   # Fail if postgres fails
After=postgresql.service      # Start AFTER postgres is ready
```

**Use `systemd-run` for one-off tasks with proper logging and isolation:**
```bash
systemd-run --property=MemoryMax=512M /usr/local/bin/data-import.sh
# Shows in journalctl, gets its own cgroup, cleans up after itself
```

**Check what environment a service actually sees:**
```bash
systemctl show myapp --property=Environment
# Or get all effective properties:
systemctl show myapp | grep -E "^(Exec|User|Group|Environment|Working)"
```

**Masking vs disabling:**
```bash
systemctl disable nginx    # Won't start at boot, but can be started manually
systemctl mask nginx       # Completely blocked — even `systemctl start nginx` fails
systemctl unmask nginx     # Remove mask
```

---

## Next:

→ **[Day 04 — Filesystem Tricks: find, xargs, Inodes & Bind Mounts](../03-filesystem-tricks/day02-find-mastery.md)**

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series — hands-on Linux for DevOps and SRE engineers.*
