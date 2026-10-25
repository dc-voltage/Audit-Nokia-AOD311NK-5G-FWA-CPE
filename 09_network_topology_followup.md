# Router Network Topology Investigation — Follow-Up

## Assessment Context
- **Assessment Date:** 2026-10-22
- **Assessment Host:** Kali Linux at 192.168.28.6/24 (eth1)
- **Target Router IP:** 192.168.0.1 (Airtel router administration Web UI)
- **Previous Reports Preserved:** 01-05 under /home/dc/Desktop/New Folder/router-webui-audit/ (unchanged)
- **This Report:** 09_network_topology_followup.md — topology diagnostics only

## Current Network Topology (Evidence Summary)

### Interface Configuration (from `ip -br -4 addr`)
| Interface | IP Address | Status | MAC Address |
|-----------|------------|--------|-------------|
| `lo` | 127.0.0.1/8 | Loopback | — |
| `eth1` | 192.168.28.6/24 | UP, assessment host LAN interface | 00:0f:c9:23:fd:85 |
| `eth0` | — | UP, no IPv4 address assigned | 00:e0:4c:3c:14:b0 |
| `docker0` | 172.17.0.1/16 | UP, Docker bridge | 62:8e:ad:8e:d7:c4 |
| `br-39f4a37ade11` | — | DOWN, no carrier | b2:36:1d:a1:4d:b1 |

**Key:** The assessment host is on subnet 192.168.28.0/24 via eth1. No other interface has an IP in this subnet.

### Routing Table (from `ip -4 route show table all`)
| Route | Interface | Source | Metric | Table |
|-------|-----------|--------|--------|-------|
| `default via 192.168.28.1 dev eth1` | eth1 | 192.168.28.6 | 100 | main |
| `172.16.235.0/24 dev br-39f4a37ade11` | br-39f4a37ade11 | 172.16.235.1 | — | main (linkdown) |
| `172.17.0.0/16 dev docker0` | docker0 | 172.17.0.1 | — | main |
| `192.168.28.0/24 dev eth1` | eth1 | 192.168.28.6 | 100 | main |

**Default route:** via 192.168.28.1 dev eth1 (the assessment host's gateway).

### Routing Rules (from `ip -4 rule show`)
| Priority | Lookup | Action |
|----------|--------|--------|
| 0 | local | from all |
| 32766 | main | from all |
| 32767 | default | from all |

**No custom policy routing** — standard kernel rule order applies.

### Route Selection for 192.168.0.1 (from `ip route get 192.168.0.1`)
```
192.168.0.1 via 192.168.28.1 dev eth1 src 192.168.28.6 uid 1000 cache
```
- **Selected egress interface:** eth1
- **Selected source IP:** 192.168.28.6
- **Selected next-hop gateway:** 192.168.28.1
- **Routing table:** main (standard kernel selection)

**Evidence:** The kernel selects the default route via the only egress interface (eth1) with the source set to the host's address on that interface. No alternative routes or policy exceptions are configured.

### Neighbour Table (from `ip neigh show`)
| IP Address | Interface | MAC Address | State |
|------------|-----------|-------------|-------|
| 192.168.28.1 | eth1 | 60:98:49:f0:f3:b1 | REACHABLE (default gateway) |
| 172.17.0.2 | docker0 | 06:fa:1f:2a:34:07 | STALE |
| fe80::6298:49ff:fef0:f3b1 | eth1 | 60:98:49:f0:f3:b1 | router (link-local) |

**Key observation:** 192.168.0.1 is **absent** from the neighbour table — the kernel has no Layer 2 mapping for the target on any interface.

## Traceroute Path Analysis (from `traceroute -m 5 -w 2 192.168.0.1`)

```
1  dsldevice.lan (192.168.28.1)       0.600 ms  0.942 ms  1.285 ms
2  192.168.1.1                         3.809 ms  3.900 ms  2.582 ms
3  192.168.225.1                       17.775 ms  4.627 ms  19.093 ms
4  * * *
5  * * *
```

### Hop 1 — 192.168.28.1 (dsldevice.lan)
- **Status:** REACHABLE — responds to ICMP and traceroute probes
- **Role:** Default gateway for the assessment host's subnet (192.168.28.0/24)
- **Evidence:** Consistent low latency (~1 ms); directly connected via eth1; MAC 60:98:49:f0:f3:b1 in neighbour table
- **Proves:** The default gateway is online and forwarding traffic

### Hop 2 — 192.168.1.1
- **Status:** REACHABLE — responds to ICMP and traceroute probes after ~3.8 ms
- **Role:** Upstream carrier/ISP router
- **Evidence:** Consistent response after hop 1; likely the next router in the carrier path
- **Proves:** Traffic from the default gateway egress reaches at least one upstream router

### Hop 3 — 192.168.225.1
- **Status:** REACHABLE — responds after ~4.6–19.1 ms
- **Role:** Further transit node in the carrier network
- **Evidence:** Consistent ICMP responses; third hop in the path
- **Proves:** Traffic traverses at least three network hops before stopping

### Hops 4–5 — No Response
- **Status:** No response (both hops show `* * *`)
- **What this proves:** The path terminates at hop 3; no further route to 192.168.0.1 is available via the observed path
- **What remains uncertain:** Whether the target 192.168.0.1 exists at all, whether it is behind a firewall that blocks ICMP but not IP, or whether it is simply inactive/off

## Interpretation of the Traceroute Path

| Aspect | Conclusion |
|--------|------------|
| **Path reaches hop 3** | Confirmed — traffic traverses 3 hops from the gateway |
| **Path does not reach 192.168.0.1** | Confirmed — hops 4–5 return no response |
| **Target is NOT on the directly connected LAN** | Confirmed — ARP probe on eth1 for 192.168.0.1 returned 0 responses; 192.168.0.1 not in neighbour table |
| **Target may be on a different subnet reachable via a different gateway** | Plausible but unconfirmed — no alternate routes or policies are configured |
| **Target may be powered off or isolated** | Plausible — no response from any protocol (ping, HTTP, HTTPS, ARP) |
| **Target may exist at a different IP address** | Possible — the assessment may be targeting an incorrect or stale IP |
| **Firewall filtering beyond hop 3** | Possible but unproven — consistent timeouts across all protocols ICMP/TCP/UDP suggest routing gap rather than selective filtering |

## Most Likely Explanation (with Confidence Rating)

### Primary Hypothesis: Target Not Reachable via Observed Path — **High Confidence (≈85%)**
- The kernel selects the default route via 192.168.28.1 → eth1, which reaches hop 3 (192.168.225.1) but no further.
- 192.168.0.1 is not on the directly connected LAN (192.168.28.0/24), confirmed by ARP probe and neighbour table absence.
- The target is either on a different subnet not served by the current gateway path, or the host at 192.168.0.1 is powered off/isolation.
- **Evidence supporting:** ping/HTTP/HTTPS timeouts; ARP probe 0 responses; 192.168.0.1 absent from neighbour table; traceroute stops at hop 3.

### Secondary Hypothesis: Incorrect or Stale Target IP — **Medium Confidence (≈10%)**
- The router's actual management interface may use a different IP (e.g., 192.168.1.1, 192.68.1.254, 10.0.0.1, etc.).
- The traceroute path (hops 1–3) suggests a carrier-grade NAT or ISP routing setup where 192.168.0.1 may not be a customer-accessible address.
- **Evidence supporting:** Private-address range (192.168.0.0/16) is commonly used internally; Airtel routers sometimes use 192.168.1.1 as the management address; the traceroute reaches the carrier network, suggesting 192.168.0.1 may not be routed to customer premises.

### Tertiary Hypothesis: Firewall/Filtering Beyond Hop 3 — **Low-Medium Confidence (≈5%)**
- A firewall between hops 3 and the target may block ICMP while allowing IP traffic (or vice versa).
- However, HTTP and HTTPS also time out, which argues against selective ICMP filtering alone.
- **Evidence supporting:** Consistent timeouts across ICMP, TCP (curl), and HTTPS; however, the lack of any response makes this harder to confirm.

### Excluded Hypotheses (with Reasoning)
- **Directly connected LAN on eth1:** Excluded — ARP probe returned 0 responses; 192.168.0.1 not in neighbour table.
- **Misconfigured static route on the Kali host:** Excluded — `ip route get` shows standard default-route selection; no custom routes.
- **Docker/bridge interference:** Excluded — br-39f4a37ade11 is linkdown; docker0 is separate subnet (172.17.0.0/16).
- **DNS resolution failure:** Excluded — static IP address used; no DNS involved.

## Alternative Explanations That Cannot Yet Be Excluded

| Hypothesis | Confidence | Required Evidence to Confirm |
|------------|------------|------------------------------|
| Target 192.168.0.1 is powered off or in deep sleep | Medium | Power-state check from an on-site host; `wake_on_lan` query if supported |
| Target exists at 192.168.1.1 (common Airtel router management IP) | Medium | Test `curl http://192.168.1.1/` from the assessment host; confirm via operator |
| Target exists at 192.168.1.254 (alternative router IP) | Low-Medium | Test `curl http://192.168.1.254/` from the assessment host |
| Target is on a VLAN-separated subnet reachable via a different gateway | Low | Operator to provide VLAN configuration and gateway IP for the router's management subnet |
| Intermediate carrier equipment (hop 3 = 192.168.225.1) blocks or NATs traffic to 192.168.0.1 | Low | Operator to confirm carrier network topology; traceroute from inside the carrier network |
| Target IP is correct but requires authentication or specific HTTP headers to respond | Low | Test from a host on the same LAN as the target; or obtain credentials for a test login |

## Insufficient Evidence — What Additional Information the Operator Needs

To proceed safely, the operator should collect **one or more** of the following:

1. **Router management IP confirmation** — The actual Airtel router administration address (common values: 192.168.1.1, 192.168.1.254, 10.0.0.1, 192.168.0.1 may be a WAN IP, not LAN management IP).

2. **Network diagram or topology document** — Showing how the Kali assessment host (192.168.28.6) connects to the target router, including all intermediate routers, gateways, and VLANs.

3. **Default gateway or alternate gateway IP** — If the router is on a different subnet, the gateway IP that provides reachability from the assessment host's network.

4. **Physical connectivity verification** — Confirmation that 192.168.0.1 is a valid, powered-on host reachable from the assessment environment; or that the correct IP is different.

5. **Route or VLAN configuration from the operator** — If the router is on a subnet not directly reachable from 192.168.28.0/24, the operator would need to add a static route or provide the correct gateway.

6. **Ping/HTTP test from a different host** — If another host on the same or different LAN can reach 192.168.0.1, this would confirm reachability (or lack thereof) without changing the Kali host configuration.

## Recommended Next Safe Step

**Do not change the Kali host's network configuration.** Instead:

1. **Verify the correct router management IP with the operator** — Confirm whether 192.168.0.1 is the correct administration address, or if the router uses 192.168.1.1, 192.168.1.254, or another IP.

2. **If authorized, test from a different network segment** — If another host is available on the router's LAN, attempt `ping` and `curl http://192.168.0.1/` from that host to confirm reachability.

3. **Leverage the existing static analysis** — The 5 report files (01-05) under `/home/dc/Desktop/New Folder/router-webui-audit/` contain the complete password-change flow, session mechanics, and governance analysis derived from binary strings and `caps.json`. These remain valid regardless of network reachability.

4. **If the operator confirms a different management IP**, update the assessment target and repeat the reachability diagnostics (phases 2-4) against the correct IP.

5. **Do not add routes, change gateways, or modify firewall rules** without explicit authorization and a clear need.

---
**End of Report**

*All confirmed observations are based on executed read-only diagnostics. Hypotheses are explicitly marked with confidence ratings and distinguished from verified findings. No existing reports (01-05) were modified.*
