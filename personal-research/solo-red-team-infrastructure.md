# Solo Operator Red Team Infrastructure

A practical guide to building autonomous, disposable attack infrastructure for solo pentesters and small red teams — from home lab to cloud-deployed operations.

## Table of Contents

- [Overview](#overview)
- [Architecture Philosophy](#architecture-philosophy)
- [The Home Lab Foundation](#the-home-lab-foundation)
- [Cloud Redirector Layer](#cloud-redirector-layer)
- [Building Your Own VPN Tunnels](#building-your-own-vpn-tunnels)
- [Infrastructure Automation with Terraform and Ansible](#infrastructure-automation-with-terraform-and-ansible)
- [Functional Separation vs. Serial Chaining](#functional-separation-vs-serial-chaining)
- [Tor: Where It Fits and Where It Doesn't](#tor-where-it-fits-and-where-it-doesnt)
- [Full Solo Operator Architecture](#full-solo-operator-architecture)
- [Operational Workflow](#operational-workflow)
- [Tools and Resources](#tools-and-resources)

---

## Overview

Red team infrastructure is the backbone of any adversary simulation. A well-designed setup provides operational security (OPSEC), resilience against blue team detection, and the ability to rapidly deploy and destroy assets between engagements. This guide is written for the solo operator or very small team looking to build a professional-grade infrastructure without enterprise-scale budgets or resources.

The key principles throughout are **disposability**, **compartmentalization**, and **automation**.

---

## Architecture Philosophy

In professional red teaming, the goal is not to hide from law enforcement (you have a contract) — it's to avoid detection by the **blue team** long enough to accomplish your objectives and realistically test defenses.

This changes the design calculus. You don't need perfect anonymity. You need:

- **Operational separation** — Your real IP never touches the target.
- **Disposability** — Burned infrastructure can be replaced in minutes, not hours.
- **Compartmentalization** — Compromise of one asset doesn't expose the others.

The best reference architecture for this concept comes from the [Red Team Infrastructure Wiki](https://github.com/bluscreenofjeff/Red-Team-Infrastructure-Wiki), which recommends placing a redirector in front of every back-end asset so that the target only ever sees the disposable front-end nodes.

---

## The Home Lab Foundation

Your home lab is the persistent core of operations — the brains that control everything else.

### Hardware

You don't need a rack of servers. A single machine with 32–64 GB of RAM and a modern multi-core CPU can host a dozen or more VMs comfortably. Options include refurbished enterprise micro-servers, Intel NUCs, or even a well-specced desktop.

### Hypervisor

**Proxmox VE** is the go-to choice for solo operators. It's open-source, Debian-based, supports KVM and LXC containers, and provides a web-based management interface. It handles everything from VM provisioning to network bridge configuration and snapshot management without any licensing costs.

### VM Layout

A practical home lab VM structure looks like this:

| VM | Purpose |
|---|---|
| **Operator Workstation** | Primary Kali/Parrot attack OS with your tooling |
| **Infra Management** | Runs Terraform, Ansible, and git repos for your playbooks |
| **C2 Team Server** | Runs your C2 framework (Sliver, Mythic, Cobalt Strike) |
| **Dev/Staging** | Testing payloads and infrastructure changes before deployment |

### Networking

Use pfSense or OPNsense as a virtual router to segment your lab into isolated subnets. Your attack VMs, your team server, and your management plane should all be on separate network segments with firewall rules controlling traffic between them.

---

## Cloud Redirector Layer

Your home lab never faces the target directly. Cloud VPS instances act as disposable front-end nodes.

### The Role of Redirectors

A redirector is a lightweight VPS that sits between the target and your team server. It forwards C2 traffic, phishing callbacks, or scanning traffic back to your home lab through an encrypted tunnel. If the blue team identifies a redirector IP, you destroy it and stand up a replacement — the team server and all your operational data remain untouched.

This concept of separating disposable ("burnable") infrastructure from persistent back-end assets is a foundational red team design principle. Tools like [Red Baron](https://github.com/Coalfire-Research/Red-Baron) and [PrimusRedir](https://primusinterp.com/posts/Primus_Redir/) were built specifically to automate this redirector lifecycle.

### Choosing VPS Providers

Use cheap VPS instances ($5–10/month tier) from providers like DigitalOcean, Vultr, or Linode. A few guidelines:

- Use **different providers** for different roles (scanning vs. C2 vs. phishing) so that a ban from one provider doesn't take down everything.
- Pick regions that make geographic sense for the target.
- Pay with methods that minimize attribution where the engagement scope allows.

---

## Building Your Own VPN Tunnels

A major weakness in many setups is depending on commercial VPN providers. They introduce a third party who can log, correlate, or compromise your traffic. The solution is to build your own encrypted tunnels.

### WireGuard

WireGuard is the standard for this. It's fast, minimal (roughly 4,000 lines of code), has a tiny attack surface, and configuration is straightforward. A basic tunnel between your home lab and a VPS is about 10–15 lines of config on each side.

**On the VPS (server):**

```ini
# /etc/wireguard/wg0.conf
[Interface]
PrivateKey = <server_private_key>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <client_public_key>
AllowedIPs = 10.0.0.2/32
```

**On the home lab VM (client):**

```ini
# /etc/wireguard/wg0.conf
[Interface]
PrivateKey = <client_private_key>
Address = 10.0.0.2/24

[Peer]
PublicKey = <server_public_key>
Endpoint = <vps_public_ip>:51820
AllowedIPs = 10.0.0.0/24
PersistentKeepalive = 25
```

**Bring the tunnel up:**

```bash
wg-quick up wg0
```

This creates a private encrypted network between your lab and the VPS with no third party in the middle. Scale this by giving each VPS its own tunnel, and each VM in your lab its own WireGuard interface connecting to a different exit point.

C2 design guidance from practitioners recommends using VPN technologies like WireGuard, Nebula, or Tailscale to create secure communication channels between redirectors and back-end team servers — with tunnels originating from the team server side rather than exposing inbound ports on the redirector.

---

## Infrastructure Automation with Terraform and Ansible

Manually provisioning and configuring VPS instances is slow, error-prone, and doesn't scale. Infrastructure-as-code tools solve this.

### Terraform — Provisioning

Terraform talks to cloud provider APIs and provisions servers, DNS records, and firewall rules from declarative config files. The core value proposition for red teams is the ability to define infrastructure once and deploy or destroy it with a single command.

A minimal Terraform config to stand up a DigitalOcean redirector:

```hcl
# main.tf
terraform {
  required_providers {
    digitalocean = {
      source = "digitalocean/digitalocean"
    }
  }
}

variable "do_token" {}

provider "digitalocean" {
  token = var.do_token
}

resource "digitalocean_droplet" "redirector" {
  image  = "ubuntu-24-04-x64"
  name   = "redir-01"
  region = "fra1"
  size   = "s-1vcpu-1gb"

  ssh_keys = [var.ssh_fingerprint]
}

output "redirector_ip" {
  value = digitalocean_droplet.redirector.ipv4_address
}
```

```bash
terraform init
terraform apply    # Stand up infrastructure
terraform destroy  # Tear it all down
```

Terraform configurations are designed to be minimal and disposable — operational sufficiency, not performance, is the goal.

### Ansible — Configuration

Once Terraform provisions the raw servers, Ansible configures them. It installs your tooling, deploys WireGuard configs, sets up firewall rules, and hardens the OS. All from a playbook you write once and reuse across engagements.

```yaml
# playbooks/setup-redirector.yml
---
- hosts: redirectors
  become: yes
  tasks:
    - name: Install WireGuard
      apt:
        name: wireguard
        state: present
        update_cache: yes

    - name: Deploy WireGuard config
      template:
        src: templates/wg0.conf.j2
        dest: /etc/wireguard/wg0.conf
        mode: '0600'

    - name: Enable WireGuard tunnel
      systemd:
        name: wg-quick@wg0
        enabled: yes
        state: started

    - name: Configure iptables forwarding
      iptables:
        table: nat
        chain: PREROUTING
        protocol: tcp
        destination_port: "443"
        jump: DNAT
        to_destination: "10.0.0.2:443"
```

The combination of Terraform and Ansible has become the de facto standard for red team infrastructure automation. Tools like Red Baron, Red Ira, and PrimusRedir all build on this foundation.

---

## Functional Separation vs. Serial Chaining

A common question is whether to chain multiple proxies in series for deeper anonymity:

```
Attacker → VPS → VPS → VPS → Target
```

The answer for most engagements is that **parallel separation beats serial depth**. Each additional hop in a serial chain adds latency, debugging complexity, and failure points — while solving a problem (forensic back-tracing through multiple providers) that rarely materializes in practice. Blue team investigations almost always stop at the first cloud provider dead end.

Instead, use multiple VPS nodes for **different functions**:

```
VPS #1 — Recon and scanning
VPS #2 — Phishing infrastructure
VPS #3 — HTTPS C2 relay
VPS #4 — DNS C2 (backup channel)
```

If the blue team catches your phishing domain and traces it to VPS #2, they learn nothing about your C2 on VPS #3. That compartmentalization provides far more operational resilience than extra serial hops.

---

## Tor: Where It Fits and Where It Doesn't

Tor provides strong anonymity, but its tradeoffs make it a supplementary tool rather than a primary channel for red team operations.

### Where Tor Is Useful

- **Early passive reconnaissance.** OSINT gathering, browsing the target's public infrastructure, reviewing job postings — activities where you want throwaway anonymity and don't need speed.
- **Hiding the link to your infrastructure.** Route traffic through Tor to reach your own VPS, so even the hosting provider's logs trace back to a Tor exit, not to your real IP.
- **C2 over hidden services.** Some C2 frameworks support `.onion` listeners, keeping all traffic within the Tor network and eliminating the exit node exposure problem.

### Where Tor Falls Short

- **Speed and reliability.** Latency kills interactive tools, scanning, and anything requiring stable connections.
- **Exit node trust.** Unencrypted traffic through unknown exit operators is a risk for sensitive operational data.
- **Detection.** Corporate security products maintain lists of known Tor exit IPs. Traffic from Tor exits is often **more** suspicious than traffic from a random cloud VPS on a corporate network.
- **Instability.** Circuit rotation drops connections, which is unacceptable for C2 channels.

For most engagements, dedicated VPS redirectors with WireGuard tunnels provide better operational security with fewer tradeoffs than routing active operations through Tor.

---

## Full Solo Operator Architecture

Putting it all together, here is a practical reference architecture for a single operator:

```
┌─────────────────────────────────────────────────────┐
│                  HOME LAB (Proxmox)                  │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │   Operator    │  │  C2 Team     │  │   Infra    │ │
│  │  Workstation  │  │   Server     │  │  Mgmt/IaC  │ │
│  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘ │
│         │                 │                 │        │
│         └────────┬────────┘                 │        │
│                  │ (internal network)        │        │
│                  │                           │        │
│              pfSense/OPNsense               │        │
│                  │                           │        │
└──────────────────┼───────────────────────────┼───────┘
                   │                           │
            WireGuard Tunnels          Terraform/Ansible
                   │                     Provisioning
                   │                           │
     ┌─────────────┼─────────────┐             │
     │             │             │             │
     ▼             ▼             ▼             │
┌─────────┐ ┌─────────┐ ┌─────────┐          │
│  VPS #1 │ │  VPS #2 │ │  VPS #3 │ ◄────────┘
│  Recon  │ │ Phish   │ │  C2     │
│ Scanner │ │ Relay   │ │ Redir   │
└────┬────┘ └────┬────┘ └────┬────┘
     │           │           │
     ▼           ▼           ▼
           Target Network
```

**Monthly Cost Estimate:** ~$15–30/month for 3 cloud VPS nodes during active engagements. $0 when idle (everything torn down). Home lab power costs only.

---

## Operational Workflow

### Pre-Engagement

1. Update your Terraform configs with new provider regions/sizes if needed.
2. Run `terraform apply` — VPS nodes are provisioned in minutes.
3. Run Ansible playbooks — WireGuard tunnels come up, tooling is deployed, firewall rules are set.
4. Verify connectivity from your home lab through each redirector.

### During Engagement

5. Route all scanning through VPS #1.
6. Phishing campaigns launch from VPS #2 infrastructure.
7. C2 callbacks route through VPS #3 back to your team server via WireGuard.
8. If any VPS gets burned, `terraform destroy` the single node and re-provision a replacement.

### Post-Engagement

9. Exfiltrate any data/logs needed for the report.
10. `terraform destroy` — all cloud infrastructure is gone.
11. Clean up credentials: `rm /tmp/.git-credentials` and any stored keys.
12. Snapshot your team server VM if you need to preserve operational data.

---

## Tools and Resources

### Infrastructure Automation

| Tool | Purpose |
|---|---|
| [Terraform](https://www.terraform.io/) | Infrastructure-as-code provisioning |
| [Ansible](https://www.ansible.com/) | Configuration management and deployment |
| [Red Baron](https://github.com/Coalfire-Research/Red-Baron) | Terraform modules for red team infrastructure |
| [PrimusRedir](https://primusinterp.com/posts/Primus_Redir/) | Automated redirector deployment with WireGuard + Caddy |
| [Red Wizard](https://cybersecurity.bureauveritas.com/blog/red-wizard-1) | Full red team infrastructure automation suite |

### C2 Frameworks

| Tool | Purpose |
|---|---|
| [Sliver](https://github.com/BishopFox/sliver) | Open-source C2 framework |
| [Mythic](https://github.com/its-a-feature/Mythic) | Modular, multi-platform C2 |
| [Cobalt Strike](https://www.cobaltstrike.com/) | Commercial adversary simulation platform |

### Tunneling and Proxying

| Tool | Purpose |
|---|---|
| [WireGuard](https://www.wireguard.com/) | Lightweight encrypted tunnels |
| [Chisel](https://github.com/jpillora/chisel) | TCP/UDP tunneling over HTTP |
| [Ligolo-ng](https://github.com/nicocha30/ligolo-ng) | Advanced pivoting with TUN interface |

### Lab and Learning

| Resource | Link |
|---|---|
| Red Team Infrastructure Wiki | [github.com/bluscreenofjeff/Red-Team-Infrastructure-Wiki](https://github.com/bluscreenofjeff/Red-Team-Infrastructure-Wiki) |
| Hack The Box | [hackthebox.com](https://www.hackthebox.com/) |
| 100 Days of Red Team (Terraform series) | [100daysofredteam.com](https://www.100daysofredteam.com/) |
| Rasta Mouse IaC Guide | [rastamouse.me](https://rastamouse.me/infrastructure-as-code-terraform-ansible/) |

---

## Final Thoughts

You don't need a six-figure budget or a team of ten to run professional red team operations. One machine under your desk, a handful of cheap cloud instances, and solid automation give a solo operator everything needed to deploy, operate, and tear down attack infrastructure on demand.

The key insight is that every third party in your chain is a potential point of failure, logging, or compromise. The more of the stack you own and automate yourself, the more control and operational security you have. Start simple — one Proxmox host, one VPS with WireGuard, one Ansible playbook — and expand as your engagements require it.

---

*This writeup is intended for educational purposes and authorized security testing only. Always obtain proper written authorization before conducting any penetration testing or red team operations.*

---

## References

1. **Jeff Dimmock** — *Red Team Infrastructure Wiki.* GitHub. [github.com/bluscreenofjeff/Red-Team-Infrastructure-Wiki](https://github.com/bluscreenofjeff/Red-Team-Infrastructure-Wiki)
2. **Cedric Owens** — *Infra Automation Primer (Red Team Edition).* Medium, 2021. [medium.com/red-teaming-with-a-blue-team-mentality](https://medium.com/red-teaming-with-a-blue-team-mentality/infra-automation-primer-red-team-edition-b4c613308beb)
3. **Rasta Mouse** — *Infrastructure as Code (Terraform + Ansible).* rastamouse.me, 2024. [rastamouse.me/infrastructure-as-code-terraform-ansible](https://rastamouse.me/infrastructure-as-code-terraform-ansible/)
4. **Secura / Bureau Veritas** — *Red Wizard: User-Friendly Automated Infrastructure for Red Teaming.* [cybersecurity.bureauveritas.com/blog/red-wizard-1](https://cybersecurity.bureauveritas.com/blog/red-wizard-1)
5. **Joe Minicucci** — *Introducing Red Ira: Red Team Infrastructure Automation Suite.* blog.joeminicucci.com, 2021. [blog.joeminicucci.com/2021/redira](https://blog.joeminicucci.com/2021/redira)
6. **Uday Mittal** — *Understanding C2 Infrastructure — Part 2: Design Considerations.* 100 Days of Red Team, 2024. [100daysofredteam.com](https://www.100daysofredteam.com/p/understanding-c2-infrastructure-part-2)
7. **100 Days of Red Team** — *Kickstarting Red Team Infrastructure Automation via Terraform.* 2025. [100daysofredteam.com](https://www.100daysofredteam.com/p/kickstarting-red-team-infrastructure-automation-via-terraform)
8. **PrimusInterp** — *PrimusRedir: Forwarding the World to Your C2.* primusinterp.com, 2025. [primusinterp.com/posts/Primus_Redir](https://primusinterp.com/posts/Primus_Redir/)
9. **Wambui Ngige** — *Provisioning Red Team Infrastructure with Terraform and Ansible on Vultr.* Medium, 2021. [wambui-ngige.medium.com](https://wambui-ngige.medium.com/provisioning-red-team-infrastructure-with-terraform-and-ansible-on-vultr-17045b500128)
10. **TrustedSec** — *Automating a RedELK Deployment Using Ansible.* trustedsec.com, 2025. [trustedsec.com/blog](https://trustedsec.com/blog/automating-a-redelk-deployment-using-ansible)
11. **Coalfire Research** — *Red Baron: Automated Resilient Red Team Infrastructure.* GitHub. [github.com/Coalfire-Research/Red-Baron](https://github.com/Coalfire-Research/Red-Baron)
12. **MDSec** — *Testing Your Red Team Infrastructure.* mdsec.co.uk, 2020. [mdsec.co.uk](https://www.mdsec.co.uk/2020/02/testing-your-redteam-infrastructure/)
13. **OOSEC** — *Automating Red Team Infrastructure for Agility — Part 1.* oosecgh.com, 2026. [oosecgh.com](https://oosecgh.com/automating-red-team-infrastructure-for-agility-part-1/)
14. **Whispergate** — *InfraGuard: C2 Redirection Proxy and Manager.* GitHub. [github.com/Whispergate/InfraGuard](https://github.com/Whispergate/InfraGuard)
15. **Joseph DePalo** — *Cybersecurity Home Lab Part 1: Proxmox Installation.* Medium, 2024. [medium.com/@josephdepalo](https://medium.com/@josephdepalo/cybersecurity-home-lab-part-1-proxmox-installation-b85aadf7fc55)
