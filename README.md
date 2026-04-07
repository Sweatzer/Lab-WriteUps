# Security Lab Writeups

Technical writeups from HackTheBox labs and vulnerability assessment coursework. Each writeup documents the full attack chain, tools used, and remediation recommendations mapped to relevant frameworks (MITRE ATT&CK, OWASP, CWE).

**Author:** Spencer Leach  
**Certifications:** CompTIA Security+  
**Focus Areas:** Network forensics, web exploitation, SQL injection, Linux privilege escalation, IDS/IPS evasion, session security, footprinting & enumeration

---

## Writeups

### Recon & Enumeration

| Lab | Topics | Frameworks |
|-----|--------|------------|
| [Footprinting — HTB Academy Skills Assessment](Pentest%20Writeups/recon-and-enumeration/footprinting/footprinting.md) | FTP, SSH, NFS, SMB, SNMP, RDP, MySQL service chaining, credential harvesting, lateral movement | MITRE T1046/T1078/T1021/T1552/T1039 |

### Web Exploitation

| Lab | Topics | Frameworks |
|-----|--------|------------|
| [Session Security — XSS, Open Redirect, Session Hijacking](Pentest%20Writeups/web-exploitation/session-security-xss-hijacking/session-security-xss-hijacking.md) | Stored XSS, cookie theft, open redirect chaining, PCAP analysis | OWASP Top 10, CWE-79/601/614/319, MITRE T1539/T1040 |
| [Web Attacks — IDOR & XXE Exploitation](Pentest%20Writeups/web-exploitation/idor-xxe/web-attacks-idor-xxe.md) | IDOR enumeration, API token harvesting, password reset abuse, XXE injection, Burp Suite | OWASP Top 10, CWE-639/611 |
| [SQL Injection — Union-Based Extraction & Authentication Bypass](Pentest%20Writeups/web-exploitation/sql-injection/sql-injection.md) | Boolean-based SQLi, UNION injection, INFORMATION_SCHEMA enumeration, password hash extraction, Burp Suite | OWASP Top 10, CWE-89, MITRE T1190 |

### Privilege Escalation

| Lab | Topics | Frameworks |
|-----|--------|------------|
| [Linux Privilege Escalation — PwnKit (CVE-2021-4034)](Pentest%20Writeups/privilege-escalation/linux-privesc/Linux-PrivEsc-PwnKit-Skills-Assessment.md) | SUID enumeration, PolicyKit exploitation, SSH key persistence, post-exploitation analysis | MITRE T1548.001/T1098.004, CWE-787 |

### Evasion

| Lab | Topics | Frameworks |
|-----|--------|------------|
| [IDS/IPS and Firewall Evasion](Pentest%20Writeups/evasion/ids-ips-firewall-evasion/ids-ips-firewall-evasion.md) | Stealth scanning, source port spoofing, banner grabbing | CWE-200 |

### Defensive

| Lab | Topics | Frameworks |
|-----|--------|------------|
| [Network Traffic Analysis — DNS & ICMP Tunneling](Defensive%20Writeups/network-traffic-analysis-dns-icmp-tunneling.md) | Wireshark, DNS tunneling (iodine), ICMP tunneling (Stunnel), C2 detection | MITRE T1071/T1095 |

---

## Home Labs

Hands-on lab environments built from scratch to simulate real-world attack and defense scenarios. These labs focus on building infrastructure, deploying security tooling, and executing end-to-end attack chains — all on isolated local networks.

| Lab | Description | Tools & Tech |
|-----|-------------|-------------|
| [Lab 1 — Cybersecurity Detection & Attack Lab](Home%20Labs/Lab%201/cybersecurity-homelab-build.md) | Multi-VM environment with Wazuh SIEM, Suricata IDS, and DVWA for purple team exercises. Covers network architecture design in VirtualBox, dual-NIC pivot configuration, Suricata-to-Wazuh log integration, web app exploitation via command injection, UAC bypass with fodhelper.exe, and fileless PowerShell reverse shells. Includes full troubleshooting log for common VirtualBox networking and tooling issues. | VirtualBox, Wazuh, Suricata, DVWA, XAMPP, PowerShell, netcat |

---

## Tools Referenced

`nmap` · `netcat` · `whatweb` · `Wireshark` · `Burp Suite` · `PHP` · `Metasploit` · `pkexec` · `ssh-keygen` · `crackmapexec` · `smbclient` · `xfreerdp` · `onesixtyone` · `snmpwalk` · `mysql` · `curl` · `hashcat`

---

## Notes

Screenshots referenced in writeups are stored in the `images/` subfolder of each lab directory.
