# SOC257 — VPN Connection Detected from Unauthorized Country

**Platform:** LetsDefend  
**Event ID:** 225  
**Rule:** SOC257 — VPN Connection Detected from Unauthorized Country  
**Severity:** Security Analyst  
**Date:** Feb 13, 2024, 02:04 AM  
**Verdict:** True Positive (Connection Attempt) — MFA Prevented Login

---

## Alert Summary

A VPN login attempt was detected from an unauthorized country targeting the user account `monica@letsdefend.io`. The source IP `113.161.158.12` originated from Vietnam (Hanoi), connecting to the VPN gateway at `https://vpn-letsdefend.io`. The attacker had valid credentials but was ultimately blocked by MFA controls.

| Field | Value |
|-------|-------|
| Source Address | `113.161.158.12` |
| Destination Address | `33.33.33.33` |
| Destination Hostname | Monica |
| Username | `monica@letsdefend.io` |
| URL | `https://vpn-letsdefend.io` |

---

## Detection & Verification

Searching Log Management for the attacker IP revealed both Firewall and Proxy log entries. The proxy logs confirmed VPN-related traffic — a POST request to `https://vpn-letsdefend.io/logon.html` at 02:01 AM returned HTTP 200, indicating the attacker successfully reached the login page and submitted credentials.

WHOIS lookup confirmed the IP belongs to Vietnam Posts and Telecommunications Group (VNPT), geolocated to Hanoi, Vietnam — an unauthorized country for this environment.

**Verdict at this stage:** True Positive for unauthorized VPN connection attempt. Further analysis required to determine whether authentication was fully successful.

---

## Analysis

### IP Reputation

The attacker IP was checked across three threat intelligence sources:

- **AbuseIPDB:** 1,246 reports from 456 sources, 100% abuse confidence score. Categories include Brute Force and SSH abuse. Most recent report was within minutes of the alert.
- **VirusTotal:** Flagged as malicious by 16/90 security vendors including CrowdSec, Fortinet, and Cyble.
- **LetsDefend Threat Intel:** Previously tagged for Brute Force activity (Dec 2023).

The IP has a well-established history of malicious activity across multiple reporting platforms.

### Initial Access — External Remote Services (T1133)

The attacker used the SSL VPN Service exposed at `https://vpn-letsdefend.io` to attempt authentication with Monica's credentials. The proxy log at 02:01 AM shows a successful POST to `logon.html` with HTTP 200 — confirming the attacker reached the authentication flow and submitted a valid password.

### MFA Bypass Attempt — Multi-Factor Authentication Request Generation (T1621)

The critical finding: the attacker entered the correct password but failed at the MFA step. Firewall logs at 02:01 AM show `Action: Incorrect OTP Code`, confirming the attacker could not provide a valid one-time password.

Email Security logs corroborate this — three OTP emails were sent to `monica@letsdefend.io` at 02:01, 02:02, and 02:03 AM, each containing a passcode and metadata (source IP, browser, location: Hanoi). The attacker attempted authentication three times, failing the OTP challenge each time.

The attacker had the password but lacked access to Monica's email or SMS to retrieve the OTP. MFA functioned as intended and prevented full account compromise.

### Reconnaissance — Active Scanning (T1595)

Additional Firewall logs revealed the attacker conducted port scanning against `33.33.33.33` prior to the VPN login attempt (starting ~01:56 AM). Requests were observed across a wide range of destination ports from varying source ports — consistent with network reconnaissance behavior preceding the credential attack.

---

## Containment

Although the attacker was blocked by MFA and no VPN session was established, Monica's account is considered **compromised** — the attacker possessed valid credentials. Containment actions:

- **Password reset** for `monica@letsdefend.io` (required — credentials are confirmed compromised)
- **No host isolation required** — the attacker never gained network access

---

## Lessons Learned

- Implement geo-based access restrictions on the firewall to block VPN connections from unauthorized countries
- Enforce unique password policies — the compromised credential likely originated from password reuse across platforms
- Establish strong password complexity requirements on all clients and servers
- Implement account lockout policies to limit brute-force and credential-stuffing attempts

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|--------|-----------|-----|
| Reconnaissance | Active Scanning | T1595 |
| Initial Access | External Remote Services | T1133 |
| Credential Access | Multi-Factor Authentication Request Generation | T1621 |

---

## Artifacts

| Field | Value |
|-------|-------|
| Attacker IP | `113.161.158.12` |
| Compromised User | `monica@letsdefend.io` |
| Internal IP | `172.16.17.163` |
| VPN Target | `https://vpn-letsdefend.io` |
| Geolocation | Hanoi, Vietnam |
| ISP | Vietnam Posts and Telecommunications Group (VNPT) |
