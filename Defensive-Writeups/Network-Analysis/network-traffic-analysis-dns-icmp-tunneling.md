# Network Traffic Analysis — DNS Tunneling and ICMP Tunneling

**Author:** Spencer Leach  
**Date:** March 2, 2026  
**Platform:** HackTheBox / Pikes Peak State College Lab  
**Topics:** Wireshark, DNS Tunneling, ICMP Tunneling, C2 Detection, Network Forensics

---

## Overview

This skills assessment involved analyzing two PCAP files to identify and characterize malicious network activity. The analysis covers a DNS tunneling campaign and an ICMP-based attack that combined flooding as a smokescreen with covert tunneling for data exfiltration.

---

## Device Identification

**DNS Traffic**
- `10.0.2.30` — DNS client (tunnel initiator)
- `10.0.2.20` — DNS server (all responses originated here)

**ICMP Traffic**
- `10.13.137.145` — Attacker host
- `192.168.178.34` — Victim host

---

## Part 1: DNS Analysis

### Findings

| Indicator | Observation | Significance |
|-----------|-------------|--------------|
| All queries are NULL record type | NULL is most efficient for raw data transfer | Hallmark of DNS tunneling tools |
| One client → one server | 10.0.2.30 → 10.0.2.20 exclusively | C2 relationship, not a flood |
| 125 queries/second starting at ~13s | Sudden traffic burst | Tunnel establishment and data transfer begin |
| More queries than responses | Selective acknowledgment by server | Client encoding data chunks upstream |
| 6 responses significantly larger than queries (710–1512 bytes) | Responses exceed 512 bytes | Downstream data transfer through tunnel |
| Base64/Base32 decoding returns garbled output | Data is encoded beyond simple Base64/32 | Likely obfuscated or double-encoded |
| `dns.resp.ttl == 0` accounts for 48.4% of packets | TTL=0 prevents caching | Consistent with iodine tunneling tool behavior |
| Zero TCP port 53 packets | Iodine forces large packets over UDP | Evades TCP-based inspection |
| No Base64 padding (`==`) in subdomains | No standard Base64 padding | Further indication of iodine encoding scheme |
| Subdomains follow pattern: `xxxx.pirate.sea` | High-volume unique subdomains | Sequential encoded data fragments being chunked |

### Verdict

**This attack was DNS tunneling, most likely using the iodine tool.**

The combination of NULL record types, TTL=0 responses, UDP-only large packets, and the absence of Base64 padding are consistent with iodine's known behavior. The varying prefixes on `pirate.sea` subdomains represent chunked, encoded data fragments being exfiltrated upstream.

**Inference about the attacker's position:** The presence of an attacker-controlled authoritative DNS server for `pirate.sea` indicates significant pre-planning. DNS had to be configured to route `.pirate.sea` queries to `10.0.2.20`, meaning the attacker had either prior network access or configuration control. DNS tunneling is a persistence and evasion mechanism — its presence indicates the attacker had already established a foothold and was maintaining covert access or exfiltrating data while blending with normal traffic.

---

## Part 2: ICMP Analysis

### Evidence of Flooding

| Indicator | Observation | Significance |
|-----------|-------------|--------------|
| 33.4% of traffic is ICMP echo requests | Disproportionately high ratio | Consistent with flooding |
| Only 2 source IPs | One-to-one attack | DoS, not DDoS |
| 200+ packets/second spike at ~3 second mark | Large automated burst | Flooding tool used |
| Flood begins 3 seconds after capture starts | Triggered event | Attacker was already present on network |
| Over 50% of captured traffic is replies | Server not overwhelmed | Flood was not the primary objective |

### Evidence of Tunneling

| Indicator | Observation | Significance |
|-----------|-------------|--------------|
| One packet at 1,117 bytes | ~20x standard ICMP size (32–64 bytes) | Entire network packet wrapped in one ICMP packet |
| Non-standard data in large packet | Data attributed to Stunnel developers, Warsaw, Poland | Documented ICMP tunneling tool identified |
| Consistent back-and-forth between two IPs | Sustained bilateral connection | Active C2 channel or live data exfiltration |
| `icmp.type == 8` shows "no response found" entries | Packets dropped rather than handshaked | Non-legitimate ICMP use |
| Non-uniform packet sizes during flood | Mixed payload sizes | Attacker is manipulating packets, not just overwhelming bandwidth |

### Evidence Against Smurf Attack

- Only one destination IP (`192.168.178.34`) — Smurf attacks target subnets, not single hosts
- `icmp.type == 0 && ip.dst == 192.168.178.34` returned no results — no data amplification, the defining characteristic of a Smurf attack

### Verdict

**This attack was ICMP tunneling, with ICMP flooding used as a calculated smokescreen.**

The 1,117-byte ICMP packet — nearly 20 times the standard size — is a strong IOC for tunneling. The **Stunnel** tool identifier embedded in the packet data is a smoking gun: Stunnel is a documented ICMP tunneling tool capable of acting as both a C2 channel and an exfiltration mechanism.

**The flooding was deliberate misdirection.** A defender focused on the flood would not check individual ICMP packet sizes, allowing the tunneling to proceed undetected. The flood also lacked the characteristics of a genuine DoS — no uniform packet sizes, no data amplification, and replies accounted for 50% of traffic (well below what a successful DoS would suppress). The 3-second delay before flooding began suggests a detection-triggered execution, indicating the attacker was already monitoring the network.

**Threat actor assessment:** This is not a noisy opportunistic attacker. The ICMP flood as a deliberate decoy, combined with pre-staged tunneling infrastructure, indicates an Advanced Persistent Threat (APT) actor or a deliberate intrusion by someone with prior knowledge of the network's monitoring capabilities. The attacker had enough access to evaluate defender behavior and craft a response to detection.

---

## Remediation

### Immediate Actions
- **Isolate affected hosts** — evidence of persistent access requires immediate isolation
- **Kill active tunnels** — block all DNS and ICMP traffic to/from affected hosts
- **Preserve PCAP files** as forensic evidence
- **Block non-existent TLD queries** (e.g., `.sea`, `.pirate`) at the DNS resolver
- **Enforce payload size limits** on UDP DNS packets

### Short-Term
- Implement **size inspection** for ICMP and UDP DNS traffic
- **Rate-limit ICMP** at the firewall level
- Restrict **outbound ICMP** at the network perimeter
- Audit all hosts for presence of **Stunnel** and **iodine** tooling
- **Reset credentials** on all affected systems
- Restrict internal hosts to **internal DNS resolvers only**
- Block all outbound **UDP/TCP port 53** except from authorized DNS servers
- Flag and inspect **NULL record type queries** — there is virtually no legitimate use for NULL records in a corporate environment

### Long-Term
- Deploy **Deep Packet Inspection (DPI)** at the perimeter
- Deploy a **SIEM with anomaly detection** rules for DNS and ICMP behavioral baselines
- Ensure proper **network segmentation** to limit lateral movement
- Conduct a full **incident response investigation**
- Implement **DNS over HTTPS or TLS** so all DNS traffic is inspectable and encrypted
- Establish a **threat hunting cadence** to validate DNS response integrity and detect forged responses
