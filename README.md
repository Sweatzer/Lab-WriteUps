# Security Lab Writeups

Technical writeups from HackTheBox labs, LetsDefend SOC investigations, and vulnerability assessment coursework. Each writeup documents the full attack chain, tools used, and remediation recommendations mapped to relevant frameworks (MITRE ATT&CK, OWASP, CWE).

**Author:** Spencer Leach  
**Certifications:** CompTIA Security+  
**Focus Areas:** Network forensics, SIEM-based alert triage, web exploitation, file upload bypass, SQL injection, Linux privilege escalation, IDS/IPS evasion, session security, footprinting & enumeration, Wi-Fi penetration testing, red team infrastructure & automation

---

## Pentest Writeups

### Web

| Lab | Topics | Frameworks |
|-----|--------|------------|
| [Session Security — XSS, Open Redirect, Session Hijacking](Pentest-Writeups/Web/session-security-xss-hijacking/session-security-xss-hijacking.md) | Stored XSS, cookie theft, open redirect chaining, PCAP analysis | OWASP Top 10, CWE-79/601/614/319, MITRE T1539/T1040 |
| [Web Attacks — IDOR & XXE Exploitation](Pentest-Writeups/Web/idor-xxe/web-attacks-idor-xxe.md) | IDOR enumeration, API token harvesting, password reset abuse, XXE injection, Burp Suite | OWASP Top 10, CWE-639/611 |
| [SQL Injection — Union-Based Extraction & Authentication Bypass](Pentest-Writeups/Web/sql-injection/sql-injection.md) | Boolean-based SQLi, UNION injection, INFORMATION_SCHEMA enumeration, password hash extraction, Burp Suite | OWASP Top 10, CWE-89, MITRE T1190 |
| [File Upload Attacks — Multi-Layer Bypass to RCE](Pentest-Writeups/Web/file-upload-attacks/file-upload-attacks-skills-assessment.md) | Extension blacklist/whitelist bypass, Content-Type fuzzing, MIME type spoofing, magic byte injection, XXE via SVG for source code exfiltration, PHP web shell deployment | OWASP 2025 A02/A05/A06, CWE-434/611, MITRE T1190/T1505.003/T1036.008, NIST 800-53 SI-10/CM-6/CM-7, CISA Web Shell CSI |

### Infrastructure

| Lab | Topics | Frameworks |
|-----|--------|------------|
| [Footprinting — HTB Academy Skills Assessment](Pentest-Writeups/Infrastructure/recon-and-enumeration/footprinting/footprinting.md) | FTP, SSH, NFS, SMB, SNMP, RDP, MySQL service chaining, credential harvesting, lateral movement | MITRE T1046/T1078/T1021/T1552/T1039 |
| [Linux Privilege Escalation — PwnKit (CVE-2021-4034)](Pentest-Writeups/Infrastructure/privilege-escalation/linux-privesc/Linux-PrivEsc-PwnKit-Skills-Assessment.md) | SUID enumeration, PolicyKit exploitation, SSH key persistence, post-exploitation analysis | MITRE T1548.001/T1098.004, CWE-787 |
| [Pivoting, Tunneling & Port Forwarding — HTB CPTS Skills Assessment](Pentest-Writeups/Infrastructure/pivoting/pivoting-skills-assessment.md) | Dual-homed web server foothold, Metasploit pivot with internal routing, SOCKS proxy for external tooling, lateral movement via RDP, Mimikatz credential dumping on pivoted host, reaching a DC share through a second internal subnet | MITRE T1090.001/T1021.001/T1021.002/T1003.001/T1018/T1078 |
| [IDS/IPS and Firewall Evasion](Pentest-Writeups/Infrastructure/evasion/ids-ips-firewall-evasion/ids-ips-firewall-evasion.md) | Stealth scanning, source port spoofing, banner grabbing | CWE-200 |
| [Wi-Fi MAC Filtering Bypass](Pentest-Writeups/Infrastructure/wifi-pentesting/wifi-mac-bypass-writeup.md) | WPA2 handshake capture, deauthentication attacks, offline password cracking, MAC address spoofing, dual-band exploitation | MITRE T1669/T1016.002/T1110.002/T1036/T1498, NIST SP 800-97/800-153 |
| [Pivoting, Tunneling & Port Forwarding — HTB CPTS Skills Assessment](Pentest-Writeups/Infrastructure/pivoting/pivoting-skills-assessment.md) | Dual-homed web server foothold, Metasploit pivot + `route add`, SOCKS5 proxy via `auxiliary/server/socks_proxy`, proxychains, RDP lateral movement, Mimikatz LSASS credential dumping, multi-subnet traversal to DC share | MITRE T1090.001/T1021.001/T1003.001/T1572/T1021.002 |

---

## Defensive Writeups

### SIEM Analysis

| Lab | Topics | Frameworks |
|-----|--------|------------|
| [SOC Alert: CVE-2024-24919 — Check Point Arbitrary File Read](Defensive-Writeups/SIEM-Analysis/CVE-2024-24919-Checkpoint-Arbitrary-File-Read.md) | Path traversal exploitation, SOC alert triage, VirusTotal/AbuseIPDB enrichment, firewall log analysis | MITRE T1190, CWE-22 |
| [SOC338: Lumma Stealer via ClickFix Phishing](Defensive-Writeups/SIEM-Analysis/lumma-stealer-clickfix-phishing.md) | Phishing analysis, ClickFix social engineering, mshta.exe proxy execution, C2 detection, host containment | MITRE T1566.002/T1204.001/T1059.001/T1218.005/T1071.001 |
| [SOC257: VPN Connection from Unauthorized Country](Defensive-Writeups/SIEM-Analysis/SOC257-VPN-Connection-Unauthorized-Country.md) | Unauthorized VPN access attempt, IP reputation analysis, MFA bypass failure, port scanning, credential compromise triage | MITRE T1595/T1133/T1621 |
| [SIEM Detection Engineering — Splunk & ELK Use Case Development](Defensive-Writeups/SIEM-Analysis/splunk-elk-siem-detection-engineering.md) | 8-lab detection engineering report: SPL/KQL authoring, alert tuning & suppression, Windows Security Event correlation, IIS log analysis for SQLi/XSS, Snort scan signatures, scripted netstat inputs, Sysmon + Winlogbeat → ELK pipeline, LSASS `0x1010` access detection, hash-based IoC triage with VirusTotal | MITRE T1110.001/T1190/T1189/T1046/T1059.001/T1105/T1003.001, CWE-89/79 |
| [SIEM Use Case Development — Splunk & ELK Detection Engineering](Defensive-Writeups/SIEM-Analysis/siem-use-case-development-splunk-elk.md) | Eight end-to-end detection labs: brute force correlation (4624/4625), IIS SQLi & XSS regex with urldecode, Snort-fed TCP/XMAS/FIN scan alerts, `netstat` scripted input for Telnet, Sysmon + Winlogbeat → ELK for PowerShell LotL, LSASS `GrantedAccess 0x1010` Mimikatz detection, Sysmon hash pivot to VirusTotal | MITRE T1110.001/T1110.004/T1190/T1595.002/T1046/T1059.001/T1105/T1003.001 |

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

## Real-Life Engagements

Security findings from real-world responsible disclosure engagements. All reports are based on publicly visible client-side source code only — no exploitation or unauthorized access was performed. Identifying details have been sanitized to protect client identity.

| Report | Topics | Frameworks |
|--------|--------|------------|
| [ASP.NET WebForms File Upload Module — Client-Side Source Review](real-life-engagements/security_findings.md) | Broken CAPTCHA, hardcoded identifiers, client-side-only validation, path traversal, DOM-based XSS, cross-window manipulation, ViewState tampering, CSRF | OWASP Top 10 A01/A03/A04/A05, CWE-352/22/79/434/611, MITRE T1190 |

---

## Personal Research

Independent research and analysis on offensive security concepts, infrastructure design, and operational tradecraft — compiled from industry sources, practitioner blogs, and hands-on experimentation.

| Topic | Description | Key Concepts |
|-------|-------------|--------------|
| [Solo Operator Red Team Infrastructure](personal-research/solo-red-team-infrastructure.md) | Practical guide to building autonomous, disposable attack infrastructure for solo pentesters — from Proxmox home lab to cloud-deployed operations | WireGuard tunnels, Terraform/Ansible IaC, VPS redirectors, C2 architecture, OPSEC, functional compartmentalization |
| [Building Your Own VPS — Self-Hosted Virtualization](personal-research/diy-vps-own-infrastructure.md) | Practical guide to replacing commercial VPS subscriptions with self-hosted Proxmox infrastructure on personal hardware — covering hardware selection, VM provisioning, Cloudflare Tunnel exposure, hardening, and cost analysis | Proxmox VE, KVM, LXC, ZFS, Cloudflare Tunnel, WireGuard, reverse proxy, NVMe storage |

---

## Tools Referenced

`nmap` · `netcat` · `whatweb` · `Wireshark` · `Burp Suite` · `PHP` · `Metasploit` · `pkexec` · `ssh-keygen` · `crackmapexec` · `smbclient` · `xfreerdp` · `onesixtyone` · `snmpwalk` · `mysql` · `curl` · `hashcat` · `mshta.exe` · `PowerShell` · `ffuf` · `printf` · `airmon-ng` · `airodump-ng` · `aireplay-ng` · `aircrack-ng` · `macchanger` · `wpa_supplicant` · `Terraform` · `Ansible` · `WireGuard` · `Proxmox` · `Chisel` · `Ligolo-ng` · `cloudflared` · `Traefik` · `Nginx Proxy Manager` · `Tailscale` · `Splunk` · `ELK / Kibana` · `Sysmon` · `Winlogbeat` · `Snort` · `Hydra` · `Mimikatz` · `proxychains` · `evil-winrm` · `VirusTotal`

---

## Notes

Screenshots referenced in writeups are stored in the `images/` subfolder of each lab directory.
