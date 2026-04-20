# SIEM Use Case Development — Splunk & ELK Detection Engineering

Eight end-to-end SOC detection labs across Splunk and ELK Stack. Each lab covers the full detection engineering lifecycle: identifying the threat, mapping it to a data source, writing the detection query, configuring a real-time alert with suppression and severity, and validating against a live attack from Kali Linux.

**Platform:** Splunk Enterprise, ELK (Elasticsearch / Logstash / Kibana on Security Onion)
**Log Sources:** Windows Security Event Log, IIS, Snort IDS, Sysmon, `netstat` scripted input
**Attack Tooling:** Hydra, nmap (`-sT` / `-sX` / `-sF`), Mimikatz, PowerShell download cradles, SQLi / XSS payloads
**Target:** WinServer2012 (`10.10.10.12` / `192.168.1.4`)

---

## Contents

| # | Lab | Platform | Data Source |
|---|-----|----------|-------------|
| 1 | [Brute Force Detection via Windows Security Events](#lab-1--brute-force-detection-via-windows-security-events) | Splunk | WinEventLog:Security (4624/4625) |
| 2 | [SQL Injection Detection on IIS](#lab-2--sql-injection-detection-on-iis) | Splunk | IIS `cs_uri_query` |
| 3 | [Cross-Site Scripting (XSS) Detection on IIS](#lab-3--cross-site-scripting-xss-detection-on-iis) | Splunk | IIS logs |
| 4 | [Network Scanning Detection (Snort IDS)](#lab-4--network-scanning-detection-snort-ids) | Splunk | Snort IDS alerts |
| 5 | [Monitoring Insecure Ports and Services (Telnet)](#lab-5--monitoring-insecure-ports-and-services-telnet) | Splunk | `netstat -ano` scripted input |
| 6 | [ELK — Monitoring PowerShell Internet Connections](#lab-6--elk--monitoring-powershell-internet-connections) | ELK | Sysmon EID 3 via Winlogbeat |
| 7 | [ELK — Detecting Credential Dumping (Mimikatz)](#lab-7--elk--detecting-credential-dumping-mimikatz) | ELK | Sysmon EID 10 (ProcessAccess) |
| 8 | [ELK — Malware Detection via Sysmon + VirusTotal](#lab-8--elk--malware-detection-via-sysmon--virustotal) | ELK | Sysmon EID 1 (hashes) |

---

## Common Environment

| Machine | Role / Credentials |
|---|---|
| WinServer2012 | Attack target (FTP / IIS / RDP) — `Administrator` / `Pa$$w0rd` |
| SIEM1 | Splunk SIEM host (`http://localhost:8000`) — `admin` / `admin@123` |
| Security Onion | ELK stack — Logstash `:5044`, Elasticsearch `:9200`, Kibana `:5601` — `martin` / `martin@123` |
| Kali Linux | Attacker machine — `root` / `toor` |

---

## Lab 1 — Brute Force Detection via Windows Security Events

### Overview

Splunk use case detecting brute force login attempts against a Windows server by correlating failed and successful logins within short time windows. Failed login telemetry (Event ID 4625) paired with a successful login (Event ID 4624) following multiple failures is a high-fidelity indicator of a successful brute force attempt.

### Detection Query

```spl
host=WinServer2012 source=WinEventLog:Security (EventCode=4624 OR EventCode=4625)
```

The query bins events into 5-minute windows and counts total attempts, failures (`4625`), and successes (`4624`) per `Account_Name`. A `where` clause filters for patterns with **≥5 total attempts, ≥1 success, and ≥2 failures** — the signal of failures followed by a successful login.

### Alert Configuration

- Real-time schedule, 5-minute evaluation window
- Trigger when results > 0
- Throttle suppression on `Account_Name` to prevent alert flooding from a single compromised account
- Severity: **High**

### Attack Simulation

From Kali, Hydra was used against the FTP service on WinServer2012:

```bash
hydra -L /root/Wordlist/userlist.txt -P /root/Wordlist/pass.txt ftp://10.10.10.12
```

Hydra iterated through username/password combinations at a high request rate, generated a burst of `4625` events, and eventually logged a valid pair (producing a `4624`).

### Verification

Under **Activity → Triggered Alerts**, the brute force alert fired on the first run. The result set correctly displayed `Account_Name`, the 5-minute window, and `Attempts`/`Failed`/`Success` counts.

### Key Takeaways

- Correlating `4625` + `4624` is essential — monitoring failures alone produces noise and misses the critical signal of a success following repeated failures.
- Per-account throttle suppression prevents alert queue flooding while still capturing the initial trigger.
- Default Hydra behavior against FTP is loud. Low-and-slow attacks would require wider time windows and lower thresholds — tradeoff between detection sensitivity and false positive rate.

---

## Lab 2 — SQL Injection Detection on IIS

### Overview

Splunk use case detecting SQL injection attempts against the LuxuryTreats ASP.NET application (`http://www.luxurytreats.com`) running on WinServer2012 IIS. IIS logs are forwarded via Splunk Universal Forwarder; detection is performed against the URL-decoded query string.

### Detection Query

The query targets IIS log entries from WinServer2012 and applies `urldecode` to the `cs_uri_query` field **before** regex matching. The regex covers both URL-encoded and literal forms of common SQLi primitives:

- Single quote — `%27` and `'`
- `UNION SELECT` statements
- Boolean-based payloads (`OR 1=1`)
- SQL comment syntax — `--` and `/*`

### Alert Configuration

- Real-time alert titled **"SQL Injection Alert"**
- Trigger when results > 0
- Throttle on `c_ip` for **30 minutes** to prevent flooding from automated scanners
- Severity: **Medium**

### Attack Simulation

From Kali, Firefox was used to browse the LuxuryTreats application. The `OrderDetail.aspx` page exposed an `Id` parameter in the URL. A boolean-based SQLi payload appended to the parameter caused the application to return all user records — confirming the parameter was vulnerable.

### Verification

The alert appeared under **Triggered Alerts**. The result set showed the URI query string containing the payload, the Kali source IP, the user agent, and the timestamp.

### Key Takeaways

- **URL-decoding before regex matching is mandatory** — without it, `%27`-encoded payloads evade detection.
- Suppression on `c_ip` for 30 minutes captures the first occurrence while blocking repeat noise from automated scanners.
- Signature-based detection can be evaded by obfuscated payloads — behavioral analytics should be layered on top in production.

---

## Lab 3 — Cross-Site Scripting (XSS) Detection on IIS

### Overview

Splunk use case detecting XSS attempts against the LuxuryTreats application. Same log source as Lab 2 (IIS logs from WinServer2012), different detection patterns.

### Detection Query

Regex matches both plaintext and URL-encoded variants of script tags:

- Plaintext — `Javascript`, `Alert`
- URL-encoded — `%3CSCRIPT` (for `<SCRIPT`), `3C%2Fscript` (for `</script`)

### Alert Configuration

- Real-time alert with throttle suppression on `c_ip` for 30 minutes
- Severity: **Medium** (attempt-based signal — escalation to High warranted on confirmed execution)

### Attack Simulation

A JavaScript `alert()` payload was submitted through the LuxuryTreats contact form. The payload executed in the browser, confirming a reflected/stored XSS condition.

### Verification

Alert fired within the expected window under **Triggered Alerts**. Log entries tied to the attack were present in the result set.

### Key Takeaways

- Both plaintext and URL-encoded variants of script tags must be matched — web servers frequently encode special characters in log entries.
- Medium severity is appropriate for attempt detection; escalate to High on confirmed payload execution.
- Obfuscated or alternatively-encoded payloads would evade these specific search terms — layered analytics remain necessary.

---

## Lab 4 — Network Scanning Detection (Snort IDS)

### Overview

Three Splunk real-time alerts covering Snort-detected scan types (TCP, XMAS, FIN) against WinServer2012. Snort alerts are ingested via the Splunk Universal Forwarder on the IDS host and surfaced in SIEM1 for analyst triage.

### Detection Queries

| Scan Type | Splunk Query |
|---|---|
| TCP | `host=WinServer2012 source=ids 10.10.10.12 "TCP Scan Attempted"` |
| XMAS | `host=WinServer2012 source=ids 10.10.10.12 "<XMAS signature string>"` |
| FIN | `host=WinServer2012 source=ids 10.10.10.12 "<FIN signature string>"` |

Each was saved as a real-time alert with throttle suppression on the source IP to prevent flood amplification from a single aggressive scan.

### Attack Simulation

Three nmap scans from Kali against WinServer2012 (`10.10.10.12`):

```bash
nmap -sT 10.10.10.12   # TCP Connect
nmap -sX 10.10.10.12   # XMAS
nmap -sF 10.10.10.12   # FIN
```

XMAS and FIN scans returned `open|filtered` on most ports — expected behavior for Windows hosts due to TCP stack implementation differences from the RFC 793 probing logic these scans rely on.

### Verification

All three alerts fired under **Triggered Alerts**. End-to-end pipeline — from attack execution to SIEM alerting — functioned with no missed detections.

### Cleanup

All three alerts were disabled via **Settings → Searches, Reports, and Alerts**. Triggered alert entries were cleared. Logged out of Splunk.

### Key Takeaways

- Throttle suppression on source IP is critical — without it a single aggressive nmap scan would generate a flood of duplicate alerts.
- Snort signature-based detection only flags what the rule set defines; coverage depends entirely on the enabled signatures.
- Windows TCP stack behavior causes XMAS/FIN scans to return `open|filtered` — itself a useful OS-fingerprinting indicator during investigations.

---

## Lab 5 — Monitoring Insecure Ports and Services (Telnet)

### Overview

Splunk use case for detecting insecure services listening on a monitored Windows host. Implemented via a scripted input running `netstat -ano` on a short polling interval, forwarded to Splunk and alerted on when port 23 (Telnet) is found in a LISTENING state.

### Implementation

**On WinServer2012:**

1. Created `watch.bat` in the Splunk Universal Forwarder's `bin` directory containing:
   ```
   Netstat -ano
   ```
2. Configured `inputs.conf` to execute the script on a **10-second polling interval** as a scripted input.

**Detection Query (SIEM1):**

The Splunk search used `multikv` to parse the columnar `netstat` output, filtered for port 23 in `LISTENING` state, and extracted the Local Address field to identify the binding.

### Alert Configuration

- Real-time alert titled **"The Telnet port has been found opened."**
- Trigger when results > 0

### Validation

The Telnet service on WinServer2012 was started from the Services console. Within the 10-second polling window, the alert fired under **Triggered Alerts** with no delay or missed triggers.

### Cleanup

Alert disabled, triggered entries cleared, Telnet service stopped.

### Key Takeaways

- Scripted inputs are a flexible bridge for ingesting data from tools that don't produce log files natively.
- `multikv` is essential for parsing columnar text output from command-line tools.
- Port 23 transmits authentication and session data in plaintext — any exposure is a significant finding.
- 10-second polling combined with real-time evaluation provides near-immediate detection.

---

## Lab 6 — ELK — Monitoring PowerShell Internet Connections

### Overview

ELK detection for PowerShell commands that connect to the internet to download and execute remote scripts — the "download cradle" technique common in fileless malware and Living-off-the-Land (LotL) attacks. Sysmon is deployed on WinServer2012 for enriched process and network telemetry, Winlogbeat ships those events to Logstash on Security Onion, and detection is performed in Kibana.

### Environment

| Component | Details |
|---|---|
| WinServer2012 | `192.168.1.4` — `Administrator` / `Pa$$w0rd` |
| Security Onion (ELK) | `192.168.1.95` — `martin` / `martin@123` (Kibana) |
| Services | Logstash `:5044`, Elasticsearch `:9200`, Kibana `:5601` |

### Deployment

**Sysmon on WinServer2012:** Copied `Sysmon.zip` from `E:\SOC-Tools\Module 04` to `C:\`, extracted, and installed from an elevated prompt with the configuration XML to enable process creation, network connection, and related events.

**Winlogbeat on WinServer2012:** Configured to monitor the `Microsoft-Windows-Sysmon/Operational` event log channel and forward to `192.168.1.95:5044`. Configuration tested successfully; service started.

### Attack Simulation

Opened a PowerShell console and executed a download cradle combining flags commonly associated with malicious use:

- `-nop` — no profile
- `-w hidden` — hidden window
- `-exec bypass` — execution policy bypass
- `IEX` — `Invoke-Expression` on downloaded content

The command made an outbound HTTP request, simulating fileless malware behavior. PowerShell returned the response body, confirming the connection.

### Detection in Kibana

Searched Discover for `event_id:3` (Sysmon Network Connection). The expected event was present with the correct fields:

| Field | Value |
|---|---|
| `event_id` | 3 |
| `SourceImage` | `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe` |
| `SourceIp` | `192.168.1.4` |
| `DestinationIp` | (Google's resolved IP) |
| `HostSourceName` | WinServer2012 |

### Key Takeaways

- Sysmon Event ID 3 captures outbound connections initiated by any process — invaluable for LotL detection.
- Behavioral indicators like `-nop`, `-w hidden`, `-exec bypass`, and `IEX` in PowerShell command lines constitute high-confidence detection signals.
- Sysmon + Winlogbeat + ELK provides a complete telemetry pipeline from endpoint to analyst console.

---

## Lab 7 — ELK — Detecting Credential Dumping (Mimikatz)

### Overview

ELK use case for detecting credential dumping via LSASS access. The detection watches for `ProcessAccess` events (Sysmon Event ID 10) targeting `lsass.exe` with the `GrantedAccess` value `0x1010` — the specific combination of rights Mimikatz requests by default.

### Attack Simulation

**Deploying Mimikatz on WinServer2012:** Copied `mimikatz_trunk.Zip` from `E:\SOC-Tools\Module 04` to `C:\`, extracted, and dismissed the Windows Security pop-up triggered by the extraction.

**Executing Mimikatz:** From `C:\mimikatz_trunk\x64`:

```
privilege::debug
log Userdetails.log
sekurlsa::logonpasswords
```

`privilege::debug` elevated to `SeDebugPrivilege` (required for LSASS access). `log Userdetails.log` began persisting output to disk. `sekurlsa::logonpasswords` dumped credential material from LSASS memory to both the console and the log file.

### Detection in Kibana

KQL query:

```
event_id:10 AND event_data.GrantedAccess: 0x1010
```

Expanded the first matching event:

| Field | Value |
|---|---|
| `event_id` | 10 |
| `event_data.TargetImage` | `C:\Windows\System32\lsass.exe` |
| `event_data.GrantedAccess` | `0x1010` |
| `event_data.SourceImage` | `C:\mimikatz_trunk\x64\mimikatz.exe` |

The event confirmed `mimikatz.exe` opened a handle to `lsass.exe` with the `0x1010` access mask.

### Key Takeaways

- Sysmon Event ID 10 with `GrantedAccess 0x1010` targeting `lsass.exe` is a high-fidelity credential dumping indicator.
- False positives are possible from legitimate processes (AV engines, EDR agents) that query LSASS — production deployments should include a whitelist of known-good `SourceImage` values.
- `0x1010` is Mimikatz's default access mask, making it an effective and low-noise detection signature. Operators aware of detection can alter the mask, so this should not be the only LSASS-targeted detection in place.

---

## Lab 8 — ELK — Malware Detection via Sysmon + VirusTotal

### Overview

ELK use case that pivots from Sysmon's embedded file hash data (process creation events) to VirusTotal for threat intelligence enrichment. Demonstrates a safe, low-risk triage workflow: hash-only lookup never uploads the binary itself.

### Deployment

On WinServer2012, copied the `wikiworm` folder from `E:\SOC-Tools\Module 05 Enhance Incident Detection with Threat Intelligence` to `C:\`. Navigated to `C:\wikiworm` and executed `wikiworm.exe` (dismissed the resulting pop-up).

### Kibana Pivot

1. Launched Kibana and logged in (`martin` / `martin@123`).
2. Searched for `wikiworm.exe` in Discover. Sysmon events for the executable appeared.
3. Added filter: `event_data.Image is C:\Wikiworm\wikiworm.exe`.
4. Expanded the event, located the `event_data.Hashes` field, and copied the `MD5=` value (between the `=` and the comma delimiter).

### VirusTotal Enrichment

Pasted the MD5 hash into the VirusTotal search field. VT returned a full report confirming the hash was associated with a known malicious file:

- Detection ratios across multiple AV engines
- Vendor-specific malware classifications
- File metadata
- Behavioral tags

### Key Takeaways

- Sysmon process creation events (Event ID 1) include hash values — stable, content-based IOCs that survive filename and path changes.
- Hash-only lookups are passive and low-risk. Submitting a hash to VirusTotal does **not** upload the file itself, making this a safe first-pass triage technique suitable for sensitive environments.
- ELK filtering on `event_data.Image` is an effective pivot point for isolating specific executable events from high-volume Sysmon data.
- This workflow is repeatable and scalable to any unknown executable observed in Sysmon logs.

---

## Results Summary

All eight detection pipelines functioned as designed with no significant troubleshooting required. Each lab demonstrated the complete detection engineering lifecycle — threat identification → data source mapping → query construction → real-time alert configuration with appropriate suppression → live attack validation → cleanup.

---

## MITRE ATT&CK Coverage

| Lab | Technique(s) |
|-----|-------------|
| 1. Brute Force Detection | T1110.001 (Password Guessing), T1110.004 (Credential Stuffing) |
| 2. SQL Injection | T1190 (Exploit Public-Facing Application) |
| 3. XSS | T1190 |
| 4. Network Scanning | T1595.002 (Vulnerability Scanning), T1046 (Network Service Discovery) |
| 5. Insecure Port Monitoring | Hardening / posture (no offensive mapping) |
| 6. PowerShell LotL | T1059.001 (PowerShell), T1105 (Ingress Tool Transfer) |
| 7. Mimikatz / LSASS | T1003.001 (LSASS Memory) |
| 8. Hash Pivot / VT | T1204.002 (Malicious File) — detective control for post-execution triage |

---

## Overall Key Takeaways

- **Correlation beats single-event detection.** The brute force use case only becomes useful when `4624` and `4625` are correlated; either alone is noise.
- **Decoding happens before matching.** URL-decoding in Lab 2/3, `multikv` parsing in Lab 5, and hash extraction in Lab 8 all reflect the same principle: raw log text is rarely where the signal lives.
- **Throttling is not optional.** Every real-time alert configured used suppression on the appropriate high-cardinality field (`Account_Name`, `c_ip`, source IP) to keep the queue survivable.
- **Signature-based detection is a floor, not a ceiling.** Labs 2, 3, and 4 all flagged specific payloads or signatures; any of them can be evaded by a determined attacker. Behavioral analytics and anomaly detection should layer on top.
- **Sysmon is a force multiplier for ELK.** Three of the four ELK labs rely on Sysmon-specific event IDs (3, 10, 1) that stock Windows logging does not produce at the same fidelity.
