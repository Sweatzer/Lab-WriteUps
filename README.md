# Security Lab Writeups

Technical writeups from HackTheBox labs and vulnerability assessment coursework. Each writeup documents the full attack chain, tools used, and remediation recommendations mapped to relevant frameworks (MITRE ATT&CK, OWASP, CWE).

**Author:** Spencer Leach  
**Certifications:** CompTIA Security+  
**Focus Areas:** Network forensics, web exploitation, Linux privilege escalation, IDS/IPS evasion, session security, footprinting & enumeration

---

## Writeups

| Lab | Topics | Frameworks |
|-----|--------|------------|
| [Footprinting — HTB Academy Skills Assessment](Pentest%20Writeups/footprinting/footprinting.md) | FTP, SSH, NFS, SMB, SNMP, RDP, MySQL service chaining, credential harvesting, lateral movement | MITRE T1046/T1078/T1021/T1552/T1039 |
| [Linux Privilege Escalation — PwnKit (CVE-2021-4034)](Pentest%20Writeups/linux-privesc/Linux-PrivEsc-PwnKit-Skills-Assessment.md) | SUID enumeration, PolicyKit exploitation, SSH key persistence, post-exploitation analysis | MITRE T1548.001/T1098.004, CWE-787 |
| [IDS/IPS and Firewall Evasion](Pentest%20Writeups/ids-ips-firewall-evasion/ids-ips-firewall-evasion.md) | Stealth scanning, source port spoofing, banner grabbing | CWE-200 |
| [Session Security — XSS, Open Redirect, Session Hijacking](Pentest%20Writeups/session-security-css-hijacking/session-security-xss-hijacking.md) | Stored XSS, cookie theft, open redirect chaining, PCAP analysis | OWASP Top 10, CWE-79/601/614/319, MITRE T1539/T1040 |
| [Network Traffic Analysis — DNS & ICMP Tunneling](Defensive%20Writeups/network-traffic-analysis-dns-icmp-tunneling.md) | Wireshark, DNS tunneling (iodine), ICMP tunneling (Stunnel), C2 detection | MITRE T1071/T1095 |

---

## Tools Referenced

`nmap` · `netcat` · `whatweb` · `Wireshark` · `Burp Suite` · `PHP` · `Metasploit` · `pkexec` · `ssh-keygen` · `crackmapexec` · `smbclient` · `xfreerdp` · `onesixtyone` · `snmpwalk` · `mysql`

---

## Notes

Screenshots referenced in writeups are stored in the `images/` subfolder of each lab directory.
