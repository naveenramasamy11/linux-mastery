# 🐧 SELinux & AppArmor — Linux Mastery

> **Mandatory Access Control stops privilege escalation cold — a root-owned process in an SELinux-confined domain still can't read `/etc/shadow` or write outside its allowed paths, which is exactly why it's the last line of defense when everything else fails.**

## 📖 Concept

**Discretionary Access Control (DAC)** — the standard Unix permission model (owner/group/other + rwx) — is *discretionary* because the file owner can grant permissions to anyone. Once a process runs as root, DAC stops protecting you. **Mandatory Access Control (MAC)** adds a second layer where the *system policy* (not the user) decides what's allowed, regardless of file permissions or uid.

**SELinux** (Security-Enhanced Linux), developed by the NSA and now maintained by Red Hat, uses a **type enforcement** model. Every process runs in a *domain* (e.g., `httpd_t`), every file has a *type* (e.g., `httpd_sys_content_t`), and the policy defines which domain-type interactions are permitted. A compromised Apache process running as `httpd_t` cannot read `/etc/shadow` (labeled `shadow_t`) even as root — the SELinux policy has no rule allowing `httpd_t → shadow_t:file read`.

**AppArmor** (used on Ubuntu/Debian) takes a path-based approach rather than label-based. Profiles are attached to executables and define exactly which files they can access (by path glob), which capabilities they can use, and which network operations are allowed. AppArmor is generally easier to write profiles for, while SELinux provides stronger guarantees (labels persist even if files are moved).

In Kubernetes, both systems play a critical role: SELinux labels on pods, AppArmor annotations, and seccomp profiles together implement defense-in-depth for container workloads.

---

## 💡 Real-World Use Cases

- **Web server breakout prevention:** Even if Apache/Nginx is compromised via a 0-day, SELinux `httpd_t` domain prevents reading SSH keys, `/etc/shadow`, or writing to system directories.
- **Container escape defense:** Kubernetes pods running with `seLinuxOptions` or AppArmor profiles prevent container escape from reaching the host filesystem.
- **Compliance (PCI-DSS, HIPAA):** Both PCI-DSS and CIS benchmarks require MAC enforcement. SELinux enforcing mode is a mandatory check in most compliance audits.
- **Database server hardening:** PostgreSQL in `postgresql_t` domain cannot spawn shells or make outbound network connections — even if the DB is fully compromised.
- **Incident investigation:** SELinux AVC denials in `/var/log/audit/audit.log` are often the first indicator of an intrusion attempt or misconfigured application.

---

## 🔧 Commands & Examples

### SELinux Basics

```bash
# Check SELinux status
sestatus
# SELinux status:                 enabled
# SELinuxfs mount:                /sys/fs/selinux
# SELinux mount point:            /sys/fs/selinux
# Loaded policy name:             targeted
# Current mode:                   enforcing     ← enforcing = blocking, permissive = logging only
# Mode from config file:          enforcing
# Policy MLS status:              enabled
# Policy deny_unknown status:     denied
# Memory protection checking:     actual (secure)
# Max kernel policy version:      33

getenforce     # Quick mode check: Enforcing / Permissive / Disabled

# Change mode at runtime (no reboot)
setenforce 0   # Switch to Permissive (logs but doesn't block)
setenforce 1   # Switch back to Enforcing

# Permanent mode change (edit /etc/selinux/config)
sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
# Reboot required for this change

# NEVER set SELINUX=disabled — disabling SELinux requires relabeling
# the entire filesystem on re-enable. Use Permissive for debugging instead.
```

### SELinux Contexts (Labels)

```bash
# Show security context of files (-Z flag)
ls -laZ /var/www/html/
# -rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 index.html
# Format: user:role:type:level

# Show context of processes
ps auxZ | grep httpd
# system_u:system_r:httpd_t:s0  apache    1234  httpd

# Show your current context
id -Z
# unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023

# Show context of a directory
ls -dZ /etc/
# system_u:object_r:etc_t:s0 /etc/

# File with WRONG context (common problem after copying files)
ls -Z /var/www/html/newfile.php
# -rw-r--r--. root root unconfined_u:object_r:user_home_t:s0 newfile.php
# ← user_home_t in /var/www/html = httpd can't serve it!
```

### Fixing SELinux Contexts

```bash
# Fix context based on filesystem policy (restorecon)
restorecon -v /var/www/html/newfile.php
# Relabeled /var/www/html/newfile.php from user_home_t to httpd_sys_content_t

# Recursively fix a directory
restorecon -Rv /var/www/html/

# Manually set context
chcon -t httpd_sys_content_t /var/www/html/custom.html
chcon -u system_u -r object_r -t httpd_sys_content_t /var/www/html/custom.html

# Set context to match another file
chcon --reference=/var/www/html/index.html /var/www/html/newfile.php

# PERSISTENT context assignment (survives restorecon)
semanage fcontext -a -t httpd_sys_content_t "/opt/myapp/public(/.*)?"
restorecon -Rv /opt/myapp/public/

# List custom fcontext rules
semanage fcontext -l | grep myapp
```

### Analyzing AVC Denials

```bash
# View SELinux denials (Access Vector Cache denials)
ausearch -m AVC -ts recent
# time->Mon Apr  7 10:23:15 2025
# type=AVC msg=audit(1712477995.123:456): avc: denied { read } for
#   pid=1234 comm="httpd" name="secret.conf"
#   scontext=system_u:system_r:httpd_t:s0
#   tcontext=system_u:object_r:admin_home_t:s0
#   tclass=file permissive=0

# Filter denials by process
ausearch -m AVC -c httpd

# Interpret denials with audit2why
ausearch -m AVC -ts recent | audit2why
# Was caused by: Missing type enforcement (TE) allow rule
# allow httpd_t admin_home_t:file read;

# Generate policy module from denial
ausearch -m AVC -ts recent | audit2allow -M myhttpd
# Creates myhttpd.te (human-readable) and myhttpd.pp (compiled module)

# Review the generated policy
cat myhttpd.te

# Install the module (if you trust it)
semodule -i myhttpd.pp

# List installed modules
semodule -l | head -20

# Remove a module
semodule -r myhttpd
```

### SELinux Booleans

```bash
# List all booleans
semanage boolean -l | head -30

# Common booleans for Apache
getsebool httpd_can_network_connect        # Allow httpd to make outbound connections
getsebool httpd_can_network_connect_db     # Allow httpd to connect to databases
getsebool httpd_enable_homedirs            # Allow serving from ~user/public_html
getsebool httpd_use_nfs                    # Allow httpd to use NFS mounts

# Enable a boolean (temporary)
setsebool httpd_can_network_connect on

# Enable a boolean (persistent across reboots)
setsebool -P httpd_can_network_connect on

# See all httpd-related booleans
semanage boolean -l | grep httpd

# Disable a boolean
setsebool -P httpd_can_network_connect off
```

### AppArmor Basics (Ubuntu/Debian)

```bash
# Check AppArmor status
aa-status
# apparmor module is loaded.
# 42 profiles are loaded.
# 35 profiles are in enforce mode.
# 7 profiles are in complain mode.
# 0 processes are unconfined but have a profile defined.

# List profiles and their modes
apparmor_status

# Change profile mode
aa-enforce /etc/apparmor.d/usr.sbin.nginx    # Switch to enforce
aa-complain /etc/apparmor.d/usr.sbin.nginx   # Switch to complain (log only)
aa-disable /etc/apparmor.d/usr.sbin.nginx    # Disable

# Reload a profile after editing
apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx

# View AppArmor denials
dmesg | grep "apparmor=DENIED"
journalctl | grep "apparmor"
```

### Writing an AppArmor Profile

```bash
# Generate a profile skeleton for a binary
aa-genprof /usr/local/bin/myapp
# This runs the app in learning mode and prompts you to approve observed behavior

# Example profile: /etc/apparmor.d/usr.local.bin.myapp
cat > /etc/apparmor.d/usr.local.bin.myapp << 'EOF'
#include <tunables/global>

/usr/local/bin/myapp {
    #include <abstractions/base>
    #include <abstractions/nameservice>

    # Allow reading config
    /etc/myapp/** r,
    /etc/myapp/secrets.conf r,

    # Allow writing to data directory
    /var/lib/myapp/ rw,
    /var/lib/myapp/** rw,

    # Allow writing logs
    /var/log/myapp/ rw,
    /var/log/myapp/*.log rw,

    # Allow network (TCP only, specific ports)
    network tcp,

    # Deny everything else explicitly (AppArmor denies by default anyway)
    deny /etc/shadow r,
    deny /root/** rw,

    # Allow execution of specific binaries
    /usr/bin/python3 rix,

    # Capabilities
    capability net_bind_service,  # Bind to ports < 1024
}
EOF

# Load the profile
apparmor_parser -r /etc/apparmor.d/usr.local.bin.myapp
aa-status | grep myapp
```

### SELinux for Kubernetes / Containers

```bash
# Set SELinux label on a Kubernetes pod
cat << 'EOF' > pod-selinux.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    seLinuxOptions:
      level: "s0:c123,c456"    # MCS categories for multi-tenancy
  containers:
  - name: app
    image: nginx
    securityContext:
      seLinuxOptions:
        type: "container_t"    # Container-specific SELinux domain
EOF

kubectl apply -f pod-selinux.yaml

# AppArmor annotation for Kubernetes pods
cat << 'EOF' > pod-apparmor.yaml
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/mycontainer: localhost/my-profile
spec:
  containers:
  - name: mycontainer
    image: nginx
EOF

# Check if container is confined
docker inspect <container_id> | grep -i "apparmor\|selinux"

# Verify SELinux context inside container
docker exec <container_id> cat /proc/self/attr/current
# system_u:system_r:container_t:s0:c123,c456
```

---

## ⚠️ Gotchas & Pro Tips

- **Never `setenforce 0` in production without a ticket.** It's tempting to disable SELinux when troubleshooting, but it masks real security misconfigurations. Always use `permissive` mode (which still logs) or fix the context/policy properly.
- **`chcon` changes are lost on `restorecon` or filesystem relabel.** Always use `semanage fcontext` + `restorecon` for permanent changes. `chcon` is only for temporary testing.
- **`audit2allow` can create dangerous policies.** Always review generated `.te` files — `audit2allow` blindly allows the denied operation, which may be overly permissive. Never blindly install generated modules in production.
- **AppArmor profiles are path-based — moves break them.** If a binary is moved/renamed, its profile no longer applies. SELinux labels follow the file (inode), not the path.
- **`dontaudit` rules hide real denials.** SELinux has `dontaudit` rules that suppress logging of certain common denials to reduce noise. Run `semodule -DB` to disable dontaudit rules and see ALL denials during debugging. Re-enable with `semodule -B`.

```bash
# Pro tip: seinfo for policy introspection
seinfo -t httpd_t -x   # What can httpd_t domain do?
seinfo --role          # All roles in policy
sesearch --allow -s httpd_t -t shadow_t   # Can httpd_t access shadow_t?
# (should return nothing — no allow rule means blocked)
```

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
