# Home Lab Writeup — Phase 1: Remote Access Foundations

**Project framing:** A hands-on learning lab built by progressively increasing complexity until hitting natural walls, then working through them. The throughline of this phase: *reach my lab from anywhere without opening a single inbound port.*

---

## Network Topology

The lab lives behind a **double-NAT** setup, which is both a constraint and the central design fact everything else works around:

- **lucy** — a shared upstream home router. Not under my control, and not something I want to depend on for port forwarding or availability.
- **misty** — a GL.iNet MT3000 (OpenWrt) sitting behind lucy. This is the controlled boundary. Personal devices live on misty's private subnet, **192.168.8.0/24** (misty at 192.168.8.1).
- Everything I care about hangs off misty's LAN, so misty is the natural always-on control point for the whole lab.

The double-NAT means nothing inside is reachable from the outside by default — which is exactly the posture I want. The work below is about getting *controlled* remote reach without giving that up.

### Hardware Inventory

| Name | What it is | Notes |
|------|-----------|-------|
| **mothership** | Windows desktop | 1TB NVMe SSD (OS), 3TB Seagate SMR HDD (bulk/data drive) |
| **misty** | GL.iNet MT3000 router | OpenWrt; the lab's perimeter and control center |
| **Raspberry Pi** | Kali Linux | Reflashed onto a new SD card; SSH key-only |
| **laptop** | General use / test client | Tailscale client |
| **Kali VM** | VirtualBox guest on mothership | Future remote-reachable target |

---

## 1. Remote Access: OpenVPN → Tailscale

### The decision
The first instinct was OpenVPN with port forwarding. Working through it, that's the wrong fit for this network:

- It requires an **inbound** listening port — the opposite of the minimize-inbound-exposure principle.
- Port forwarding would have to happen on **lucy**, the upstream router I don't fully control.
- That makes availability dependent on shared infrastructure I can't rely on.

**Tailscale** wins because its connection model is **outbound-only**. Each node dials out to coordinate, so there's nothing to forward on lucy and nothing listening on the WAN. It also sidesteps the double-NAT entirely instead of fighting it.

### What was done
- Installed Tailscale on the **laptop** and on **misty**.
- **Version update:** misty shipped a security-flagged build (1.80.3). Updated to **1.98.4** by manually swapping the binary over SSH on misty's OpenWrt filesystem.
- **Boot persistence:** OpenWrt's `/usr/bin` is on the overlay, so a manual binary plus a startup script survive reboots — but a *firmware update* would wipe them. Registered the relevant files so the setup is durable across flashes (keep-settings on reflash).
- Confirmed connectivity with a one-hop test.
- Configured **misty as an exit node**, so traffic can be routed out through the lab when desired.

**Takeaway:** the connection model matters more than the protocol. Outbound-only beats "VPN with a forwarded port" for a network like this one.

---

## 2. Raspberry Pi: Reflash + SSH Hardening

The Pi needed a clean Kali install on a new SD card, with the wrinkle that the only spare boot media was the SD slot itself.

### The USB-pivot reflash
You can't overwrite the card you're booted from. The move was to **boot the Pi from the 128GB USB flash drive temporarily**, which frees the SD slot so it can be written cleanly.

- Verified the Kali image against its published SHA256 before writing (a corrupt image flashed twice wastes a lot of time). An earlier Etcher write had failed — the hash match proved the *download* was fine and the problem was in the *writing*, so the fix was reflashing with **Raspberry Pi Imager** (does its own post-write verify) rather than re-downloading.
- The critical safety rule throughout: **every `dd`/flash targets a whole device, and the wrong target wipes it instantly.** On a Pi the two media enumerate differently — SD is `/dev/mmcblk0`, USB is `/dev/sda` — and `lsblk` confirms before every write.

### SSH hardening
- Re-added the Windows (`primaryuser@mothership`) **ed25519 public key** to the Pi's `authorized_keys` (the reflash had wiped it).
- Confirmed key login works **before** disabling passwords, to avoid locking myself out.
- Set `PasswordAuthentication no` — the Pi is now **key-only**.
- Routine post-flash config: hostname, timezone via `timedatectl`, NTP sync.

### The flash drive
After the pivot, the 128GB USB drive was formatted ext4 and mounted at `/mnt/storage` with an `fstab` `nofail` entry (so a missing drive doesn't block boot). It was later **wiped and repurposed** as a file-transfer mechanism. Its long-term home is still an open decision — most likely a small **Samba NAS hanging off misty's USB port** rather than Pi storage.

---

## 3. Network File Sharing: SMB on the 3TB Drive

Goal: share mothership's 3TB data drive over the network and reach it remotely. This was the most layered debugging of the phase.

### Root cause of the auth failures
The primary Windows account (**`primaryuser`**) is **Microsoft-account-backed**. That behaves differently for local network auth than a local account does — the tell was **error 8646** when attempting a `net user` password reset on it.

**Fix:** created a dedicated **local `shareuser` account** purely for file sharing. This also gives clean separation of privilege — sharing credentials live apart from my main login, so access can be revoked or changed without touching the primary account.

### The Private/Public profile trick
The physical Ethernet connection ran into Windows' Public/Private network-profile firewall rules blocking SMB. The clean sidestep: Windows classifies the **Tailscale interface as Private**, so reaching the share over Tailscale just works — and the *same* mount works identically from a café as from the LAN, because SMB simply rides the tunnel.

### Hardening pass
- Scoped the firewall allow-rule to my actual LAN subnet (**192.168.8.0/24** — caught and corrected a `.6.0/24` slip that wouldn't have matched real traffic).
- Removed an **unintentionally created Users-folder share**.
- Tightened the data drive share from **`Everyone` → `shareuser`** on *both* the share and NTFS permission layers. This shifts the model from authentication-only to authentication **+ authorization**, so future accounts don't silently inherit access.

A note on judgment here: behind double-NAT with credential gating, the `Everyone` permission was a low-urgency cleanup item, not an emergency. The hardening was the right move, but threat-modeled — not maximal-caution reflex.

---

## 4. Wake-on-LAN: Half Built

Since mothership is kept shut down for power reasons, I need to wake it remotely. WoL needs **both sides** configured — and right now only one side is done.

### misty side — DONE
- `etherwake` script at **`/usr/bin/wake-mothership`** that fires the magic packet on `br-lan` to mothership's MAC.
- Registered in **`/etc/sysupgrade.conf`** so it survives firmware updates.
- Triggerable from anywhere over Tailscale:
  ```
  ssh root@<misty-tailscale-ip> 'wake-mothership'
  ```
  (passwordless via key auth in misty's Dropbear `authorized_keys`, aliased to `wake-desktop` on the laptop).

### mothership side — TO DO
This is the remaining work, and the likely failure mode if skipped is "wakes from sleep but not from full shutdown":
- Enable **Wake on Magic Packet** in BIOS/UEFI.
- Disable **ErP / Deep-S5** (these cut power to the NIC at full-off).
- Disable Windows **Fast Startup** (it breaks WoL from a true shutdown).
- Windows NIC adapter settings: allow the device to wake the computer, magic-packet-only.
- Confirm mothership is **wired** to a misty LAN port (WoL over WiFi from off is unreliable) and reserve its IP in DHCP.

> Note: the Pi can't be a WoL *target* (hardware limitation) — it could only ever act as a sender. So this task is specifically about waking **mothership**.

---

## Key Learnings & Principles

- **Threat model drives decisions, not maximal caution.** Double-NAT + credential gating changes what's actually urgent.
- **Minimize inbound exposure.** This single principle drove the OpenVPN→Tailscale pivot and the whole architecture. Keep listening services inside the controlled boundary.
- **Defense in depth, appropriately.** Layer controls (share + NTFS, key-only SSH) without sacrificing usability against risks that don't exist.
- **The connection model > the tool.** Outbound-only is the property that mattered, not "is it a VPN."
- **WoL is a two-sided contract.** Tooling on the sender is only half the equation.
- **Windows account type matters for SMB.** MS-account-backed logins behave differently for local auth; a dedicated local account is the clean answer.
- **Verify before you destroy.** Hash-check images before flashing; confirm `lsblk` before every `dd`; test key login before disabling passwords; test a fresh `shareuser` mount before closing the working session.

---

## Current State & Roadmap

**Done this phase:** Tailscale on laptop + misty (updated, persistent, exit node) · OpenVPN evaluated and ruled out · Pi reflashed via USB pivot · Pi SSH key-only + initial config · 3TB drive shared over SMB and reachable over Tailscale · SMB permissions hardened · misty-side Wake-on-LAN tooling.

**Near-term to do:**
- Complete **mothership-side WoL** (BIOS, ErP, Fast Startup, NIC settings) — the natural next thread, since the misty side is built and waiting.
- Install **Tailscale on the Pi**.
- Test full remote access from a **truly external network**.
- Decide the **128GB flash drive's** fate (Samba NAS on misty vs. Pi storage).
- **Kali VM capstone:** autostart headless on mothership boot + make it remotely reachable (likely Tailscale *inside the VM* — cleanest given the rest of the build).
- Tailscale **subnet routing** on misty (advertise 192.168.8.0/24); learn/apply Tailscale **ACLs**.

**Future phases (cost-paced):** Domain Controller + Active Directory · SIEM · intentionally vulnerable targets · dedicated red-team box.

---

*Tracked in the Notion "Home Lab Project" database.*
