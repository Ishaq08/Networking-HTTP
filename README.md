# DevOps Networking Labs: Week 2 (Days 1-2)

Hands-on labs covering networking fundamentals, DNS, HTTP, TLS and reverse proxies. Every lab follows the same loop: **learn, build, break safely, diagnose with evidence, fix, document.**

> Environment: Ubuntu VM (local). All "break" exercises were run on a local lab machine, never on a shared or production server.

## Contents

- [Repository structure](#repository-structure)
- [Day 1: TCP/IP, subnets, routing, TCP vs UDP, ports](#day-1-tcpip-subnets-routing-tcp-vs-udp-ports)
- [Day 2: DNS, HTTP, TLS, reverse proxy, curl](#day-2-dns-http-tls-reverse-proxy-curl)
- [Troubleshooting cheat sheet](#troubleshooting-cheat-sheet)
- [Incident log](#incident-log)
- [Interview questions practised](#interview-questions-practised)
- [Progress checklist](#progress-checklist)

## Repository structure

```
devops-networking-labs/
├── README.md
├── week-2/
│   ├── day1-notes.md          # subnetting answers, ip/ss output, tcpdump capture, break/fix write-ups
│   └── day2-notes.md          # dig +trace, curl timings, TLS inspection, break/fix write-ups
└── configs/
    └── nginx-myapp.conf       # reverse proxy configuration
```

---

## Day 1: TCP/IP, subnets, routing, TCP vs UDP, ports

### Concepts covered

| Topic | Key points |
| --- | --- |
| TCP/IP model | Application, Transport, Internet, Link. Troubleshoot bottom-up. |
| CIDR | `/n` = n network bits. Usable hosts = 2^(32-n) - 2. |
| Private ranges | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` |
| Routing | Local subnet = direct delivery. Otherwise, send to the default gateway. Most specific route wins. `0.0.0.0/0` = default route. |
| TCP | Connection-oriented, reliable, ordered. Three-way handshake: SYN, SYN-ACK, ACK. |
| UDP | Connectionless, low overhead. Used by DNS queries, DHCP, streaming. |
| Ports | 0-65535, identify the program on a host. A connection = source IP:port + destination IP:port. |

### Subnet reference

| CIDR | Netmask | Usable hosts |
| --- | --- | --- |
| /24 | 255.255.255.0 | 254 |
| /25 | 255.255.255.128 | 126 |
| /26 | 255.255.255.192 | 62 |
| /27 | 255.255.255.224 | 30 |
| /28 | 255.255.255.240 | 14 |
| /30 | 255.255.255.252 | 2 |

**Worked example: `192.168.1.0/24` split into four `/26` networks**

| Subnet | Usable range | Broadcast |
| --- | --- | --- |
| 192.168.1.0/26 | .1 - .62 | .63 |
| 192.168.1.64/26 | .65 - .126 | .127 |
| 192.168.1.128/26 | .129 - .190 | .191 |
| 192.168.1.192/26 | .193 - .254 | .255 |

### Common ports

| Port | Service | Port | Service |
| --- | --- | --- | --- |
| 22 | SSH | 3306 | MySQL |
| 25 | SMTP | 5432 | PostgreSQL |
| 53 | DNS (UDP/TCP) | 6379 | Redis |
| 67/68 | DHCP (UDP) | 8080 | Alt HTTP |
| 80 | HTTP | 9090 / 3000 | Prometheus / Grafana |
| 443 | HTTPS | 27017 | MongoDB |

### Labs completed

**Lab A: Inspect the machine**

```bash
ip addr show
ip route
ip route get 8.8.8.8
cat /etc/resolv.conf
traceroute 8.8.8.8
```

My results (fill in):

| Item | Value |
| --- | --- |
| IP / CIDR | |
| Default gateway | |
| DNS server | |
| Hops to 8.8.8.8 | |

**Lab B: Subnetting by hand, verified with tools**

```bash
ipcalc 192.168.1.0/26
python3 -c "import ipaddress; n=ipaddress.ip_network('10.0.0.0/16'); print(list(n.subnets(new_prefix=18)))"
```

**Lab C: Listening services**

```bash
sudo ss -tulpn
python3 -m http.server 8080
ss -tlnp | grep 8080
```

Key distinction: `0.0.0.0:22` listens on all interfaces. `127.0.0.1:631` is local only.

**Lab D: TCP handshake capture**

```bash
sudo tcpdump -i lo -n port 8080     # terminal 1
curl http://localhost:8080           # terminal 2
```

Observed `[S]`, `[S.]`, `[.]` for the handshake and `[F.]` at close. Capture saved in `week-2/day1-notes.md`.

### Break and fix

| # | Scenario | Symptom | Evidence | Cause | Fix |
| --- | --- | --- | --- | --- | --- |
| 1 | Port conflict | `Address already in use` | `ss -tlnp \| grep 8080` shows existing PID | Second process tried to bind the same port | `kill <PID>`, restart |
| 2 | Closed vs filtered port | Refused = instant failure. Timeout = hangs. | `curl -v`, `ufw status` | Refused: nothing listening. Timeout: firewall dropping packets. | Start the service or remove the firewall rule |
| 3 | Missing default route | `Network is unreachable` for external IPs | `ip route` has no `default via` line | Default route deleted | `ip route add default via <gateway>` |
| 4 | Wrong subnet mask (thought exercise) | Host sends local traffic to the gateway, or the reverse | Compare mask against destination | Mask too narrow or too wide | Correct the prefix length |

**Refused vs timeout (frequent interview question):**
- **Connection refused:** the host answered and said no port is listening. Check the service.
- **Connection timed out:** no answer at all. Suspect a firewall, security group or routing problem.

---

## Day 2: DNS, HTTP, TLS, reverse proxy, curl

### Concepts covered

**DNS resolution order:** browser/OS cache, `/etc/hosts`, stub resolver, recursive resolver, root server, TLD server, authoritative server. The answer is cached for the record's TTL.

| Record | Purpose |
| --- | --- |
| A / AAAA | Name to IPv4 / IPv6 |
| CNAME | Alias |
| MX | Mail servers |
| TXT | SPF, verification |
| NS | Authoritative nameservers |

**HTTP status codes**

| Code | Meaning | Typical cause |
| --- | --- | --- |
| 200 | OK | Success |
| 301 | Moved permanently | http to https redirect |
| 400 | Bad request | Malformed request |
| 401 | Unauthorized | Not authenticated |
| 403 | Forbidden | Authenticated but not permitted, or file permissions |
| 404 | Not found | Wrong path |
| 500 | Internal server error | Application bug or crash |
| 502 | Bad gateway | Proxy cannot reach or understand the backend |
| 503 | Service unavailable | Backend overloaded, down or in maintenance |
| 504 | Gateway timeout | Backend too slow |

**TLS:** the client verifies the certificate is issued by a trusted CA, matches the hostname and has not expired. Common failures: expired certificate, hostname mismatch, self-signed or untrusted CA, incomplete chain.

**Reverse proxy:** sits in front of backends and handles TLS termination, load balancing, caching and a single entry point.

### Labs completed

**Lab A: DNS**

```bash
dig example.com
dig +short example.com
dig example.com MX
dig @8.8.8.8 example.com
dig +trace example.com
resolvectl status
```

**Lab B: HTTP with curl**

```bash
curl -I http://example.com
curl -v http://example.com
curl -L http://github.com
curl -s -o /dev/null -w "%{http_code}\n" https://example.com
curl -s -o /dev/null -w "DNS: %{time_namelookup}s Connect: %{time_connect}s TLS: %{time_appconnect}s Total: %{time_total}s\n" https://example.com
```

The timing breakdown shows where latency lives: DNS lookup, TCP connect, TLS handshake or server response.

**Lab C: TLS inspection**

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

**Lab D: Nginx reverse proxy**

Request flow: `client -> Nginx :80 -> backend 127.0.0.1:8081`

```nginx
server {
    listen 80;
    server_name myapp.local;

    location / {
        proxy_pass http://127.0.0.1:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -v http://myapp.local
sudo tail -f /var/log/nginx/error.log
```

Always run `nginx -t` before reloading.

### Break and fix

| # | Scenario | Symptom | Evidence | Cause | Fix |
| --- | --- | --- | --- | --- | --- |
| 1 | Dead DNS server | `Could not resolve host`. `ping 8.8.8.8` works, `ping example.com` fails. | `dig @<dead-ip>` times out. `dig @8.8.8.8` works. | Bad resolver configured | Restore a working nameserver |
| 2 | Wrong `/etc/hosts` entry | Request hangs, wrong IP | `dig +short` shows real IP. `getent hosts` shows the bad one. | Static entry overrides DNS | Remove the line |
| 3 | Nginx stopped | `Connection refused` | `ss -tlnp \| grep :80` shows nothing | Service not running | `systemctl start nginx` |
| 4 | Backend down | `502 Bad Gateway` | Error log: `connect() failed (111: Connection refused)` | Nginx cannot reach the upstream | Restart the backend |
| 5 | Missing page | `404 Not Found` | `curl -i` | Path does not exist | Correct the path |

**Key tooling note:** `dig` ignores `/etc/hosts`. `getent hosts` uses the same resolution path as applications.

---

## Troubleshooting cheat sheet

**Reachable by IP but not by name**
1. Confirm the IP works (`ping`, `curl` by IP).
2. `dig +short name` and `getent hosts name`.
3. Compare resolvers: `dig @8.8.8.8 name` vs the default.
4. Check `/etc/hosts` and `resolvectl status`.
5. Verify the DNS record itself and its TTL.
6. Flush caches if stale: `resolvectl flush-caches`.

**General investigation order**
`DNS -> connectivity -> port -> service -> logs`

| Symptom | First suspect | First command |
| --- | --- | --- |
| Name does not resolve | DNS | `dig`, `getent hosts` |
| Connection refused | Service down or wrong port | `ss -tlnp` |
| Connection timeout | Firewall or routing | `ip route`, `ufw status`, traceroute |
| 502 | Backend unreachable | Nginx `error.log` |
| 504 | Backend slow | Backend logs, `curl -w` timings |
| TLS error | Certificate | `openssl s_client` |

---

## Incident log

| Date | Symptom | Investigation | Root cause | Resolution | Prevention |
| --- | --- | --- | --- | --- | --- |
| | | | | | |
| | | | | | |
| | | | | | |

Goal: at least three fully documented incidents by the end of the roadmap.

---

## Interview questions practised

**Day 1**
1. TCP vs UDP, with one real use for each
2. The TCP three-way handshake
3. CIDR and hosts in a /27
4. Default gateway and `0.0.0.0/0`
5. Public vs private IPs and the private ranges
6. "Connection refused" vs "Connection timed out"
7. Finding the process using port 8080
8. Why DNS uses UDP 53 (and sometimes TCP)

**Day 2**
1. What happens after you enter a URL in a browser
2. DNS vs DHCP
3. Explaining HTTP status codes
4. 401 vs 403
5. What a reverse proxy is and why use one
6. Troubleshooting "reachable by IP but not by name"
7. 502 vs 504
8. What TLS checks a client performs on a certificate

---

## Progress checklist

**Day 1**
- [ ] Notes on layers, CIDR, routing, TCP vs UDP
- [ ] Subnetted `192.168.1.0/24` into four /26s by hand
- [ ] Recorded IP, gateway, DNS and hop count
- [ ] Captured a TCP handshake with tcpdump
- [ ] Fixed a port conflict and a missing default route
- [ ] Can explain refused vs timeout
- [ ] Memorised 10 common ports

**Day 2**
- [ ] Notes on DNS, HTTP, TLS
- [ ] Ran `dig +trace` and explained each hop
- [ ] Used `curl -v`, `-I`, `-L` and `-w` timings
- [ ] Inspected a TLS certificate with `openssl`
- [ ] Nginx reverse proxy working
- [ ] Reproduced DNS failure, connection refused and 502
- [ ] Can explain 401 vs 403 and 502 vs 504

## Self-assessment (0-2)

| Skill | Score | Notes |
| --- | --- | --- |
| Linux | | |
| Networking | | |
| DNS / HTTP | | |
| Git | | |

*0 = cannot do independently. 1 = can do with notes. 2 = can demonstrate and explain independently.*

## Next

Day 3: Git branches, merge conflicts, pull requests, `.gitignore` and recovery.
