# Security Lab Writeups

Technical writeups from HackTheBox labs, LetsDefend SOC investigations, and vulnerability assessment coursework. Each writeup documents the full attack chain, tools used, and remediation recommendations mapped to relevant frameworks (MITRE ATT&CK, OWASP, CWE).

**Author:** Spencer Leach  
**Certifications:** CompTIA Security+  
**Focus Areas:** Network forensics, SIEM-based alert triage, web exploitation, SQL injection, Linux privilege escalation, IDS/IPS evasion, session security, footprinting & enumeration

---

## Pentest Writeups

### Web

| Lab | Topics | Frameworks |
|-----|--------|------------|
| [Session Security — XSS, Open Redirect, Session Hijacking](Pentest-Writeups/Web/session-security-xss-hijacking/session-security-xss-hijacking.md) | Stored XSS, cookie theft, open redirect chaining, PCAP analysis | OWASP Top 10, CWE-79/601/614/319, MITRE T1539/T1040 |
| [Web Attacks — IDOR & XXE Exploitation](Pentest-Writeups/Web/idor-xxe/web-attacks-idor-xxe.md) | IDOR enumeration, API token harvesting, password reset abuse, XXE injection, Burp Suite | OWASP Top 10, CWE-639/611 |
| [SQL Injection — Union-Based Extraction & Authentication Bypass](Pentest-Writeups/Web/sql-injection/sql-injection.md) | Boolean-based SQLi, UNION injection, INFORMATION_SCHEMA enumeration, password hash extraction, Burp Suite | OWASP Top 10, CWE-89, MITRE T1190 |

### Infrastructure

| Lab | Topics | Frameworks |
|-----|--------|------------|
| [Footprinting — HTB Academy Skills Assessment](Pentest-Writeups/Infrastructure/recon-and-enumeration/footprinting/footprinting.md) | FTP, SSH, NFS, SMB, SNMP, RDP, MySQL service chaining, credential harvesting, lateral movement | MITRE T1046/T1078/T1021/T1552/T1039 |
| [Linux Privilege Escalation — PwnKit (CVE-2021-4034)](Pentest-Writeups/Infrastructure/privilege-escalation/linux-privesc/Linux-PrivEsc-PwnKit-Skills-Assessment.md) | SUID enumeration, PolicyKit exploitation, SSH key persistence, post-exploitation analysis | MITRE T1548.001/T1098.004, CWE-787 |
| [IDS/IPS and Firewall Evasion](Pentest-Writeups/Infrastructure/evasion/ids-ips-firewall-evasion/ids-ips-firewall-evasion.md) | Stealth scanning, source port spoofing, banner grabbing | CWE-200 |

---

## Defensive Writeups

### SIEM Analysis

| Lab | Topics | Frameworks |
|-----|--------|------------|
| [SOC Alert: CVE-2024-24919 — Check Point Arbitrary File Read](Defensive-Writeups/SIEM-Analysis/CVE-2024-24919-Checkpoint-Arbitrary-File-Read.md) | Path traversal exploitation, SOC alert triage, VirusTotal/AbuseIPDB enrichment, firewall log analysis | MITRE T1190, CWE-22 |
| [SOC338: Lumma Stealer via ClickFix Phishing](Defensive-Writeups/SIEM-Analysis/lumma-stealer-clickfix-phishing.md) | Phishing analysis, ClickFix social engineering, mshta.exe proxy execution, C2 detection, host containment | MITRE T1566.002/T1204.001/T1059.001/T1218.005/T1071.001 |

### Network Analysis

| Lab | Topics | Frameworks |
|-----|--------|------------|
| [Network Traffic Analysis — DNS & ICMP Tunneling](Defensive-Writeups/Network-Analysis/network-traffic-analysis-dns-icmp-tunneling.md) | Wireshark, DNS tunneling (iodine), ICMP tunneling (Stunnel), C2 detection | MITRE T1071/T1095 |

---

## Home Labs

Hands-on lab environments built from scratch to simulate real-world attack and defense scenarios. These labs focus on building infrastructure, deploying security tooling, and executing end-to-end attack chains — all on isolated local networks.

| Lab | Description | Tools & Tech |
|-----|-------------|-------------|
| [Lab 1 — Cybersecurity Detection & Attack Lab](Home-Labs/Lab-1/cybersecurity-homelab-build.md) | Multi-VM environment with Wazuh SIEM, Suricata IDS, and DVWA for purple team exercises. Covers network architecture design in VirtualBox, dual-NIC pivot configuration, Suricata-to-Wazuh log integration, web app exploitation via command injection, UAC bypass with fodhelper.exe, and fileless PowerShell reverse shells. Includes full troubleshooting log for common VirtualBox networking and tooling issues. | VirtualBox, Wazuh, Suricata, DVWA, XAMPP, PowerShell, netcat |

---

## Tools Referenced

`nmap` · `netcat` · `whatweb` · `Wireshark` · `Burp Suite` · `PHP` · `Metasploit` · `pkexec` · `ssh-keygen` · `crackmapexec` · `smbclient` · `xfreerdp` · `onesixtyone` · `snmpwalk` · `mysql` · `curl` · `hashcat` · `mshta.exe` · `PowerShell`

---

## Notes

Screenshots referenced in writeups are stored in the `images/` subfolder of each lab directory.
