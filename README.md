# Security Lab Writeups

Technical writeups from SOC triage alerts, detection engineering labs, pentest coursework, home lab builds, and real-world security research. Each writeup documents the full investigation or attack chain, tools used, and remediation recommendations mapped to relevant frameworks (MITRE ATT&CK, OWASP, CWE).

**Author:** Spencer Leach
**Background:** Cybersecurity student at Pikes Peak State College
**Certifications:** CompTIA Security+
**Focus Areas:** SOC triage & detection engineering, network forensics, web exploitation, Active Directory and Linux privilege escalation, IDS/IPS evasion, session security, footprinting & enumeration

---

## Real-World Security Work

| Engagement | Scope | Findings |
|---|---|---|
| [ASP.NET WebForms File Upload Module — Client-Side Source Review](https://github.com/Sweatzer/Lab-WriteUps/blob/main/real-life-engagements/security_findings.md) | Responsible disclosure, client-side code review only, no exploitation attempted | 11 findings: non-functional reCAPTCHA, client-side-only validation, permissive extension whitelist, path traversal on download handler, DOM-based XSS via `controlID`, cross-window DOM manipulation without origin checks, potentially vulnerable ViewState, theoretical attack chain |

---

## Detection Engineering & SOC Triage

### SIEM Triage — LetsDefend Alerts

| Event | Alert / Rule | Topics | Frameworks |
|---|---|---|---|
| [SOC257](https://github.com/Sweatzer/Lab-WriteUps/blob/main/Defensive-Writeups/SIEM-Analysis/SOC257-VPN-Connection-Unauthorized-Country.md) | VPN Connection Detected from Unauthorized Country | Credential-stuffing VPN login from Vietnam, MFA bypass attempt via OTP brute, IP reputation triage (AbuseIPDB/VT), firewall + proxy + email correlation | MITRE T1595 / T1133 / T1621 |
| [SOC287](https://github.com/Sweatzer/Lab-WriteUps/blob/main/Defensive-Writeups/SIEM-Analysis/CVE-2024-24919-Checkpoint-Arbitrary-File-Read.md) | CVE-2024-24919 — Check Point Quantum Spark Arbitrary File Read | Path traversal exploitation of `/clients/MyCRL`, `/etc/passwd` and `/etc/shadow` exfiltration, two-IP attacker correlation, containment + credential rotation | MITRE T1190 / T1552.001 / T1005, CVSS 8.6 |
| [SOC338](https://github.com/Sweatzer/Lab-WriteUps/blob/main/Defensive-Writeups/SIEM-Analysis/lumma-stealer-clickfix-phishing.md) | Lumma Stealer via ClickFix Phishing | Phishing → ClickFix social engineering → PowerShell → `mshta.exe` → C2 callback, email/sandbox/network correlation, host isolation | MITRE T1566.002 / T1204.001 / T1059.001 / T1218.005 / T1071.001 |
| [Detection Engineering Lab](https://github.com/Sweatzer/Lab-WriteUps/blob/main/Defensive-Writeups/SIEM-Analysis/siem-use-case-development-splunk-elk.md) | Splunk & ELK Use Case Development | 8 end-to-end detections: brute force (4624/4625 correlation), SQLi, XSS, Snort scan detection, insecure port monitoring, PowerShell LotL, Mimikatz LSASS access (0x1010), Sysmon+VT hash pivot | MITRE T1110 / T1190 / T1595 / T1046 / T1059.001 / T1003.001 / T1105 |

### DFIR — HTB Sherlocks

| Sherlock | Scenario | Artifacts | Frameworks |
|---|---|---|---|
| [Brutus](https://github.com/Sweatzer/Lab-WriteUps/blob/main/Defensive-Writeups/DFIR-Sherlocks/brutus/brutus.md) | Linux SSH brute-force → interactive root compromise → backdoor account persistence (`cyberjunkie` added to sudo) → `/etc/shadow` dump → `linper` persistence toolkit download | `/var/log/auth.log`, `/var/log/wtmp` (parsed via Python utmp parser) | MITRE T1110.001 / T1078.003 / T1136.001 / T1098 / T1003.008 / T1105 |

### Network Forensics

| Lab | Topics | Frameworks |
|---|---|---|
| [Network Traffic Analysis — DNS & ICMP Tunneling](https://github.com/Sweatzer/Lab-WriteUps/blob/main/Defensive-Writeups/Network-Analysis/network-traffic-analysis-dns-icmp-tunneling.md) | Wireshark, DNS tunneling (iodine), ICMP tunneling (Stunnel), C2 detection | MITRE T1071 / T1095 |

---

## Home Labs

Hands-on lab environments built from scratch to simulate real-world attack and defense scenarios. Focus is on infrastructure design, security tooling deployment, and executing end-to-end attack chains on isolated networks.

| Lab | Description | Tools & Tech |
|---|---|---|
| [Lab 1 — Cybersecurity Detection & Attack Lab](https://github.com/Sweatzer/Lab-WriteUps/blob/main/Home-Labs/Lab-1/cybersecurity-homelab-build.md) | Multi-VM environment with Wazuh SIEM, Suricata IDS, and DVWA for purple team exercises. Network architecture design in VirtualBox, dual-NIC pivot configuration, Suricata-to-Wazuh log integration, web app exploitation via command injection, UAC bypass with `fodhelper.exe`, and fileless PowerShell reverse shells. Includes full troubleshooting log. | VirtualBox, Wazuh, Suricata, DVWA, XAMPP, PowerShell, netcat |

---

## Pentest Writeups

### HTB Machines — End-to-End CTFs

| Machine | Difficulty / OS | Attack Chain | Frameworks |
|---|---|---|---|
| [Optimum](https://github.com/Sweatzer/Lab-WriteUps/blob/main/Pentest-Writeups/HTB-Machines/optimum/optimum.md) | Easy / Windows Server 2012 R2 | HFS 2.3 RCE → x86 Meterpreter as `kostas` → autologon registry creds (WOW64 redirection bypass) → `local_exploit_suggester` → `ms16_032` → `NT AUTHORITY\SYSTEM` | MITRE T1190 / T1552.002 / T1068 / T1134, CVE-2014-6287, CVE-2016-0099 |
| [Cap](https://github.com/Sweatzer/Lab-WriteUps/blob/main/Pentest-Writeups/HTB-Machines/cap/cap.md) | Easy / Linux | IDOR on `/data/<id>` → `.pcap` with cleartext FTP creds → SSH as `nathan` → `getcap` enumeration → `cap_setuid` on `/usr/bin/python3.8` → root | MITRE T1190 / T1552.001 / T1040 / T1021.004 / T1548.001 |

### Recon & Enumeration

| Lab | Topics | Frameworks |
|---|---|---|
| [Footprinting — HTB Academy Skills Assessment](https://github.com/Sweatzer/Lab-WriteUps/blob/main/Pentest-Writeups/Infrastructure/recon-and-enumeration/footprinting/footprinting.md) | FTP, SSH, NFS, SMB, SNMP, RDP, MySQL service chaining, credential harvesting, lateral movement | MITRE T1046 / T1078 / T1021 / T1552 / T1039 |

### Web Exploitation

| Lab | Topics | Frameworks |
|---|---|---|
| [Session Security — XSS, Open Redirect, Session Hijacking](https://github.com/Sweatzer/Lab-WriteUps/blob/main/Pentest-Writeups/Web/session-security-xss-hijacking/session-security-xss-hijacking.md) | Stored XSS, cookie theft, open redirect chaining, PCAP analysis | OWASP Top 10, CWE-79/601/614/319, MITRE T1539/T1040 |
| [Web Attacks — IDOR & XXE Exploitation](https://github.com/Sweatzer/Lab-WriteUps/blob/main/Pentest-Writeups/Web/idor-xxe/web-attacks-idor-xxe.md) | IDOR enumeration, API token harvesting, password reset abuse, XXE injection, Burp Suite | OWASP Top 10, CWE-639/611 |
| [SQL Injection — Union-Based Extraction & Authentication Bypass](https://github.com/Sweatzer/Lab-WriteUps/blob/main/Pentest-Writeups/Web/sql-injection/sql-injection.md) | Boolean-based SQLi, UNION injection, INFORMATION_SCHEMA enumeration, password hash extraction, Burp Suite | OWASP Top 10, CWE-89, MITRE T1190 |

### Privilege Escalation

| Lab | Topics | Frameworks |
|---|---|---|
| [Linux Privilege Escalation — PwnKit (CVE-2021-4034)](https://github.com/Sweatzer/Lab-WriteUps/blob/main/Pentest-Writeups/Infrastructure/privilege-escalation/linux-privesc/Linux-PrivEsc-PwnKit-Skills-Assessment.md) | SUID enumeration, PolicyKit exploitation, SSH key persistence, post-exploitation analysis | MITRE T1548.001 / T1098.004, CWE-787 |

### Evasion

| Lab | Topics | Frameworks |
|---|---|---|
| [IDS/IPS and Firewall Evasion](https://github.com/Sweatzer/Lab-WriteUps/blob/main/Pentest-Writeups/Infrastructure/evasion/ids-ips-firewall-evasion/ids-ips-firewall-evasion.md) | Stealth scanning, source port spoofing, banner grabbing | CWE-200 |

---

## Personal Research

Self-directed writeups exploring infrastructure and operator topics outside structured coursework.

| Topic | Description |
|---|---|
| [Building Your Own VPS — Self-Hosted Virtualization](https://github.com/Sweatzer/Lab-WriteUps/blob/main/personal-research/diy-vps-own-infrastructure.md) | Practical guide to replacing commercial VPS subscriptions with a Proxmox-based home lab: hardware selection, hypervisor install, VM/container provisioning, networking, internet exposure, hardening, and cost analysis. |
| [Solo Operator Red Team Infrastructure](https://github.com/Sweatzer/Lab-WriteUps/blob/main/personal-research/solo-red-team-infrastructure.md) | End-to-end architecture for disposable, compartmentalized attack infrastructure: home lab foundation, cloud redirectors, self-hosted VPN tunnels, Terraform/Ansible automation, functional separation, and where Tor fits (and doesn't). |

---

## Tools Referenced

`nmap` · `netcat` · `whatweb` · `Wireshark` · `Burp Suite` · `Splunk` · `ELK Stack` · `Sysmon` · `Winlogbeat` · `Wazuh` · `Suricata` · `Snort` · `Hydra` · `Mimikatz` · `PowerShell` · `Metasploit` · `pkexec` · `ssh-keygen` · `crackmapexec` · `smbclient` · `xfreerdp` · `onesixtyone` · `snmpwalk` · `mysql` · `curl` · `hashcat` · `Proxmox VE` · `Terraform` · `Ansible`

---

## Notes

Screenshots referenced in writeups are stored in the `images/` subfolder of each lab directory. All real-world engagement content is sanitized to protect the identity of the application and its operator — no exploitation, testing, or unauthorized access was performed.
