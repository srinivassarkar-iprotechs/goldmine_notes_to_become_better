## The Mental Model First

Every network problem falls into one of these buckets:

**"Can I reach it?" → "Can I connect?" → "Is the service responding?" → "Is it fast/secure enough?"**

Always troubleshoot in that order. Don't dive into packet captures when DNS is broken.

---

## Tier 1: The 20% You'll Use 80% of the Time

### 1. `ss` — Socket State (replaces `netstat`)
```bash
ss -tlnp          # TCP listening ports + which process owns them
ss -tlnp | grep 80
ss -s             # summary of all socket states
ss -tnp state established  # active connections
```
**What to look for:** Is your service actually listening? On `0.0.0.0` (all interfaces) or `127.0.0.1` (only localhost)? That one distinction solves 30% of "I can't reach my service" tickets.

---

### 2. `ip` — Everything about interfaces & routing
```bash
ip addr show              # all IPs on all interfaces (is the NIC up? right IP?)
ip link show              # link state - is it UP or DOWN?
ip route show             # routing table - where does traffic go?
ip route get 8.8.8.8      # which interface/gateway handles THIS specific destination
```
**What to look for:** Missing default route (`0.0.0.0/0`) = can't reach internet. Wrong interface IP = misconfigured DHCP or cloud metadata. `state DOWN` on a link = NIC/cable issue.

---

### 3. `ping` + `traceroute`/`mtr`
```bash
ping -c 4 8.8.8.8         # basic reachability + latency
ping -c 4 google.com      # if this fails but IP ping works = DNS problem
mtr --report google.com   # combines ping + traceroute, shows per-hop packet loss
traceroute -n 8.8.8.8     # -n skips DNS lookups (faster)
```
**What to look for:** With `mtr`, packet loss that starts at hop N and continues = problem AT hop N. Loss only at one middle hop but recovery after = ICMP rate-limiting (usually fine, ignore it).

---

### 4. `dig` / `nslookup` — DNS
```bash
dig google.com                    # full DNS response
dig google.com +short             # just the IPs
dig @8.8.8.8 google.com          # query a SPECIFIC DNS server (bypass your local resolver)
dig google.com A                  # A record
dig google.com MX                 # mail records
dig -x 142.250.80.46              # reverse lookup
```
**What to look for:** If `dig @8.8.8.8 domain.com` works but `dig domain.com` fails = your local DNS (`/etc/resolv.conf`) is broken. Compare TTL values to understand caching delays.

---

### 5. `curl` — HTTP(S) Troubleshooting
```bash
curl -v https://example.com                    # verbose: see TLS handshake, headers
curl -I https://example.com                    # headers only
curl -o /dev/null -w "%{http_code} %{time_total}\n" https://example.com  # status + timing
curl --resolve example.com:443:1.2.3.4 https://example.com  # test with specific IP (bypass DNS)
curl -k https://example.com                    # skip TLS verification (test cert issues)
```
**What to look for:** `-v` shows you exactly where it fails — DNS resolution, TCP connect, TLS handshake, or HTTP response. The `time_total` breakdown reveals if latency is DNS, connect, or TTFB.

---

### 6. `tcpdump` — Packet Capture (when you need ground truth)
```bash
tcpdump -i eth0 port 80                       # capture HTTP on interface eth0
tcpdump -i any host 10.0.0.5                  # all traffic to/from a host
tcpdump -i any port 53 -n                     # watch DNS queries live
tcpdump -w capture.pcap -i eth0 port 443      # save to file for Wireshark
```
**What to look for:** SYN sent but no SYN-ACK = firewall dropping packets or server not listening. SYN+SYN-ACK+RST = port is closed. Lots of retransmits = packet loss in the path.

---

### 7. `netcat` (`nc`) — The Swiss Army Knife
```bash
nc -zv 10.0.0.5 22          # test if port 22 is open (z=scan, v=verbose)
nc -zv 10.0.0.5 80 443      # test multiple ports
nc -l 9999                  # listen on port 9999 (test if firewall allows inbound)
echo "" | nc -w1 host port  # quick TCP connect test from scripts
```
**What to look for:** `Connection refused` = service not running. `Timed out` = firewall. `Connected` = port is open. Use this to isolate "is it the app or the network?"

---

### 8. `iptables` / `nftables` — Firewall State
```bash
iptables -L -n -v           # list all rules with packet counts
iptables -L -n -v --line-numbers  # with line numbers for editing
iptables -t nat -L -n -v    # NAT table (port forwarding, masquerade)
nft list ruleset            # nftables equivalent (newer systems)
```
**What to look for:** Unexpected DROP or REJECT rules. High packet counts on a DROP rule = something is being actively blocked. Check INPUT, OUTPUT, FORWARD chains.

---

## Tier 2: DevSecOps-Specific Commands (high value, less frequent)

```bash
# TLS/Certificate inspection
openssl s_client -connect example.com:443 -servername example.com
# Look for: certificate chain, expiry date, cipher suite

# Who's connecting to what (security audit)
ss -tnp state established
lsof -i -P -n | grep LISTEN

# Check open ports from outside (on target machine or use nmap from outside)
nmap -sV -p 1-65535 target_ip   # service version detection

# Network namespace (containers/K8s)
ip netns list
ip netns exec <ns> ip addr show

# Check DNS from inside a container
kubectl exec -it pod -- nslookup kubernetes.default
```

---

## The Troubleshooting Playbook (follow this order every time)

```
Problem: "Can't reach service X"

1. ip addr show          → Am I on the right network?
2. ip route get X        → Do I have a route to X?
3. ping X                → Basic ICMP reachability
4. dig X (if hostname)   → DNS resolving correctly?
5. nc -zv X port         → Is the port open?
6. curl -v http://X:port → Is the service responding correctly?
7. ss -tlnp on server    → Is the service actually listening?
8. iptables -L on server → Is a firewall blocking it?
9. tcpdump               → Ground truth at packet level
```

---

## Key Files to Know

| File | Purpose |
|------|---------|
| `/etc/resolv.conf` | DNS servers + search domains |
| `/etc/hosts` | Static DNS overrides |
| `/etc/sysctl.conf` | Kernel network parameters (ip_forward, etc.) |
| `/proc/net/dev` | Interface statistics (raw) |
| `journalctl -u NetworkManager` | Network service logs |

---

## The 3 Mental Shortcuts That Separate Experts

**1. Localhost vs. All interfaces** — `127.0.0.1:8080` can't be reached externally. `0.0.0.0:8080` can. This is the #1 overlooked issue.

**2. Stateful firewall asymmetry** — A packet can get IN but the RETURN packet gets blocked. Always test both directions.

**3. DNS caching layers** — Your app → OS resolver → local DNS → upstream. `dig @8.8.8.8` bypasses all local layers and tells you the source of truth.

---

Start by drilling commands 1-4 (`ss`, `ip`, `ping/mtr`, `dig`) until they're muscle memory. Those alone handle the majority of real-world problems. Add `curl`, `tcpdump`, and `nc` next. The rest you'll reach for situationally.
