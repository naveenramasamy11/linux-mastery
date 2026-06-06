# 🐧 DNS Internals & dig Mastery — Linux Mastery

> **Understand how DNS actually works and master dig/drill to diagnose any resolution problem in 60 seconds.**

## 📖 Concept

DNS is the backbone of every networked system — and DNS failures are responsible for a disproportionate share of production outages. The "it's always DNS" joke exists for a reason. Yet most engineers only know enough to check if `ping google.com` works. Deep DNS knowledge separates engineers who spend 10 minutes diagnosing an outage from those who spend 3 hours.

**The DNS resolution chain:** When your application does `getaddrinfo("api.example.com")`, the OS follows this path: (1) check `/etc/hosts`, (2) check the local resolver cache (nscd, systemd-resolved), (3) query the configured nameserver in `/etc/resolv.conf`, (4) that nameserver performs recursive resolution — asking root nameservers → TLD nameservers → authoritative nameservers. The full chain touches 3-12 servers before returning an answer.

**Record types you must know:**
- **A / AAAA** — IPv4/IPv6 address records
- **CNAME** — alias to another name (cannot coexist with other records at the zone apex)
- **MX** — mail server with priority
- **NS** — authoritative nameserver for a zone
- **SOA** — Start of Authority — zone metadata including TTL, serial, retry intervals
- **TXT** — arbitrary text — used for SPF, DKIM, DMARC, domain verification
- **SRV** — service location (used by etcd, Kubernetes, VoIP)
- **PTR** — reverse DNS (IP → hostname)

**In Linux environments:** `systemd-resolved` (modern systems) maintains a per-link resolver cache and stub resolver at `127.0.0.53`. CoreDNS is the cluster DNS in Kubernetes. Route 53 is AWS's managed DNS. Understanding the resolution path end-to-end is essential for debugging microservices, K8s DNS failures, and cross-account AWS resource resolution.

---

## 💡 Real-World Use Cases

- Debug why pods in Kubernetes can't resolve external hostnames (CoreDNS misconfiguration)
- Identify split-brain DNS causing different resolution results inside vs outside your VPC
- Diagnose TTL issues causing stale records after a failover or IP change
- Verify Route 53 health check failover is working correctly
- Trace the full DNS resolution chain to identify slow or broken resolvers

---

## 🔧 Commands & Examples

### Basic dig Usage

```bash
# Simple A record lookup
dig example.com

# Short output — just the answer
dig +short example.com

# Look up a specific record type
dig example.com MX
dig example.com TXT
dig example.com NS
dig example.com AAAA
dig example.com SOA
dig example.com CNAME

# Look up PTR record (reverse DNS)
dig -x 8.8.8.8
dig -x 52.94.197.1   # find hostname for an AWS IP

# Query a specific nameserver directly (bypass /etc/resolv.conf)
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com A
dig @ns1.amazonaws.com example.com

# Query Route 53 nameserver directly
dig @ns-123.awsdns-45.com example.com

# All records for a domain (ANY query — not always honoured)
dig example.com ANY
```

### Tracing DNS Resolution

```bash
# Trace the full recursive resolution path (+trace does iterative queries)
dig +trace example.com

# Example trace output:
# . 518400 IN NS a.root-servers.net.           ← root NS
# com. 172800 IN NS a.gtld-servers.net.        ← .com TLD NS  
# example.com. 172800 IN NS ns1.example.com.   ← authoritative NS
# example.com. 300 IN A 93.184.216.34          ← final answer

# Trace a specific record type
dig +trace example.com MX

# Full verbose output (shows every step)
dig +qr +all example.com

# Show only the answer section
dig +noall +answer example.com

# Show answer + authority sections
dig +noall +answer +authority example.com

# Check if DNSSEC is enabled
dig +dnssec example.com
# Look for the AD (Authenticated Data) flag in the response header
```

### TTL and Caching Analysis

```bash
# Check the TTL of a record (important after DNS changes)
dig +ttl example.com
# Answer section shows TTL in seconds before the record type

# Find authoritative nameservers for a domain
dig +noall +answer example.com NS

# Query authoritative server directly (bypasses caching resolvers)
# This shows the real TTL configured at the source
AUTH_NS=$(dig +short example.com NS | head -1)
dig @$AUTH_NS +noall +answer +ttl example.com A

# Check systemd-resolved cache
resolvectl query example.com
resolvectl statistics
resolvectl flush-caches   # clear the resolver cache

# Check what /etc/resolv.conf points to
cat /etc/resolv.conf
# On systemd systems: nameserver 127.0.0.53
# options edns0 trust-ad

# Check nscd DNS cache (if running)
nscd -g | grep hostname

# Flush nscd cache
nscd -i hosts
```

### Kubernetes DNS Debugging

```bash
# Check CoreDNS is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS configmap
kubectl get configmap coredns -n kube-system -o yaml

# Run a debug pod to test DNS from inside the cluster
kubectl run dns-debug --image=busybox:1.28 --rm -it -- nslookup kubernetes.default
kubectl run dns-debug --image=nicolaka/netshoot --rm -it -- dig kubernetes.default.svc.cluster.local

# Common K8s DNS query patterns
# Service DNS: <service>.<namespace>.svc.cluster.local
# Pod DNS:     <pod-ip-dashes>.<namespace>.pod.cluster.local
dig @<coredns-cluster-ip> kubernetes.default.svc.cluster.local

# Check ndots setting (affects search path behavior)
kubectl run debug --image=busybox --rm -it -- cat /etc/resolv.conf
# search default.svc.cluster.local svc.cluster.local cluster.local
# nameserver 10.96.0.10
# options ndots:5

# ndots:5 means names with <5 dots try search domains first
# This causes: "api" → tries api.default.svc.cluster.local first (6 DNS queries before external!)

# Debug CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns -f

# Check CoreDNS metrics (if Prometheus is set up)
kubectl port-forward -n kube-system svc/kube-dns 9153:9153
curl localhost:9153/metrics | grep coredns_dns_request
```

### AWS Route 53 Diagnostics

```bash
# Find the Route 53 nameservers for a hosted zone
aws route53 list-hosted-zones --query 'HostedZones[?Name==`example.com.`].Id' --output text | \
  xargs -I{} aws route53 get-hosted-zone --id {} --query 'DelegationSet.NameServers' --output text

# Query Route 53 directly (verify records before DNS propagation)
dig @ns-1234.awsdns-12.org example.com A +short

# Check Route 53 health check status
aws route53 list-health-checks --output json | jq '.HealthChecks[] | {Id: .Id, Status: .HealthCheckConfig.FullyQualifiedDomainName}'

# Verify Route 53 failover is working
# Query both primary and secondary nameservers
REGION_NS=$(aws route53 get-hosted-zone --id /hostedzone/XXXXX --query 'DelegationSet.NameServers[0]' --output text)
dig @$REGION_NS api.example.com +short

# Test VPC internal DNS (169.254.169.253 is the VPC resolver on EC2)
dig @169.254.169.253 s3.amazonaws.com
dig @169.254.169.253 rds.endpoint.us-east-1.rds.amazonaws.com

# Check if a domain resolves differently inside vs outside the VPC (split-horizon DNS)
# From EC2:
dig @169.254.169.253 api.internal.example.com
# From local:
dig @8.8.8.8 api.internal.example.com
```

### Advanced dig Techniques

```bash
# Batch queries from a file
cat <<EOF > domains.txt
example.com
google.com
aws.amazon.com
EOF
while read domain; do
    echo -n "$domain: "
    dig +short "$domain" A
done < domains.txt

# Check SPF record (email security)
dig example.com TXT | grep spf

# Check DMARC policy
dig _dmarc.example.com TXT +short

# Check DKIM for a specific selector
dig default._domainkey.example.com TXT

# Find all subdomains in a zone (requires zone transfer — usually blocked)
dig axfr @ns1.example.com example.com

# Compare resolution from multiple resolvers
for resolver in 8.8.8.8 1.1.1.1 9.9.9.9 208.67.222.222; do
    echo -n "$resolver: "
    dig @$resolver +short +time=2 example.com A
done

# Check propagation — query multiple resolvers to see if change has propagated
RECORD="example.com"
for ns in $(dig +short $RECORD NS); do
    echo -n "$ns: "
    dig @$ns +short +time=2 $RECORD A
done

# Measure DNS query latency
for i in {1..5}; do
    dig @8.8.8.8 example.com | grep "Query time"
done

# Use drill (ldns) for cleaner DNSSEC tracing
drill -TD example.com
```

### /etc/resolv.conf and systemd-resolved

```bash
# Check current resolver configuration
cat /etc/resolv.conf

# Check systemd-resolved status and per-link DNS
resolvectl status

# Check which DNS server a specific interface uses
resolvectl status eth0

# Temporarily change DNS for a link
resolvectl dns eth0 8.8.8.8 8.8.4.4

# Check DNS search domains
resolvectl domain eth0

# View LLMNR, mDNS status
resolvectl status | grep -E "LLMNR|mDNS"

# On legacy systems — check /etc/nsswitch.conf for resolution order
grep "^hosts:" /etc/nsswitch.conf
# hosts: files dns
# files = /etc/hosts, dns = nameservers in /etc/resolv.conf

# Add a custom entry to /etc/hosts (fastest override, instant effect)
echo "10.0.1.100 internal-api.example.com" >> /etc/hosts
```

---

## ⚠️ Gotchas & Pro Tips

- **TTL is honored by caches, not by you:** When you change a DNS record, you must wait for the old TTL to expire on all caches worldwide. Pre-lowering TTL to 60 seconds 24 hours before a planned migration is the professional approach. Changing TTL only takes effect when clients re-query after the old TTL expires.

- **Kubernetes ndots:5 causes 6× DNS queries for external names:** With `ndots:5`, a name like `api.external.com` has only 3 dots, so K8s tries `api.external.com.default.svc.cluster.local`, `api.external.com.svc.cluster.local`, `api.external.com.cluster.local` before trying `api.external.com` directly. This causes latency for external calls. Set `ndots:2` in your pod spec if external DNS calls are latency-sensitive.

- **CNAME at zone apex is illegal:** You cannot have a CNAME for `example.com` itself (the apex). That's why services use ALIAS records (Route 53's ALIAS), ANAME, or CNAME flattening. Using CNAME at apex causes resolution failures for MX, NS, and other records.

- **`dig +short` hides CNAME chains:** If a name CNAMEs to another name, `dig +short` shows only the final IP. Use `dig +noall +answer` to see the full chain and spot unintended redirection.

- **Split-horizon DNS in AWS VPCs:** Route 53 private hosted zones return different answers inside the VPC than public DNS. Ensure your EC2 instances use the VPC resolver (`169.254.169.253`) not an external DNS server, or your private hostnames won't resolve.

- **DNS over TCP (port 53 TCP) matters:** DNS responses over 512 bytes (common with DNSSEC or many TXT records) fall back to TCP. Ensure your firewall allows TCP port 53 outbound, not just UDP. Blocking TCP/53 causes hard-to-diagnose failures with DNSSEC and large TXT records.

---

*Part of the [Linux Mastery](https://github.com/naveenramasamy11/linux-mastery) series.*
