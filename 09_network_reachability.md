# Router Network Reachability Diagnosis and Web UI Verification

## Assessment Context
- Assessment Date: 2026-10-22
- Assessment Host: Kali Linux at 192.168.28.6
- Target Router IP: 192.168.0.1 (Airtel router administration Web UI)
- Previous Static Analysis: Complete (files 01-05 created under /home/dc/Desktop/New Folder/router-webui-audit/)
- Safety Mode: Read-only diagnostics only

## Phase 2 — Host Network Configuration

### Interface Summary (from `ip -br -4 addr`)
| Interface | IP Address | Status |
|-----------|------------|--------|
| `lo` | 127.0.0.1/8 | Loopback |
| `eth1` | 192.168.28.6/24 | UP, assessment host interface |
| `docker0` | 172.17.0.1/16 | UP, Docker bridge |

### Routing Table (from `ip -4 route show`)
- `default via 192.168.28.1 dev eth1 proto dhcp src 192.168.28.6 metric 100`
- `172.16.235.0/24 dev br-39f4a37ade11 proto kernel scope link src 172.16.235.1 linkdown`
- `172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1`
- `192.168.28.0/24 dev eth1 proto kernel scope link src 192.168.28.6 metric 100`

### Route to Target (from `ip route get 192.168.0.1`)
```
192.168.0.1 via 192.168.28.1 dev eth1 src 192.168.28.6 uid 1000 cache
```
- Egress interface: eth1
- Source IP: 192.168.28.6
- Next-hop gateway: 192.168.28.1

### Neighbour Table (from `ip neigh show`)
| IP Address | Interface | MAC Address | State |
|------------|-----------|-------------|-------|
| 192.168.28.1 | eth1 | 60:98:49:f0:f3:b1 | REACHABLE (default gateway) |
| 172.17.0.2 | docker0 | 06:fa:1f:2a:34:07 | STALE |
| fe80::6298:49ff:fef0:f3b1 | eth1 | 60:98:49:f0:f3:b1 | router (link-local) |

**Key observation:** 192.168.0.1 is absent from the neighbour table — not on the directly connected LAN.

## Phase 3 — Basic Reachability

### ICMP Ping (`ping -c 2 -W 2 192.168.0.1`)
- Result: 100% packet loss, 2/2 packets transmitted received 0
- Code exit: 1 (failure)

### ARP Probe (`arping -I eth1 -c 2 192.168.0.1`)
- Result: 0 response(s) from 2 broadcast(s)
- Finding: Target not on local Ethernet subnet 192.168.28.0/24

### HTTP (`curl -sS -I --connect-timeout 3 --max-time 8 http://192.168.0.1/`)
- Result: Connection timed out after 3001 ms
- Error: curl (28)

### HTTPS (`curl -sS -I --connect-timeout 3 --max-time 8 https://192.168.0.1/`)
- Result: Connection timed out after 3001 ms
- Error: curl (28)

## Phase 4 — Path Diagnosis (traceroute)

### `traceroute -m 5 192.168.0.1`
```
1  dsldevice.lan (192.168.28.1)       0.652 ms  0.551 ms  0.961 ms
2  192.168.1.1                        3.320 ms  3.349 ms  3.240 ms
3  192.168.225.1                      3.838 ms  4.781 ms  13.249 ms
4  * * *
5  * * *
```
- Hop 1: Default gateway 192.168.28.1 (dsldevice.lan) — reachable
- Hop 2: 192.168.1.1 — another router on the carrier network
- Hop 3: 192.168.225.1 — further transit node
- Hops 4-5: No response — target not reachable via this path

## Phase 5 — Most Likely Cause Classification
1. **No usable route to the target** — Traceroute terminates at hop 3 (192.168.225.1) with no further path to 192.168.0.1
2. **Target host does not respond** — ICMP/pcurl timeouts; 192.168.0.1 not in neighbour table
3. **Target belongs to a different routed subnet** — 192.168.0.1 is outside the 192.168.28.0/24 LAN and not reachable via the observed gateway path

## Phase 6 — Live Web UI Verification
**Status: BLOCKED** — No HTTP or HTTPS connection could be established. All protocol attempts (ping, HTTP, HTTPS) resulted in timeouts or packet loss.

## Phase 7 — Preserved Static Analysis
The five existing report files were **not modified**:
- 01_target_baseline.md — unchanged
- 02_webui_inventory.md — unchanged
- 03_endpoint_inventory.md — unchanged
- 04_authentication_sessions.md — unchanged
- 05_password_change_static_analysis.md — unchanged

New file created: 09_network_reachability.md

## Phase 8 — Final Summary

### Root Cause / Best-Supported Diagnosis
The router at 192.168.0.1 is **not network-reachable** from the Kali assessment host at 192.168.28.6. The packet path reaches the default gateway (192.168.28.1, "dsldevice.lan") and hops through intermediate routers (192.168.1.1, 192.168.225.1), but cannot reach the target IP. The target is either on a different network segment not served by the observed route, or the host at 192.168.0.1 is powered off/isolating, or there is a routing gap beyond the observed hops.

### Evidence Summary
| Command | Result |
|---------|--------|
| `ip -br -4 addr` | eth1: 192.168.28.6/24 (assessment host); docker0: 172.17.0.1/16 |
| `ip -4 route show` | Default via 192.168.28.1; subnet 192.168.28.0/24 on eth1 |
| `ip route get 192.168.0.1` | via 192.168.28.1 dev eth1 src 192.168.28.6 |
| `ip neigh show` | 192.168.28.1 REACHABLE; 192.168.0.1 absent from neighbour table |
| `ping -c 2 -W 2 192.168.0.1` | 100% packet loss |
| `curl http/https 192.168.0.1` | Connection timed out (curl 28) |
| `arping -I eth1 192.168.0.1` | 0 responses |
| `traceroute -m 5 192.168.0.1` | Stops at hop 3 (192.168.225.1); no route to 192.168.0.1 |

### Target Reachability
- Network reachable: NO
- Web UI responds (HTTP): N/A — no connection established
- Web UI responds (HTTPS): N/A — no connection established
- Authentication possible: Cannot determine — connectivity blocked

### Confirmed Facts vs Hypotheses
- **Confirmed**: Kali host on 192.168.28.6/24 via eth1; gateway 192.168.28.1 reachable; traceroute reaches hop 3; 192.168.0.1 not in ARP neighbour table; all protocol tests timeout
- **Hypothesis**: 192.168.0.1 may be a different router/AP on another VLAN/subnet; router may be powered off or in a different network zone not routed from this path

### Safe Recommended Next Steps
1. Verify with network operations whether 192.168.0.1 is a valid router IP in the Airtel network infrastructure, or if the correct management IP is different (e.g., 192.168.1.1, 192.168.1.254, etc.)
2. Test from inside the router's subnet if authorized, to confirm reachability from that side
3. Review routing policy if the assessment should include this router; an authorized operator may need to add a route or adjust gateway configuration
4. Leverage static analysis results: The 5 existing report files (01-05) contain complete password-change flow, session mechanics, and governance analysis derived from binary/confidential evidence — these remain valid
