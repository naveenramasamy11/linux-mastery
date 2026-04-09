# 🐧 NIC Teaming & VLAN Trunking — Linux Mastery

> **NIC teaming and VLAN trunking are the backbone of enterprise-grade Linux networking — essential for bare-metal servers, on-prem VMware hosts, and physical Kubernetes nodes where redundancy and network segmentation are non-negotiable.**

## 📖 Concept

**NIC Teaming** (superseding the older `bonding` driver on RHEL 7+) is the logical aggregation of multiple physical NICs into a single virtual interface. The `teamd` daemon manages link monitoring and traffic distribution using pluggable **runner modules**:

- `activebackup` — one active link, others on standby; failover on link loss
- `loadbalance` — hash-based distribution across all active links
- `lacp` — 802.3ad Link Aggregation Control Protocol; requires switch support
- `roundrobin` — sequential packet distribution (rarely used in production)
- `broadcast` — sends all traffic on all ports (used for fault tolerance testing)
- `random` — random port selection

The team interface (`team0`) appears to the OS as a single NIC with one MAC and one IP. Kernel handles the link-layer aggregation via the `team` kernel module.

**VLAN Trunking** allows a single physical or team interface to carry traffic for multiple VLANs simultaneously. Linux implements 802.1Q VLAN tagging via the `8021q` kernel module. Each VLAN gets a virtual sub-interface (e.g., `eth0.100` for VLAN 100) that tags/untags frames at the kernel level. You can stack VLANs on top of team interfaces (`team0.100`), bonds, or regular NICs.

In on-prem K8s, VMware ESXi host networks, and colo servers, this pattern is ubiquitous: two physical NICs teamed for redundancy, with multiple VLAN sub-interfaces for management, storage, and application traffic — all on the same cable pair.

---

## 💡 Real-World Use Cases

- **Bare-metal K8s node:** Two 25GbE NICs teamed in LACP for 50Gbps bandwidth + redundancy, VLAN 100 for cluster traffic, VLAN 200 for storage (Ceph/iSCSI), VLAN 300 for BMC/IPMI management.
- **VMware ESXi management host:** Team NICs for vMotion HA, trunk VLANs for VM network segmentation without needing a physical switch per VLAN.
- **OpenShift node networking:** OVN-Kubernetes uses VLAN-backed networks; physical team + VLAN trunks give the SDN layer physical segments to work with.
- **Network isolation for compliance:** Separate VLAN for PCI-DSS workloads on the same physical server, enforced at the physical switch trunk port.
- **pfSense/VyOS on KVM:** Trunk VLANs into a VM hypervisor using a team interface, let the VM router do inter-VLAN routing.

---

## 🔧 Commands & Examples

### NIC Teaming with nmcli

```bash
# Step 1: Load team kernel module
modprobe team
lsmod | grep team

# Step 2: Create team master interface
nmcli con add type team \
    con-name team0 \
    ifname team0 \
    config '{"runner": {"name": "activebackup"}, "link_watch": {"name": "ethtool"}}'

# Assign IP to team0
nmcli con mod team0 \
    ipv4.addresses 192.168.1.10/24 \
    ipv4.gateway 192.168.1.1 \
    ipv4.method manual

# Step 3: Add physical NICs as team ports (slaves)
nmcli con add type team-slave \
    con-name team0-port1 \
    ifname eth0 \
    master team0

nmcli con add type team-slave \
    con-name team0-port2 \
    ifname eth1 \
    master team0

# Step 4: Bring up the team
nmcli con up team0
nmcli con up team0-port1
nmcli con up team0-port2

# Verify
teamdctl team0 state
# setup:
#   runner: activebackup
# ports:
#   eth0
#     link watches:
#       link summary: up
#       instance[link_watch_0]:
#         name: ethtool
#         link: up
#         down count: 0
#   eth1
#     link watches:
#       link summary: up
```

### LACP Teaming (Requires Switch Support)

```bash
# Create LACP team — switch port must be configured as LACP aggregate
nmcli con add type team \
    con-name team0-lacp \
    ifname team0 \
    config '{
        "runner": {
            "name": "lacp",
            "active": true,
            "fast_rate": true,
            "tx_hash": ["eth", "ipv4", "ipv6", "tcp", "udp"]
        },
        "link_watch": {"name": "ethtool"}
    }'

# Add ports
nmcli con add type team-slave con-name port1 ifname eth0 master team0
nmcli con add type team-slave con-name port2 ifname eth1 master team0
nmcli con add type team-slave con-name port3 ifname eth2 master team0
nmcli con add type team-slave con-name port4 ifname eth3 master team0

nmcli con up team0-lacp

# Check LACP status
teamdctl team0 state view
teamdctl team0 getoption activeport
```

### Monitoring and Failover Testing

```bash
# View team state
teamdctl team0 state
teamdctl team0 config dump

# Check port link status
teamdctl team0 ports

# Force failover: bring down active port, verify failover
ip link set eth0 down
teamdctl team0 state   # eth1 should now be active

ip link set eth0 up
teamdctl team0 state   # eth0 should rejoin (as standby in activebackup)

# Watch real-time state changes
watch -n1 teamdctl team0 state

# teamnl — low-level team netlink tool
teamnl team0 ports
teamnl team0 options
```

### VLAN Trunking with ip link

```bash
# Load 8021q module
modprobe 8021q
echo "8021q" >> /etc/modules-load.d/8021q.conf   # Persist across reboots

# Create VLAN interface on a physical NIC
ip link add link eth0 name eth0.100 type vlan id 100
ip link add link eth0 name eth0.200 type vlan id 200
ip link add link eth0 name eth0.300 type vlan id 300

# Assign IPs
ip addr add 10.100.0.10/24 dev eth0.100
ip addr add 10.200.0.10/24 dev eth0.200
ip addr add 10.300.0.10/24 dev eth0.300

# Bring up
ip link set eth0.100 up
ip link set eth0.200 up
ip link set eth0.300 up

# Verify VLAN interfaces
ip -d link show eth0.100
# 5: eth0.100@eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
#     link/ether 52:54:00:ab:cd:ef brd ff:ff:ff:ff:ff:ff
#     vlan protocol 802.1Q id 100 <REORDER_HDR>

# Show all VLANs
cat /proc/net/vlan/config
# VLAN Dev name     | VLAN ID
# Name-Type: VLAN_NAME_TYPE_RAW_PLUS_VID_NO_PAD
# eth0.100        | 100  | eth0
# eth0.200        | 200  | eth0
# eth0.300        | 300  | eth0
```

### VLAN on Top of Team Interface

```bash
# This is the real production pattern:
# Two NICs teamed for redundancy, VLANs on top for segmentation

# 1. Create team0 (see above steps)
# 2. Add VLANs on top of team0

ip link add link team0 name team0.100 type vlan id 100
ip link add link team0 name team0.200 type vlan id 200

ip addr add 10.100.0.10/24 dev team0.100   # Cluster network
ip addr add 10.200.0.10/24 dev team0.200   # Storage network

ip link set team0.100 up
ip link set team0.200 up

# Or use nmcli to persist
nmcli con add type vlan \
    con-name vlan100 \
    ifname team0.100 \
    dev team0 \
    id 100 \
    ipv4.addresses 10.100.0.10/24 \
    ipv4.method manual

nmcli con add type vlan \
    con-name vlan200 \
    ifname team0.200 \
    dev team0 \
    id 200 \
    ipv4.addresses 10.200.0.10/24 \
    ipv4.method manual

nmcli con up vlan100
nmcli con up vlan200
```

### Legacy ifcfg Files (RHEL 7 / Older Systems)

```bash
# /etc/sysconfig/network-scripts/ifcfg-team0
cat > /etc/sysconfig/network-scripts/ifcfg-team0 << 'EOF'
DEVICE=team0
DEVICETYPE=Team
TEAM_CONFIG='{"runner":{"name":"activebackup"},"link_watch":{"name":"ethtool"}}'
BOOTPROTO=none
ONBOOT=yes
EOF

# /etc/sysconfig/network-scripts/ifcfg-team0-port1
cat > /etc/sysconfig/network-scripts/ifcfg-team0-port1 << 'EOF'
DEVICE=eth0
DEVICETYPE=TeamPort
TEAM_MASTER=team0
ONBOOT=yes
BOOTPROTO=none
EOF

# VLAN on top of team
cat > /etc/sysconfig/network-scripts/ifcfg-team0.100 << 'EOF'
DEVICE=team0.100
VLAN=yes
VLAN_ID=100
PHYSDEV=team0
BOOTPROTO=none
ONBOOT=yes
IPADDR=10.100.0.10
PREFIX=24
EOF
```

### Verification and Debugging

```bash
# Check all network interfaces and their relationships
ip link show

# Show VLAN detail
ip -d link show eth0.100

# Verify VLAN tagging is working with tcpdump
tcpdump -i eth0 -e vlan   # Shows VLAN-tagged frames
tcpdump -i eth0.100       # Shows only VLAN 100 traffic (untagged at this level)

# Check bonding/team throughput
iperf3 -c 10.100.0.20 -t 30 -P 4   # 4 parallel streams

# Verify LACP negotiation (requires switch support)
cat /proc/net/bonding/bond0   # If using bonding instead of teaming

# Check team statistics
teamnl team0 options | grep -i stat

# ethtool to check physical link speed
ethtool eth0 | grep -E "Speed|Duplex|Link"
ethtool eth1 | grep -E "Speed|Duplex|Link"
```

---

## ⚠️ Gotchas & Pro Tips

- **LACP requires switch-side configuration.** If you configure LACP on Linux but the switch port is not an LACP aggregate, traffic will still flow but you'll lose load balancing and proper failover. Always coordinate with your network team.
- **`team` vs `bond` — don't mix them.** RHEL 7+ introduced `teamd` as the replacement for the kernel bonding driver. They are NOT the same. `teamd` runs in userspace and is more flexible. In RHEL 9 / Fedora, `team` is deprecated in favor of `bond` again (full circle), managed through NetworkManager.
- **VLAN MTU must be ≤ parent interface MTU.** If your parent NIC is 1500 MTU and you need jumbo frames on a VLAN (9000 MTU), you must set the parent to 9000+ first. `ip link set eth0 mtu 9100` before creating VLAN sub-interfaces.
- **NetworkManager manages team/VLAN by default on modern RHEL.** Don't mix manual `ip` commands with nmcli configs — NetworkManager will overwrite manual changes on restart. Use nmcli exclusively or disable NM for the interface.
- **Pro tip — verify switch trunk config matches:**

```bash
# Check what VLANs are currently being received on an interface
tcpdump -i eth0 -c 100 -e 2>/dev/null | grep -o '802.1Q.*vlan [0-9]*' | sort -u
```

- **`teamd` config is JSON and version-controlled.** Keep your team configs in `/etc/teamd/` and version them in Git. This is a production best practice — network config changes should go through proper change management.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
