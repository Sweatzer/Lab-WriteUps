# ⭐ Featured — Full Enterprise Compromise + GRC Assessment

> **The flagship piece in this repo.** A complete external-to-domain-admin kill chain across an enterprise network, paired with a professional-grade governance assessment that scores, remediates, and root-causes every finding against current industry frameworks. This is the work that best represents both halves of the skill set: **breaking in, and explaining what it means to a business.**

---

## Why this one matters

Most writeups stop at "I got root." This one goes the full distance a real engagement requires — two documents, two audiences:

| Document | Audience | What it demonstrates |
|---|---|---|
| **[Technical Writeup](./INLANEFREIGHT_Lab_Writeup.md)** | Red team / operators | Full reproducible kill chain — every command, pivot, and decision |
| **[GRC Assessment Report](./INLANEFREIGHT_GRC_Assessment_Report.md)** | CISO / auditors / blue team | Severity scoring, framework-mapped remediation, governance gap analysis, and SOPs |

---

## The attack, in one breath

**One unauthenticated internet-facing host → full domain compromise → root on an isolated management subnet.** No zero-days — every step was misconfiguration, weak credentials, excessive privilege, or missing monitoring.

```
External recon → 8 web vhosts, 8 distinct bug classes → DMZ foothold
   → audit-log cred leak + sudo openssl → root → Ligolo pivot
   → NFS leak → DotNetNuke → xp_cmdshell → SeImpersonate → SYSTEM
   → AutoLogon/LSA/unattend/NETLOGON cred harvesting → domain creds
   → Kerberoast → BloodHound ACL chain → DCSync (krbtgt + Administrator)
   → double pivot through dual-homed DC → PwnKit (CVE-2021-4034) → root
```

### By the numbers
- **3 subnets · 7 hosts · 11 of 14 MITRE ATT&CK tactics** exercised end to end
- **8 web vulnerability classes** in one environment: IDOR · file upload via HTTP verb tampering · LFI · SQLi · stored XSS · SSRF · XXE · authenticated command injection
- **A 3-link AD ACL chain** (`GenericWrite` → `GenericAll` → replication rights) turned one low-priv service account into **DCSync** — invisible to manual enumeration, surfaced with BloodHound
- **2 CVEs** chained: CVE-2016-10956 (Mail-Masta LFI) and CVE-2021-4034 (PwnKit, CISA KEV-listed)

---

## What makes the GRC report stand out

This is the half that's rare in a student portfolio. The assessment takes the raw attack and turns it into a board-ready deliverable:

- **CVSS v4.0** severity register with full vector strings for all 28 findings
- **MITRE ATT&CK** mapping of every action to a detection requirement
- **OWASP Top 10:2025** classification (the current edition — SSRF now folded into A01, Injection at A05)
- **Framework-sourced remediation** cross-walked to **NIST SP 800-53 Rev 5**, **CIS Controls v8.1**, **ISO/IEC 27001:2022**, and **NIST SP 800-171 / CMMC**
- **Governance gap analysis** structured on **NIST CSF 2.0** (Govern · Identify · Protect · Detect · Respond · Recover)
- **Seven updated SOPs**, each tied to the findings it closes and the controlling framework reference

> The throughline: every technical finding is traced back to the *governance failure* that allowed it — the difference between reporting a symptom and diagnosing the disease.

---

## Skills on display

`Active Directory attacks` · `BloodHound ACL analysis` · `DCSync / Kerberoasting` · `web exploitation (8 classes)` · `Linux + Windows privilege escalation` · `multi-hop pivoting (Ligolo-ng)` · `CVSS v4.0 scoring` · `NIST CSF 2.0 / 800-53 / 800-171 / CMMC` · `OWASP Top 10:2025` · `MITRE ATT&CK mapping` · `remediation & SOP authoring`

---

*Lab environment: HTB Academy — Attacking Enterprise Networks. Assessment report authored as an original companion deliverable.*
