# SOC338 — Lumma Stealer via ClickFix Phishing

**Platform:** LetsDefend | **Event ID:** 316 | **Severity:** Critical | **Date:** Mar 13, 2025

---

## Summary

A phishing email impersonating Microsoft was delivered to an internal user (Dylan), containing a link to a fake Windows 11 Pro upgrade page. The user clicked the link, triggering a ClickFix social engineering chain that led to Lumma Stealer execution via `mshta.exe` and PowerShell. C2 communication was confirmed. The host was contained and the malicious email was deleted.

---

## Alert Details

| Field | Value |
|---|---|
| Rule | SOC338 — Lumma Stealer — DLL Side-Loading via Click Fix Phishing |
| Source Address | `update@windows-update[.]site` |
| SMTP IP | `132.232.40[.]201` |
| Recipient | `dylan@letsdefend.io` |
| Subject | Upgrade your system to Windows 11 Pro for FREE |
| Device Action | Allowed |

> **Note on the rule name:** The alert references DLL side-loading, but no DLL side-loading was observed during investigation. The actual execution chain was ClickFix → PowerShell → mshta.exe → C2 callback. The rule name appears to be a misnomer or generalized detection label.

---

## Detection & Email Analysis

The email passed through the mail gateway without being blocked (Device Action: Allowed). Filtering Dylan's mailbox in Email Security confirmed delivery.

Immediate red flags:

- **Sender domain:** `windows-update[.]site` — not a Microsoft-owned domain.
- **Urgency language:** "Hurry," "This offer expires soon," fake countdown timer.
- **Spoofed branding:** Full Microsoft/Windows 11 Pro visual layout designed to look legitimate.

---

## Threat Intel

**VirusTotal** flagged `https://www.windows-update[.]site/` with 11/97 detections. Vendors categorized it as phishing (BitDefender, ESET, G-Data, Sophos, SOCRadar, CyRadar) and malware (Fortinet, Kaspersky, VIPRE). Lionic and Webroot flagged it as generically malicious.

**Any.run** sandbox simulation confirmed the site presents a fake Windows Update page with a ClickFix prompt — instructing the user to press `Win+R`, then `Ctrl+V` to paste and execute a pre-staged PowerShell command. This is the core social engineering trick: the clipboard is silently loaded with a malicious command, and the user is tricked into executing it themselves.

**Domain ownership note:** The domain `windows-update[.]site` is registered through a legitimate Russian domain registrar. This complicates any legal takedown or attribution efforts — the registrar is unlikely to cooperate with Western remediation requests, and the infrastructure can be cheaply replaced.

---

## Execution Chain

1. **Initial Access (Mar 13, 2025, 09:44 AM):** Dylan receives and opens the phishing email. Clicks the link to `https://www.windows-update[.]site/`.

2. **User Execution:** The fake update page uses the ClickFix technique — user is instructed to open Run dialog and paste a command from the clipboard. This launches PowerShell.

3. **PowerShell → mshta.exe:** Terminal history on Dylan's host shows PowerShell spawning `mshta.exe` at 23:26:19 the same day.

4. **C2 Callback:** `mshta.exe` (PID 7284) reaches out to `https://overcoatpassably[.]shop/Z8UZbPyVpGfdRS/maloy.mp4` — a Lumma Stealer C2 endpoint disguised as a media file. The target process command line spawns `conhost.exe` with a `-ForceV1` flag.

5. **Network confirmation:** Firewall logs show Dylan's host (`172.16.17.216`) connecting to `172.67.139.19` (C2 IP) on port 443 at 23:26:20, immediately after mshta.exe execution.

---

## Containment

- **Host isolated** via Endpoint Security. Hostname: Dylan, IP: `172.16.20.151`.
- **Phishing email deleted** from Dylan's mailbox via Email Security.
- Credentials should be assumed compromised — Lumma Stealer targets browser-stored credentials, cookies, crypto wallets, and session tokens.

---

## MITRE ATT&CK

| Tactic | Technique | Context |
|---|---|---|
| Initial Access | T1566.002 — Phishing: Spearphishing Link | Email with link to fake Windows Update site |
| Execution | T1204.001 — User Execution: Malicious Link | User clicked phishing link and followed ClickFix instructions |
| Execution | T1059.001 — Command and Scripting Interpreter: PowerShell | PowerShell executed via ClickFix Run dialog trick |
| Defense Evasion | T1218.005 — System Binary Proxy Execution: Mshta | `mshta.exe` used to fetch and execute remote payload |
| Command and Control | T1071.001 — Application Layer Protocol: Web Protocols | HTTPS C2 to `overcoatpassably[.]shop` on port 443 |

---

## IOCs

| Type | Value |
|---|---|
| URL (Phishing) | `https://www.windows-update[.]site/` |
| SMTP IP | `132.232.40[.]201` |
| URL (C2) | `https://overcoatpassably[.]shop/Z8UZbPyVpGfdRS/maloy.mp4` |
| IP (C2) | `172.67.139.19` |
| Process Hash (mshta.exe) | `15c80b5be235bf2a8c38291eb697a702c07dde087eb459e9ea46a2bee17c5f03` |

---

## Remediation

- Force credential reset for Dylan's account across all services.
- Audit for lateral movement from Dylan's host — check if any credentials stored in the browser were reused elsewhere.
- Block IOCs at the firewall/proxy level.
- Add `windows-update[.]site` and `overcoatpassably[.]shop` to domain blocklists.
- Review email gateway rules — this email should not have been allowed through. Implement stricter filtering on `.site` TLDs and look-alike Microsoft domains.
- User awareness training on ClickFix-style attacks specifically — these bypass traditional "don't click links" advice because the user is executing the payload manually.

---

## Lessons Learned

The ClickFix technique is effective because it shifts execution responsibility to the user, bypassing endpoint protections that would normally flag automated script execution. Traditional phishing awareness training focuses on not clicking links and not opening attachments — ClickFix exploits a gap in that training by making the user the execution vector. Security awareness programs need to specifically address this pattern: if a website is asking you to open Run, press keyboard shortcuts, or paste anything into a terminal, that's an attack.
