# INLANEFREIGHT — Penetration Test Report

### Framework Mapping, Severity Analysis, Remediation & Governance Review

> **Companion analysis to the technical writeup** *"INLANEFREIGHT — Attacking Enterprise Networks."* The technical writeup is the reproduction-and-evidence base; this document is the assessment deliverable layered on top of it: offensive-framework mapping, CVSS v4.0 severity scoring, framework-sourced remediation, a governance (GRC) gap analysis, and updated standard operating procedures.

---

## Document Control

| Field                     | Value                                                                                                      |
| ------------------------- | ---------------------------------------------------------------------------------------------------------- |
| **Report title**          | INLANEFREIGHT Internal/External Network Penetration Test — Assessment Report                               |
| **Engagement type**       | Black-box external → internal, full-scope (assumed-breach not required; achieved unauthenticated foothold) |
| **Target environment**    | `INLANEFREIGHT.LOCAL` — 3 subnets, 7 hosts (DMZ, internal `172.16.8.0/24`, management `172.16.9.0/24`)     |
| **Classification**        | *(mark per your handling policy — e.g., CUI//SP-PRVCY, Confidential, etc.)*                                |
| **Methodology standards** | PTES, NIST SP 800-115, OWASP WSTG/Top 10:2025                                                              |
| **Scoring standard**      | CVSS v4.0 (FIRST)                                                                                          |
| **Version / Date**        | 1.0 — *5/30/2026*                                                                                          |
| **Author**                | *Spencer Leach*                                                                                            |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Scope, Methodology & Frameworks](#2-scope-methodology--frameworks)
3. [Attack Narrative → MITRE ATT&CK Mapping](#3-attack-narrative--mitre-attck-mapping)
4. [Findings & Severity Register (CVSS v4.0)](#4-findings--severity-register-cvss-v40)
5. [Detailed Findings](#5-detailed-findings)
6. [Remediation Recommendations (Framework-Sourced)](#6-remediation-recommendations-framework-sourced)
7. [GRC Gap Analysis — What Went Wrong on the Governance Side](#7-grc-gap-analysis--what-went-wrong-on-the-governance-side)
8. [Updated Standard Operating Procedures (SOPs)](#8-updated-standard-operating-procedures-sops)
9. [References](#9-references)
10. [Appendix A — Control Crosswalk](#appendix-a--control-crosswalk)

---

## 1. Executive Summary

A full-scope penetration test of the INLANEFREIGHT environment achieved **complete domain compromise and management-subnet root** starting from a single unauthenticated, internet-facing host. The assessor required no zero-day exploits; every step relied on **misconfiguration, weak credential hygiene, excessive privilege, and missing detective controls** — i.e., failures that are governance and process problems first and technical problems second.

The kill chain progressed through three stages:

- **External → DMZ foothold.** Eight public virtual hosts each exposed a distinct web vulnerability class (IDOR, HTTP verb tampering → unrestricted upload, LFI via an unpatched WordPress plugin, SQL injection, stored XSS, SSRF, XXE, and authenticated command injection). A weak SSH credential and an OS command-injection path produced the first shell; cleartext credentials in readable audit logs and a `sudo NOPASSWD` `openssl` entry produced root.
- **Internal pivot → host compromise.** A Ligolo-ng tunnel exposed the internal subnet. A world-readable NFS export disclosed SQL connection strings; a DotNetNuke superuser enabled `xp_cmdshell`; `SeImpersonatePrivilege` and a misconfigured Sysax scheduled task each yielded `SYSTEM`. Plaintext credentials recovered from AutoLogon LSA Secrets, an unattended-install file, a NETLOGON script, and a fileshare backup script provided successive domain credentials.
- **Active Directory → total compromise.** Kerberoasting and a **three-link ACL abuse chain** (`mssqladm` *GenericWrite* → `TTIMMONS` *GenericAll* → `Server Admins`, which held the *DCSync* replication rights) led to **DCSync extraction of the `krbtgt` and Administrator hashes**. A second Ligolo pivot through the dual-homed DC reached the management subnet, where an unpatched `polkit` (PwnKit, CVE-2021-4034) granted root.

**Bottom line for leadership:** The environment had no single catastrophic flaw — it had a *systemic absence of credential management, configuration hardening, least-privilege governance, and security monitoring*. Any one control domain, implemented well, would have broken the chain.

### Severity at a glance

| Severity | Count | Representative findings |
| --- | --- | --- |
| **Critical** | 19 | Web RCE chain (unrestricted upload, command injection, DNN/`xp_cmdshell`); SQLi; SSRF; XXE; world-readable NFS leaking SA creds; SeImpersonate & Sysax privesc; AutoLogon LSA Secrets; hardcoded credentials; Kerberoasting; AD ACL chain → DCSync; systemic weak passwords; no LAPS; weak SSH foothold |
| **High** | 7 | DNS zone transfer, anonymous FTP, IDOR, LFI (CVE-2016-10956), stored XSS, NETLOGON cleartext credentials, PwnKit (CVE-2021-4034) |
| **Medium** | 2 | Open GitLab registration, insufficient logging/alerting |
| **Low/Info** | — | (folded into related findings) |

---

## 2. Scope, Methodology & Frameworks

### 2.1 Methodology standards

The engagement followed the **Penetration Testing Execution Standard (PTES)** lifecycle — pre-engagement, intelligence gathering, threat modeling, vulnerability analysis, exploitation, post-exploitation, and reporting [16] — and aligns to **NIST SP 800-115**, *Technical Guide to Information Security Testing and Assessment* [6], which is the federal/defense reference for assessment execution and is the methodology a CMMC/RMF assessor will expect to see cited.

### 2.2 Offensive frameworks (the attack side)

- **MITRE ATT&CK (Enterprise)** [9] — every action in the technical writeup is mapped to a tactic/technique in [Section 3](#3-attack-narrative--mitre-attck-mapping). ATT&CK is the lingua franca that lets a finding ("DCSync") translate directly into a detection requirement (Event ID 4662 / directory-replication monitoring).
- **Lockheed Martin Cyber Kill Chain** — used as the executive-level narrative spine (Recon → Weaponization/Delivery → Exploitation → Installation → C2 → Actions on Objectives).
- **OWASP Top 10:2025** [1] — used to classify the eight web findings. This is the **current** edition (finalized January 2026); note the structural changes that affect classification: **SSRF is now absorbed into A01 Broken Access Control**, **Injection moved to A05**, **Security Misconfiguration is now A02**, and **Software Supply Chain Failures is the new A03** [1].

### 2.3 Governance / GRC frameworks (the defense side)

| Framework | Role in this report | Reference |
| --- | --- | --- |
| **NIST CSF 2.0** | Organizing model for the governance gap analysis; uses the six Functions — **Govern, Identify, Protect, Detect, Respond, Recover** (Govern added in v2.0) | [3] |
| **NIST SP 800-53 Rev 5** | Authoritative control catalog for remediation mapping (control IDs such as AC-6, IA-5, CM-7, SI-2, AU-6) | [4] |
| **NIST SP 800-171 (Rev 3) / CMMC** | Defense-contractor lens for CUI environments; maps to the same control families and is the compliance backdrop most relevant if INLANEFREIGHT handles federal data | [5] |
| **CIS Critical Security Controls v8.1** | Prioritized, implementation-level safeguards (Controls 1–18) | [7] |
| **ISO/IEC 27001:2022 Annex A** | International control reference (themes: Organizational, People, Physical, Technological; 93 controls) | [8] |
| **CVSS v4.0** | Severity scoring (Base metrics; environmental refinement noted where relevant) | [2] |

---

## 3. Attack Narrative → MITRE ATT&CK Mapping

Each row ties an action from the technical writeup (the **Evidence** column) to its ATT&CK tactic and technique. This is the table a blue team would convert directly into detection coverage.

| # | Tactic | Technique (ID) | Evidence (writeup phase) |
| --- | --- | --- | --- |
| 1 | Reconnaissance | Gather Victim Network Info: DNS / **Zone Transfer** (T1590.002) | Phase 1 — `dig AXFR` succeeded |
| 2 | Reconnaissance | Active Scanning: Vuln/Wordlist Scanning (T1595.003) | Phase 1 — `ffuf` vhost fuzzing; Phase 2 service enum |
| 3 | Initial Access | Exploit Public-Facing Application (T1190) | Phase 3 — IDOR, upload, LFI, SQLi, XSS, SSRF, XXE, cmd injection |
| 4 | Initial Access | Valid Accounts (T1078) / Brute Force (T1110) | Foothold via weak SSH credential; `admin:12qwaszx` on monitoring vhost |
| 5 | Execution | Command & Scripting Interpreter (T1059); SQL Stored Procedures (T1505.001) | Phase 3 `socat` payload; Phase 7 `xp_cmdshell` + base64 PowerShell |
| 6 | Persistence | Server Software Component: Web Shell (T1505.003) | Phase 3 — PHP web shell on dev vhost |
| 7 | Privilege Escalation | Abuse Elevation Control: Sudo (T1548.003) | Phase 5 — `sudo openssl` (GTFOBins) → root file read [17] |
| 8 | Privilege Escalation | Access Token Manipulation (T1134) | Phase 7 — PrintSpoofer abusing `SeImpersonatePrivilege` → SYSTEM |
| 9 | Privilege Escalation | Scheduled Task/Job (T1053) | Phase 8 — Sysax triggered task running as LocalSystem |
| 10 | Privilege Escalation | Exploitation for Privilege Escalation (T1068) | Phase 10 — PwnKit, CVE-2021-4034 [13][14] |
| 11 | Defense Evasion | Impair Defenses: Disable/Modify Tools (T1562.001) | Phase 10 — `Set-MpPreference -DisableRealtimeMonitoring` |
| 12 | Credential Access | Unsecured Credentials: Credentials in Files (T1552.001) | `web.config`, `adum.vbs`, `SQL Express Backup.ps1`, `unattend.xml` [12] |
| 13 | Credential Access | OS Credential Dumping: SAM (T1003.002) | Phase 7 — `reg save HKLM\SAM` → `secretsdump LOCAL` |
| 14 | Credential Access | OS Credential Dumping: LSA Secrets (T1003.004) | AutoLogon `DefaultPassword` on DEV01 & MS01 |
| 15 | Credential Access | OS Credential Dumping: Cached Domain Creds (T1003.005) | DCC2 hash for `hporter` |
| 16 | Credential Access | Steal/Forge Kerberos Tickets: **Kerberoasting** (T1558.003) | Phase 8 — `GetUserSPNs -request`; Phase 9 — targeted SPN injection [11] |
| 17 | Credential Access | OS Credential Dumping: **DCSync** (T1003.006) | Phase 9 — `secretsdump -just-dc` for `krbtgt`/Administrator [10] |
| 18 | Credential Access | Brute Force: Password Cracking (T1110.002) | hashcat `-m 1000 / 13100` + rockyou + best64 |
| 19 | Discovery | Network Service Discovery (T1046); Network Share Discovery (T1135) | Internal `nmap`; `showmount`/`smbclient`/`spider_plus` |
| 20 | Lateral Movement | Remote Services: WinRM (T1021.006), RDP (T1021.001), SSH (T1021.004) | `evil-winrm`, `xfreerdp`, SSH to mngt01 |
| 21 | Lateral Movement | Use Alternate Auth Material: Pass-the-Hash (T1550.002) | `evil-winrm -H` with DA NT hash to DC01 |
| 22 | Lateral Movement | Account Manipulation (T1098) | `bloodyAD` ForceChangePassword on `ssmalls`; group/SPN additions |
| 23 | Command & Control | Protocol Tunneling / Proxy (T1572 / T1090) | Ligolo-ng agents #1 and #2 (double pivot) |
| 24 | Collection | Data from Network Shared Drive (T1039) | `Department Shares`, `IT$` exfiltration |

**Tactics exercised:** Reconnaissance, Initial Access, Execution, Persistence, Privilege Escalation, Defense Evasion, Credential Access, Discovery, Lateral Movement, Collection, and Command & Control — **11 of the 14 Enterprise tactics**, demonstrating end-to-end coverage of a realistic intrusion.

---

## 4. Findings & Severity Register (CVSS v4.0)

Scores below are **CVSS v4.0 Base-metric assessments** with vector strings shown for transparency [2]. For the two findings tied to published CVEs, the **official NVD/vendor score** is used and cited. Base scores reflect worst-case assumptions; environmental refinement (data criticality, network exposure) is noted where it would move a rating.

> **Severity bands (CVSS v4.0):** Critical 9.0–10.0 · High 7.0–8.9 · Medium 4.0–6.9 · Low 0.1–3.9

| ID       | Finding                                                                             | Host(s)          | CVSS v4.0 Vector                                         | Score | Severity          |
| -------- | ----------------------------------------------------------------------------------- | ---------------- | -------------------------------------------------------- | ----- | ----------------- |
| **F-01** | Weak SSH credential / no lockout (external foothold)                                | dmz01            | `AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:L/SI:L/SA:L` | 9.3   | **Critical**      |
| **F-02** | Unrestricted DNS zone transfer (AXFR) — topology disclosure                         | dmz01            | `AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:N/VA:N/SC:N/SI:N/SA:N` | 8.7   | High              |
| **F-03** | Anonymous FTP exposing sensitive files                                              | dmz01            | `AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:N/VA:N/SC:N/SI:N/SA:N` | 8.7   | High              |
| **F-04** | IDOR — sequential profile IDs (Broken Access Control)                               | careers vhost    | `AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:N/VA:N/SC:N/SI:N/SA:N` | 8.7   | High              |
| **F-05** | HTTP verb tampering → header auth bypass → **unrestricted file upload → RCE**       | dev vhost        | `AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:H/SI:H/SA:H` | 10.0  | **Critical**      |
| **F-06** | LFI via outdated Mail-Masta plugin — **CVE-2016-10956**                             | ir vhost         | NVD v3.1: `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`          | 7.5   | High [15]         |
| **F-07** | SQL injection — authentication-data disclosure                                      | status vhost     | `AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:N/SC:N/SI:N/SA:N` | 9.3   | **Critical**      |
| **F-08** | Stored XSS → admin session hijack                                                   | support vhost    | `AV:N/AC:L/AT:N/PR:N/UI:A/VC:H/VI:L/VA:N/SC:L/SI:N/SA:N` | 7.0   | High              |
| **F-09** | SSRF → arbitrary file read (now OWASP A01:2025)                                     | tracking vhost   | `AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:N/VA:N/SC:H/SI:N/SA:N` | 9.2   | **Critical**      |
| **F-10** | Open GitLab self-registration → data exposure                                       | gitlab vhost     | `AV:N/AC:L/AT:N/PR:N/UI:N/VC:L/VI:N/VA:N/SC:L/SI:N/SA:N` | 6.9   | Medium            |
| **F-11** | XXE injection → arbitrary file read                                                 | shopdev2 vhost   | `AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:N/VA:N/SC:H/SI:L/SA:N` | 9.2   | **Critical**      |
| **F-12** | Authenticated **OS command injection** + weak blacklist + weak admin password → RCE | monitoring vhost | `AV:N/AC:L/AT:N/PR:L/UI:N/VC:H/VI:H/VA:H/SC:H/SI:H/SA:H` | 9.4   | **Critical**      |
| **F-13** | Cleartext credentials in `adm`-readable audit/journal logs                          | dmz01            | `AV:L/AC:L/AT:N/PR:L/UI:N/VC:H/VI:H/VA:H/SC:H/SI:H/SA:H` | 9.3   | **Critical**      |
| **F-14** | Excessive `sudo` — `NOPASSWD: /usr/bin/openssl` → root file read (GTFOBins)         | dmz01            | `AV:L/AC:L/AT:N/PR:L/UI:N/VC:H/VI:H/VA:H/SC:H/SI:H/SA:H` | 9.3   | **Critical** [17] |
| **F-15** | World-readable NFS export (`everyone`) leaking SA SQL connection strings            | DEV01            | `AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:N/SC:H/SI:L/SA:N` | 9.9   | **Critical**      |
| **F-16** | DotNetNuke superuser → SQL console → **`xp_cmdshell` RCE**                          | DEV01            | `AV:N/AC:L/AT:N/PR:L/UI:N/VC:H/VI:H/VA:H/SC:H/SI:H/SA:H` | 9.4   | **Critical**      |
| **F-17** | `SeImpersonatePrivilege` on service account → PrintSpoofer → SYSTEM                 | DEV01            | `AV:L/AC:L/AT:N/PR:L/UI:N/VC:H/VI:H/VA:H/SC:H/SI:H/SA:H` | 9.3   | **Critical**      |
| **F-18** | AutoLogon cleartext `DefaultPassword` in LSA Secrets                                | DEV01, MS01      | `AV:L/AC:L/AT:N/PR:L/UI:N/VC:H/VI:H/VA:N/SC:H/SI:L/SA:N` | 9.2   | **Critical**      |
| **F-19** | Systemic weak / dictionary-crackable passwords                                      | domain-wide      | `AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:H/SI:H/SA:H` | 10    | **Critical**      |
| **F-20** | Plaintext credentials in NETLOGON script (`adum.vbs`)                               | DC01             | `AV:N/AC:L/AT:N/PR:L/UI:N/VC:H/VI:N/VA:N/SC:H/SI:L/SA:N` | 8.4   | High              |
| **F-21** | Kerberoastable service accounts with weak passwords                                 | DC01             | `AV:N/AC:L/AT:N/PR:L/UI:N/VC:H/VI:H/VA:N/SC:H/SI:L/SA:N` | 9.3   | **Critical** [11] |
| **F-22** | Hardcoded credentials in fileshare backup script                                    | DC01 share       | `AV:N/AC:L/AT:N/PR:L/UI:N/VC:H/VI:H/VA:N/SC:H/SI:L/SA:N` | 9.3   | **Critical**      |
| **F-23** | Cleartext local-admin password in `unattend.xml` (`C:\panther`)                     | MS01             | `AV:L/AC:L/AT:N/PR:L/UI:N/VC:H/VI:H/VA:N/SC:H/SI:L/SA:N` | 9.2   | **Critical**      |
| **F-24** | Sysax Automation scheduled-task misconfig → SYSTEM                                  | MS01             | `AV:L/AC:L/AT:N/PR:L/UI:N/VC:H/VI:H/VA:H/SC:H/SI:H/SA:H` | 9.3   | **Critical**      |
| **F-25** | **Dangerous AD ACL chain → DCSync → full domain compromise**                        | DC01 / domain    | `AV:N/AC:L/AT:N/PR:L/UI:N/VC:H/VI:H/VA:H/SC:H/SI:H/SA:H` | 9.4   | **Critical** [10] |
| **F-26** | No LAPS — local-admin password reuse / no rotation                                  | all Windows      | `AV:N/AC:L/AT:N/PR:L/UI:N/VC:H/VI:H/VA:N/SC:H/SI:L/SA:N` | 9.3   | **Critical**      |
| **F-27** | Missing patch — **PwnKit, CVE-2021-4034** (KEV-listed)                              | mngt01           | NVD v3.1: `AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H`          | 7.8   | High [13][14]     |
| **F-28** | Insufficient logging & alerting (loud actions undetected)                           | enterprise       | `AV:N/AC:L/AT:N/PR:N/UI:N/VC:L/VI:L/VA:N/SC:L/SI:L/SA:N` | 6.9   | Medium            |

\* **Caveat** - Many of these findings are particularly high mostly because of the nature of this being a lab environment. This means from a security perspective this network was essentially close to worst case scenario when as far as there being any sort of defensive framework in place that even small networks would implement by law. Also there is no metrics available for recovery, urgency, or other variables that might improve the criticality ratings. 

---

## 5. Detailed Findings

The four **Critical** findings and the most instructive **High** findings are detailed below. Full reproduction steps, commands, and tool output are in the technical writeup (referenced by phase).

### F-05 — Unrestricted File Upload via HTTP Verb Tampering (Critical, 10.0)

**OWASP:** A06:2025 Insecure Design + A02:2025 Security Misconfiguration. **ATT&CK:** T1190 → T1505.003.
**Description.** An `OPTIONS` response disclosed an over-permissive method set (`Allow: GET, POST, PUT, TRACK, OPTIONS`); a `TRACK` request leaked the trusted-IP header `X-Custom-IP-Authorization`. Spoofing that header reached an upload endpoint that enforced file type by `Content-Type` rather than content inspection, allowing a PHP web shell tagged `image/png` to execute (Phase 3, *dev vhost*).
**Impact.** Unauthenticated remote code execution on the web tier — the most direct path to foothold.
**Root cause.** Client-controlled trust (an IP header used for authorization) and content-type-based upload validation; both are design failures, not coding typos.

### F-12 — Authenticated OS Command Injection with Bypassable Blacklist (Critical, 9.4)

**OWASP:** A05:2025 Injection. **ATT&CK:** T1190 → T1059.
**Description.** The monitoring vhost's "connection test" passed user input to a server-side `ping` unescaped. The application blacklisted reverse-shell binaries (`nc`, `bash -i`, `python`, …) but **omitted `socat`**, which was used to obtain a stable shell (Phase 3). Access required the admin credential `admin:12qwaszx`, itself recovered from the SQLi dump (F-07) — two findings chaining into one compromise.
**Root cause.** Denylist input filtering (inherently incomplete) instead of allowlisting / parameterized execution; plus credential reuse across vhosts.

### F-16 — DotNetNuke Superuser → `xp_cmdshell` RCE (Critical, 9.4)

**OWASP:** A02:2025 Security Misconfiguration. **ATT&CK:** T1190 → T1505.001 → T1059.001.
**Description.** SA-equivalent SQL connection strings recovered from a world-readable `web.config` (F-15) granted DotNetNuke superuser access; the SQL console confirmed `sysadmin` and re-enabled `xp_cmdshell`, yielding a reverse shell as `nt service\mssql$sqlexpress` (Phase 7).
**Root cause.** Database service account running with `sysadmin`; `xp_cmdshell` re-enableable; secrets stored in a readable config on an open NFS export.

### F-25 — Active Directory ACL Abuse Chain → DCSync (Critical, 9.4)

**ATT&CK:** T1098 → T1558.003 → **T1003.006 (DCSync)** [10][11].
**Description.** A three-link chain — `mssqladm` **GenericWrite** over `TTIMMONS`, `TTIMMONS` **GenericAll** over the `Server Admins` group, and `Server Admins` holding the **`GetChanges` / `GetChangesAll` / `GetChangesInFilteredSet`** replication rights — allowed a low-privilege service account to escalate to domain replication and **DCSync the `krbtgt` and Administrator hashes** (Phase 9). The assessor injected a temporary SPN onto `TTIMMONS` (targeted Kerberoasting) to obtain its password, then nested group membership to reach the replication rights.
**Impact.** Full Active Directory compromise; `krbtgt` extraction enables Golden Ticket persistence surviving password resets.
**Root cause.** Excessive, unmonitored AD object ACLs — the single most damaging governance failure in the environment, and one **invisible to manual enumeration** (it required BloodHound graph analysis to surface).

### F-13 / F-14 — Credential Leak in Logs + `sudo openssl` → Root (Critical)

**ATT&CK:** T1552.001 → T1548.003 [12][17]. Cleartext credentials (`srvadm`) were readable to the web service account because it belonged to the `adm` (log-reader) group; `srvadm` then held `sudo NOPASSWD` on `/usr/bin/openssl`, a documented GTFOBins arbitrary-file-read primitive used to exfiltrate root's SSH key (Phase 4–5). Two independent least-privilege failures compounded into root.

### F-18 / F-20 / F-22 / F-23 — Plaintext Credentials in Files & Shares (Critical / High, recurring)

**OWASP:** A04:2025 Cryptographic Failures / A02:2025. **ATT&CK:** T1552.001 / T1003.004.
The **single largest finding category**: credentials recovered from AutoLogon LSA Secrets (DEV01, MS01), a NETLOGON script (`adum.vbs`), an IT-share PowerShell backup script (`backupadm`), and an unattended-install file (`unattend.xml`, MS01). Each is the same root cause — **static secrets stored in cleartext in locations readable below the privilege of the secret they protect**.

### F-27 — Unpatched PwnKit, CVE-2021-4034 (High, 7.8 — KEV)

**ATT&CK:** T1068. `policykit-1` on `mngt01` was below the fixed version, leaving the `pkexec` memory-corruption flaw exploitable for local root [13]. NVD scores this **7.8 (High)**; it is listed in the **CISA Known Exploited Vulnerabilities catalog** [14], which under BOD 22-01 obligates federal agencies to remediate on a fixed timeline — a relevant compliance hook for any defense-adjacent environment.

---

## 6. Remediation Recommendations (Framework-Sourced)

Each remediation theme is mapped to **NIST SP 800-53 Rev 5** controls [4], **CIS Controls v8.1** safeguards [7], and **ISO/IEC 27001:2022 Annex A** controls [8]. Defense environments should treat the same items as **NIST SP 800-171 / CMMC** requirements [5].

### R-1 — Credential Management & Secrets Hygiene
*Addresses F-13, F-18, F-19, F-20, F-21, F-22, F-23.*
- Eliminate static/embedded credentials from scripts, config files, unattend files, and shares; migrate to a vault / managed identities. Remove `unattend.xml` after OOBE and disable AutoLogon.
- Enforce length-based passphrases (≥ 16 chars) for users and **randomly generated 25+ char passwords for service accounts**; use **gMSA** where possible so secrets are non-extractable.
- **NIST 800-53:** `IA-5`, **`IA-5(7)` (no embedded unencrypted static authenticators)**, `IA-5(1)`, `AC-2`. **CIS:** 5.2, 5.4, 3.11. **ISO 27001:2022:** A.5.17, A.8.5, A.8.24.

### R-2 — Least Privilege & AD ACL Governance
*Addresses F-04, F-14, F-16, F-17, F-24, F-25, F-26.*
- Run periodic **BloodHound/PingCastle** ACL reviews; remove `GenericAll`/`GenericWrite`/`WriteDACL` and stray replication rights; restrict `DS-Replication-Get-Changes*` to DCs only.
- Remove `sudo NOPASSWD` on file-processing binaries (`openssl`, etc.); strip `SeImpersonatePrivilege` from interactive service contexts; deploy **Windows LAPS** for unique, rotated local-admin passwords [18].
- Database services must not run as `sysadmin`; disable `xp_cmdshell` and enforce by policy.
- **NIST 800-53:** `AC-6`, `AC-6(1)/(2)/(5)/(9)/(10)`, `AC-5`, `AC-2(7)`. **CIS:** 5.4, 6.8, 4.1, 4.7. **ISO 27001:2022:** A.8.2, A.8.3, A.5.15, A.5.18.

### R-3 — Secure Configuration & Hardening (Least Functionality)
*Addresses F-02, F-03, F-05, F-10, F-15, F-16.*
- Establish hardened baselines (CIS Benchmarks / DISA STIGs). Disable DNS zone transfer to untrusted hosts; disable anonymous FTP; lock NFS exports to specific hosts; remove unnecessary HTTP methods (`PUT`/`TRACK`); set GitLab registration to admin-approval.
- **NIST 800-53:** `CM-2`, `CM-6`, **`CM-7` / `CM-7(1)` (least functionality)**, `SC-20/21/22` (DNS). **CIS:** 4.1, 4.8, 12.x. **ISO 27001:2022:** A.8.9, A.8.27.

### R-4 — Secure Software Development & Input Validation
*Addresses F-04, F-05, F-06, F-07, F-08, F-09, F-11, F-12.*
- Adopt **parameterized queries** (SQLi), **allowlist input validation** and content-based file inspection (upload/command injection), **disable external entities** in XML parsers (XXE), **server-side authorization checks per object** (IDOR), **output encoding + CSP** (XSS), and **egress/destination allowlists** for server-side fetches (SSRF). Validate against **OWASP ASVS**.
- **NIST 800-53:** **`SI-10` (input validation)**, `SA-11`, `SA-15`, `SI-15`. **CIS:** 16.x (Application Software Security). **ISO 27001:2022:** A.8.25, A.8.26, A.8.28.

### R-5 — Vulnerability & Patch Management
*Addresses F-06, F-27.*
- Stand up authenticated vulnerability scanning with risk-based SLAs; treat **CISA KEV** entries (e.g., CVE-2021-4034) as expedited. Patch/upgrade the Mail-Masta plugin (CVE-2016-10956) and `polkit`.
- **NIST 800-53:** **`SI-2` (flaw remediation)**, `RA-5`, `CA-8`. **CIS:** 7.x (Continuous Vulnerability Management), 18.x (Penetration Testing). **ISO 27001:2022:** A.8.8.

### R-6 — Logging, Monitoring & Detection
*Addresses F-13, F-25, F-28; also Defense Evasion (T1562.001).*
- Centralize logs (SIEM); build ATT&CK-aligned detections for the techniques in [Section 3](#3-attack-narrative--mitre-attck-mapping) — especially **DCSync (directory-replication from non-DC source)**, **Kerberoasting (4769 anomalies)**, **mass password resets / group changes**, and **AV tamper events**. Protect audit logs so they cannot store or expose secrets in cleartext.
- **NIST 800-53:** `AU-2`, **`AU-6` (review/analysis)**, **`AU-9` (protect audit info)**, `SI-4`, `IR-4`. **CIS:** 8.x (Audit Log Management), 13.x (Network Monitoring & Defense). **ISO 27001:2022:** A.8.15, A.8.16, A.5.7.

### R-7 — Network Segmentation & Boundary Control
*Addresses lateral movement / double pivot (T1090, T1572).*
- Enforce segmentation between DMZ, internal, and management subnets; the dual-NIC DC bridging `172.16.8.0/24` ↔ `172.16.9.0/24` collapsed a trust boundary and enabled the second pivot. Restrict tooling egress and management-plane access (jump hosts, PAW).
- **NIST 800-53:** `SC-7`, `AC-17`, `AC-4`. **CIS:** 12.x, 13.4. **ISO 27001:2022:** A.8.22, A.8.20.

---

## 7. GRC Gap Analysis — What Went Wrong on the Governance Side

The technical findings are symptoms. Underneath them sit **program-level governance failures** — the absence of policies, ownership, and processes that *should* have prevented the conditions the assessor exploited. This section is organized by **NIST CSF 2.0 Function** [3], because that is how a board, an auditor, or a CMMC assessor will expect the deficiencies framed.

### GOVERN (GV) — the root layer
The Govern function is new in CSF 2.0 and is the layer most clearly missing here. There is no evidence of:
- **A credential/secrets management policy** — the recurrence of plaintext secrets in four distinct mechanisms (LSA Secrets, NETLOGON, fileshare scripts, unattend files) indicates *no organizational standard exists*, not that one control failed. (GV.PO, GV.RR)
- **Defined roles and accountability** for AD privilege administration — the unmonitored ACL chain (F-25) implies no owner for entitlement governance. (GV.RR)
- **Risk-management strategy / risk acceptance discipline** — "convenient" choices (anonymous FTP, AutoLogon, `sudo` on `openssl`, `socat` left on a host) were never weighed as risk decisions. (GV.RM)
- **Supply-chain governance** — an outdated WordPress plugin (CVE-2016-10956) maps directly to the new OWASP **A03:2025 Software Supply Chain Failures** and to CSF **GV.SC**.

### IDENTIFY (ID)
- **No reliable asset/software inventory** (CIS 1 & 2): an internet-exposed host on port 1337, eight unmanaged vhosts, and an unpatched management box suggest assets and software are not tracked. (ID.AM)
- **No data classification / data-flow mapping**: SA connection strings and PII-bearing profiles sat unprotected because their sensitivity was never established. (ID.AM-07)
- **No continuous risk assessment / pentest cadence** (CA-8, CIS 18): these issues persisted because nobody was looking. (ID.RA)

### PROTECT (PR)
- **Identity & access management is the dominant gap** — weak passwords domain-wide, no MFA, no LAPS, no least privilege, no service-account hardening. (PR.AA, PR.AC)
- **No configuration-hardening baseline** — every "Security Misconfiguration" finding ties here. (PR.PS)
- **No secure-SDLC requirements** — eight injection/access-control bug classes in production web apps. (PR.PS, PR.IR)

### DETECT (DE)
- **The most damaging blind spot.** Password resets, group-membership changes, SPN injection, and DCSync are **loud, well-known detections** — yet the engagement notes (Findings & Lessons Learned) that these actions "would be caught instantly outside a lab" *only because no monitoring existed here*. Absent **DE.CM / DE.AE** coverage, the organization would not have known a domain takeover occurred.

### RESPOND / RECOVER (RS / RC)
- No evidence of incident-response capability, `krbtgt` double-reset runbooks, or recovery planning. After a `krbtgt` compromise, recovery specifically requires a **dual `krbtgt` rotation** and likely AD forest recovery planning — neither of which appears to exist. (RS.MA, RC.RP)

### Compliance framing (for a CUI / defense context)
If INLANEFREIGHT processes federal CUI, the above gaps are not merely best-practice misses — they are **failed NIST SP 800-171 / CMMC Level 2 objectives** across families 3.1 (Access Control), 3.3 (Audit & Accountability), 3.4 (Configuration Management), 3.5 (Identification & Authentication), 3.11 (Risk Assessment), 3.12 (Security Assessment), and 3.14 (System & Information Integrity) [5]. A C3PAO assessment would not certify this environment.

---

## 8. Updated Standard Operating Procedures (SOPs)

>These are concrete, finding-driven procedures the organization should adopt and assign owners to. Each cites the findings it closes and the governing control(s).

### SOP-01 — Secrets & Credential Management Procedure
**Closes:** F-13, F-18, F-19, F-20, F-21, F-22, F-23 · **Controls:** IA-5(7), AC-2, CIS 5, ISO A.8.5
1. No credential may be stored in cleartext in source, config files, scripts, unattend files, fileshares, or logs. All secrets live in an approved vault; applications retrieve at runtime.
2. Disable AutoLogon enterprise-wide; where unattended provisioning is required, inject secrets at deploy time and **delete `unattend.xml`/`sysprep` artifacts** as a post-build step.
3. Service accounts use 25+ char random passwords or **gMSA**; rotate on a defined cadence and on personnel change.
4. Quarterly automated sweep for secrets-in-files (e.g., secret-scanning across shares, SYSVOL/NETLOGON, build artifacts); remediate within SLA.

### SOP-02 — Privileged Access & AD Entitlement Review Procedure
**Closes:** F-14, F-16, F-17, F-24, F-25, F-26 · **Controls:** AC-6, AC-5, AC-2(7), CIS 6, ISO A.8.2
1. **Monthly** ACL review using BloodHound/PingCastle; any `GenericAll`/`GenericWrite`/`WriteDACL`/`WriteOwner` on groups or Tier-0 objects requires documented justification or removal.
2. Replication rights (`DS-Replication-Get-Changes*`) restricted to domain controllers; alert on any other principal holding them.
3. No `sudo NOPASSWD` on interpreters or file-processing binaries; review `/etc/sudoers` against GTFOBins quarterly.
4. Remove `SeImpersonate`/`SeAssignPrimaryToken` from contexts that don't require it; deploy Windows LAPS to all member servers/workstations [18].
5. DB engines run under least-privilege accounts; `xp_cmdshell` disabled and change-controlled.

### SOP-03 — System Hardening & Configuration Baseline Procedure
**Closes:** F-02, F-03, F-05, F-10, F-15 · **Controls:** CM-2, CM-6, CM-7, CIS 4, ISO A.8.9
1. All systems built from a documented baseline (CIS Benchmark / DISA STIG); deviations are change-controlled exceptions with expiry.
2. Standard "deny by default" service posture: no anonymous FTP, no unrestricted DNS AXFR, NFS exports host-scoped (never `everyone`), web servers limited to required HTTP methods, application self-registration off by default.
3. New-build checklist signed off before a host enters production; drift detection runs continuously.

### SOP-04 — Secure Development & Web Application Validation Procedure
**Closes:** F-04, F-05, F-06, F-07, F-08, F-09, F-11, F-12 · **Controls:** SI-10, SA-11/15, CIS 16, ISO A.8.28
1. All input validated by **allowlist**; all DB access **parameterized**; XML parsers configured with external entities **disabled**; server-side fetches restricted to a destination **allowlist**.
2. Authorization enforced **server-side, per object** (no trust in client-supplied IDs or headers); output encoding + CSP mandatory.
3. SAST/DAST in CI; pre-release testing against **OWASP ASVS** and the **OWASP Top 10:2025** categories; no release with open High/Critical web findings.

### SOP-05 — Vulnerability & Patch Management Procedure
**Closes:** F-06, F-27 · **Controls:** SI-2, RA-5, CIS 7, ISO A.8.8
1. Authenticated vulnerability scans on a fixed cadence; results triaged by risk.
2. Remediation SLAs: **CISA KEV** items and Critical → expedited (e.g., 72 h / 7 days); High → 30 days; Medium → 90 days.
3. Third-party/plugin components tracked in an SBOM and patched as part of the supply-chain program (OWASP A03:2025 / CSF GV.SC).

### SOP-06 — Logging, Monitoring & Detection Procedure
**Closes:** F-13, F-25, F-28 · **Controls:** AU-2/6/9, SI-4, CIS 8 & 13, ISO A.8.15/16
1. All hosts ship logs to a central SIEM with tamper-resistant retention; audit logs must never contain cleartext secrets.
2. Maintain a detection library mapped to MITRE ATT&CK; minimum coverage for: DCSync (replication from non-DC), Kerberoasting (4769 anomalies / weak-cipher TGS requests), mass password resets and privileged group changes, AV/EDR tamper, and new local-admin creation.
3. Detection content is tested against red-team/pentest activity; gaps feed back into engineering each cycle.

### SOP-07 — Incident Response & AD Recovery Procedure
**Closes:** RS/RC gaps · **Controls:** IR-4/8, CIS 17, ISO A.5.24–A.5.30
1. Maintain IR runbooks including a **`krbtgt` double-reset** procedure and AD forest-recovery plan for confirmed domain compromise.
2. Define segmentation/containment steps for cross-subnet pivot scenarios (isolate dual-homed hosts; revoke pivot routes).
3. Tabletop the "external web foothold → DA" scenario annually using this report as the reference attack.

---

## 9. References

1. OWASP Foundation — *OWASP Top 10:2025* (current edition, finalized Jan 2026). https://owasp.org/Top10/2025/
2. FIRST — *Common Vulnerability Scoring System v4.0 Specification* (current; released Nov 2023). https://www.first.org/cvss/v4.0/
3. NIST — *Cybersecurity Framework (CSF) 2.0*, NIST CSWP 29 (Feb 2024). https://nvlpubs.nist.gov/nistpubs/CSWP/NIST.CSWP.29.pdf
4. NIST — *SP 800-53 Rev. 5, Security and Privacy Controls for Information Systems and Organizations.* https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final
5. NIST — *SP 800-171 (Protecting CUI)* / CMMC program reference. https://csrc.nist.gov/pubs/sp/800/171/r3/final
6. NIST — *SP 800-115, Technical Guide to Information Security Testing and Assessment.* https://csrc.nist.gov/pubs/sp/800/115/final
7. Center for Internet Security — *CIS Critical Security Controls v8.1.* https://www.cisecurity.org/controls
8. ISO/IEC — *27001:2022, Information security management systems.* https://www.iso.org/standard/27001
9. MITRE — *ATT&CK for Enterprise.* https://attack.mitre.org/
10. MITRE ATT&CK — *OS Credential Dumping: DCSync (T1003.006).* https://attack.mitre.org/techniques/T1003/006/
11. MITRE ATT&CK — *Steal or Forge Kerberos Tickets: Kerberoasting (T1558.003).* https://attack.mitre.org/techniques/T1558/003/
12. MITRE ATT&CK — *Unsecured Credentials: Credentials In Files (T1552.001).* https://attack.mitre.org/techniques/T1552/001/
13. Qualys — *PwnKit: Local Privilege Escalation in polkit's pkexec (CVE-2021-4034)* (CVSS 7.8). https://blog.qualys.com/vulnerabilities-threat-research/2022/01/25/pwnkit-local-privilege-escalation-vulnerability-discovered-in-polkits-pkexec-cve-2021-4034
14. CISA — *Known Exploited Vulnerabilities Catalog.* https://www.cisa.gov/known-exploited-vulnerabilities-catalog
15. NVD — *CVE-2016-10956 (Mail-Masta WordPress LFI), CVSS 7.5.* https://nvd.nist.gov/vuln/detail/CVE-2016-10956
16. The Penetration Testing Execution Standard (PTES). http://www.pentest-standard.org/
17. GTFOBins — *openssl.* https://gtfobins.github.io/gtfobins/openssl/
18. Microsoft — *Windows Local Administrator Password Solution (LAPS).* https://learn.microsoft.com/windows-server/identity/laps/laps-overview

---

## Appendix A — Control Crosswalk

| Remediation theme | NIST CSF 2.0 | NIST 800-53 r5 | CIS v8.1 | ISO 27001:2022 | NIST 800-171 |
| --- | --- | --- | --- | --- | --- |
| Credential/secrets mgmt | PR.AA | IA-5, IA-5(7), AC-2 | 5 | A.5.17, A.8.5 | 3.5 |
| Least privilege / AD ACLs | PR.AA, GV.RR | AC-6, AC-5, AC-2(7) | 6 | A.8.2, A.8.3 | 3.1 |
| Secure configuration | PR.PS | CM-2, CM-6, CM-7 | 4 | A.8.9 | 3.4 |
| Secure SDLC / input validation | PR.PS | SI-10, SA-11/15 | 16 | A.8.25, A.8.28 | 3.14 |
| Vulnerability & patch mgmt | ID.RA, PR.PS | SI-2, RA-5, CA-8 | 7, 18 | A.8.8 | 3.11, 3.12 |
| Logging / monitoring / detection | DE.CM, DE.AE | AU-2/6/9, SI-4 | 8, 13 | A.8.15, A.8.16 | 3.3 |
| Segmentation / boundary | PR.IR | SC-7, AC-4, AC-17 | 12, 13 | A.8.20, A.8.22 | 3.13 |
| Supply chain (plugins/components) | GV.SC | SR-3, SA-9 | 7, 15 | A.5.19–A.5.21 | 3.12 |
| Incident response / recovery | RS, RC | IR-4, IR-8, CP-10 | 17 | A.5.24–A.5.30 | 3.6 |

---

*Prepared as a companion assessment deliverable to the INLANEFREIGHT technical writeup. Severity scores are CVSS v4.0 Base-metric assessments (or cited official CVE scores) and should be refined with environmental metrics against the organization's data-criticality ratings before risk acceptance.*