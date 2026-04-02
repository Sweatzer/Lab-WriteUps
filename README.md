# Security Lab Writeups

Technical writeups from HackTheBox labs and vulnerability assessment coursework. Each writeup documents the full attack chain, tools used, and remediation recommendations mapped to relevant frameworks (MITRE ATT&CK, OWASP, CWE).

**Author:** Spencer Leach  
**Certifications:** CompTIA Security+  
**Focus Areas:** Network forensics, web exploitation, IDS/IPS evasion, session security

---

## Writeups

| Lab | Topics | Frameworks |
|-----|--------|------------|
| [IDS/IPS and Firewall Evasion](ids-ips-firewall-evasion.md) | Stealth scanning, source port spoofing, banner grabbing | CWE-200 |
| [Session Security — XSS, Open Redirect, Session Hijacking](session-security-xss-hijacking.md) | Stored XSS, cookie theft, open redirect chaining, PCAP analysis | OWASP Top 10, CWE-79/601/614/319, MITRE T1539/T1040 |
| [Network Traffic Analysis — DNS & ICMP Tunneling](network-traffic-analysis-dns-icmp-tunneling.md) | Wireshark, DNS tunneling (iodine), ICMP tunneling (Stunnel), C2 detection | MITRE T1071/T1095 |

---

## Tools Referenced

`nmap` · `netcat` · `whatweb` · `Wireshark` · `Burp Suite` · `PHP` · `Metasploit`

---

## Notes

Screenshots referenced in writeups are stored in the `images/` subfolder of each lab directory.
