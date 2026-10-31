---
title: "Homelab: An Infrastructure-as-Code Security Lab"
date: 2026-10-31
tags: [homelab, proxmox, opnsense, active-directory, packer, terraform, ansible, purple-team]
---

## What this lab is

This is a single-server security lab that I build entirely as code, so I can tear it down and rebuild it on demand. Its main job is an Active Directory attack range: a small fake company (`corp.lab.internal`) that I attack from Kali, then study from the defender's side through a SIEM.

Everything runs on one second-hand Dell server under Proxmox VE. An OPNsense VM routes six zones over VLANs, and every allowed path between zones is an explicit firewall rule. Packer builds the images, Terraform deploys the VMs, and Ansible configures them.

The design rests on a few principles:

- **Default deny between zones.** Every cross-zone flow has a named rule and a reason.
- **Code first.** After the first few hand-built VMs (OPNsense, the control node), every VM comes from a template and a commit.
- **No secrets in templates.** Images hold software only; credentials are set per VM at deploy time and rotated on first contact.
- **Separate trust chains.** The infrastructure CA that my laptop trusts is never the AD Certificate Services I attack.
- **Clean versus dirty.** Infrastructure zones stay clean; attack and target zones are treated as compromised.
- **Verify, don't assume.** Every step has a check: plan output, dry runs, hashes, signatures, functional tests.

This page describes the target design and marks clearly what is built today and what is still planned.

## Status at a glance

The core, the internal PKI and the red team zone are built; the AD zone is in progress, with the Windows Server template being built now. Detection is designed but not yet built.

| Area | What | Status |
| --- | --- | --- |
| Core | OPNsense router/firewall, offline root CA, Ansible control node | Built |
| INFRA | Bastion jump host, step-ca intermediate CA | Built |
| REDTEAM | Kali attack box, configured by Ansible | Built |
| Templates | Debian 13 and Kali templates | Built |
| Templates | Windows Server 2025 template | In progress |
| Templates | Windows 11 Enterprise template | Planned (next) |
| AD | Domain controller, workstation, Terraform root, AD roles | Planned |
| Detection | Elastic SIEM, Sysmon, Suricata IDS on OPNsense | Designed, not built |
| Backups | Off-box copies of repo, secrets DB and Terraform state | Partial |
| CTF, MALWARE | Solver workstations; isolated analysis zone | Planned |

## Hardware and hypervisor

The whole lab runs on one second-hand Dell PowerEdge R630 running Proxmox VE 9.

| Component | Detail |
| --- | --- |
| Server | Dell PowerEdge R630 |
| CPU | 2 x Xeon E5-2660 v3 (40 threads) |
| RAM | 64 GB (24 DIMM slots, so cheap to grow) |
| Storage | PERC H730P, 8 x 300 GB 10k SAS HDD, one hardware RAID volume with LVM-thin (~2.2 TB) |
| Hypervisor | Proxmox VE 9.2 |

The Proxmox host stays on my home LAN, outside the lab, and its firewall admits SSH and the web UI only from my laptop and the Ansible control node. That way a broken OPNsense config can never lock me out of the hypervisor.

Only the core VMs stay on; zones are powered up when I use them. With everything planned, the RAM budget is roughly 40 to 45 GB of the 64 GB (my estimate). The real bottleneck is the spinning disks: a fully patched Windows template takes about 2.5 hours to build.

Storage is an open decision. The array is currently striped for capacity, so a single disk failure would take the whole lab with it. I keep off-box copies of the code, the secrets database and the Terraform state, and I am weighing a rebuild as RAID 10.

## Network design

Every routed zone is a VLAN on one internal Open vSwitch bridge, and the only path between zones runs through the OPNsense VM.

![Lab topology: one Proxmox host, three bridges, six zones]({{ '/assets/img/homelab/topology.svg' | relative_url }})

OPNsense is the hub: its WAN sits on the home LAN, and one trunk vNIC carries all lab VLANs. Each lab VM gets a tagged port for its zone, so a packet from Kali to a domain controller leaves on VLAN 40, is filtered by the REDTEAM rules, and is routed out on VLAN 30. The malware zone sits on a separate bridge with no OPNsense interface at all, so there is nothing to route through. (docker01, the AD hosts and the CTF and MALWARE zones are planned.)

Three bridges, one job each:

- **vmbr0** (Linux bridge, physical NIC): the outside world and the only bridge with a host IP. It stays a plain Linux bridge so experiments with OVS can never cut me off from the host.
- **vmbr1** (OVS, no physical port, no host IP): the virtual managed switch for all routed zones. Giving a lab bridge an IP would make the hypervisor a member of that network, bypassing the firewall.
- **vmbr2** (OVS, no uplink, no router): the malware island.

Addressing follows one pattern: zone VLAN N uses `10.10.N.0/24`, with OPNsense at `.1` and static, Terraform-assigned hosts below `.100`. The lab avoids `192.168.0.x` and `192.168.1.x` so it never collides with home or hotel networks.

Names live under `lab.internal`, a top-level domain reserved for private use. Unbound on OPNsense resolves every zone. The AD zone is the exception: domain members use the domain controllers for DNS, and Unbound will forward `corp.lab.internal` to them so the red team can still resolve the targets.

## Zones and VMs

Six zones split the lab by trust: two clean infrastructure zones, three zones treated as compromised, and one air-gapped island.

| Zone | Purpose | Trust |
| --- | --- | --- |
| MGMT (VLAN 10) | Admin network for the firewall | Clean |
| INFRA (VLAN 20) | Control plane: automation, jump host, CA, SIEM | Clean; nothing internet-facing or vulnerable runs here |
| AD (VLAN 30) | `corp.lab.internal`, the fake company I attack | Treated as compromised; no internet egress |
| REDTEAM (VLAN 40) | Attack tooling | Treated as compromised |
| CTF (VLAN 50) | Workstations for solving external CTFs; internet only, not targets | Treated as compromised |
| MALWARE (vmbr2) | Malware analysis with a fake internet | Isolated: no router, no route |

Each zone has its own Proxmox resource pool and VM ID range (AD uses 3000 to 3999, for example), so Terraform's API token can be scoped to just the pools it manages.

| VM | Zone | Role | OS | Status |
| --- | --- | --- | --- | --- |
| opnsense | Core | Router, firewall, DNS (Unbound), planned Suricata IDS | OPNsense | Built |
| Root CA | Core | Offline root CA, powered off except to sign | Debian 13 | Built |
| ansible01 | INFRA | Terraform, Packer and Ansible control node; Git repo | Debian 13 | Built |
| bastion01 | INFRA | SSH jump host, restricted forwarding only | Debian 13 | Built |
| ca01 | INFRA | step-ca intermediate CA with ACME | Debian 13 | Built |
| docker01 | INFRA | Elastic SIEM (Elasticsearch, Kibana, Fleet) | Linux + Docker | Planned |
| kali01 | REDTEAM | Headless attack box | Kali Rolling | Built |
| dc01 | AD | Domain controller for `corp.lab.internal` | Windows Server 2025 Standard | Planned |
| ws01 | AD | Domain workstation, target of the defense tiers | Windows 11 Enterprise | Planned |
| dc02 | AD | Second DC on Server Core, managed with RSAT | Windows Server 2025 Core | Later |
| flare01, remnux01 | MALWARE | Windows and Linux analysis boxes | Windows 11, REMnux | Later |

Templates live in their own pool: Debian 13, Kali, Windows Server 2025 (building now) and Windows 11 (next). Each one is built by Packer and never edited by hand.

Two choices here deviate from Microsoft's guidance on purpose. dc01 uses Desktop Experience rather than Server Core, because I want to learn the GUI tools first; a Server Core DC comes later. The Windows images are evaluation editions, so they expire and get rebuilt on a schedule, which the automation makes cheap.

## Firewall policy

OPNsense blocks every interface that has no pass rule, so each zone starts at deny and gets only the flows below.

| From | To | Ports | Why | Status |
| --- | --- | --- | --- | --- |
| Laptop | bastion01 | SSH | Only way into the lab; bastion allows forwarding to named hosts only | Built |
| INFRA | Firewall | DNS, NTP | Name resolution and time | Built |
| INFRA | Internet (not private nets) | HTTP, HTTPS | Package updates | Built |
| ansible01 | Proxmox API | 8006 | Terraform and Packer | Built |
| ansible01 | REDTEAM, AD | SSH only | Configuration push (Windows is managed over SSH too) | Built |
| bastion01 | kali01 | SSH | My interactive access to Kali | Built |
| REDTEAM | Firewall | DNS, NTP | Name resolution and time | Built |
| REDTEAM | AD | Any | Attack paths | Built |
| REDTEAM | Internet (not private nets) | HTTP, HTTPS | Tool updates | Built |
| Domain controllers | Firewall | DNS (+ NTP for the PDC) | DNS forwarding | Planned |
| AD | REDTEAM | Callback ports | Reverse shells and C2 stay inside the lab | Planned |
| AD | Internet | HTTP, HTTPS | Windows Update, disabled by default | Planned |

Nothing in a dirty zone may reach MGMT, INFRA, my home LAN or the Proxmox host. Return traffic to INFRA exists only as state created by ansible01, so a compromised box cannot open a connection to the control node.

Two gotchas shaped the rule set:

- **"Internet" means "not private networks".** A pass rule with destination `any` would also let the lab reach my home LAN and the firewall's WAN address, so every internet rule uses a `NOT PRIVATE_NETS` alias.
- **Block-to-firewall goes right after DNS.** A destination like "AD net" includes OPNsense's own `.1` address on that VLAN. Each dirty interface therefore blocks traffic to the firewall itself immediately after its DNS and NTP pass rules.

A second firewall layer runs on Proxmox. It does not copy OPNsense; it covers what OPNsense never sees. The host firewall drops everything except SSH and the web UI from my laptop and the API from ansible01. ansible01's own NIC accepts SSH only from the bastion. Later, per-VM rules will segment traffic inside a zone, which stays on the switch and never crosses OPNsense.

## Build pipeline

Packer builds images, Terraform clones VMs from them, and Ansible configures what Terraform created; all three run from one Git repository on ansible01.

![Build pipeline: Packer, then template, then Terraform, then Ansible]({{ '/assets/img/homelab/pipeline.svg' | relative_url }})

The split follows one rule: software goes into the image, anything secret or frequently changed goes into Ansible. Every change runs through a gate first: `terraform plan` must show the expected changes and no surprise destroys, and Ansible runs with `--check --diff` before it touches a host.

The Windows Server 2025 template shows what that looks like in practice:

- **Unattended install from the Microsoft ISO**, with the image selected by name rather than index, so a re-ordered ISO fails loudly instead of installing the wrong edition.
- **UEFI with Secure Boot**, enrolled with Microsoft's 2023 certificates (the 2011 ones expired in 2026).
- **VirtIO drivers with signature checks.** The build verifies every driver file's Authenticode signature and skips any folder that is unsigned or test-signed.
- **SSH instead of WinRM.** Windows Server 2025 ships OpenSSH, and current Ansible manages Windows over SSH officially, so the lab has one management protocol: TCP 22 from ansible01 only.
- **Windows Update through the update API**, three passes with reboots, logging every step to a file inside the guest so a failure leaves evidence.
- **cloudbase-init**, pinned by hash and signature, reads the Proxmox config drive on first boot: hostname, network, and the Ansible SSH key.
- **Sysprep**, after scrubbing answer files, logs and host keys, so every clone gets a unique identity.

One build takes about 2.5 hours on these disks. The sixth run is in progress as I write this; the fifth got as far as sysprep. Each failed run produced a fix in the repo rather than a hand edit on the VM.

## Secrets and PKI

No credential lives in a template, a `.tf` file or the Git repo; secrets are pulled at run time and scoped per zone.

- **KeePassXC** on ansible01 holds the Proxmox API tokens and build passwords. Scripts read them with `keepassxc-cli` into environment variables, never onto a command line.
- **Ansible Vault** will hold AD passwords, unlocked through a vault-id script that asks KeePassXC.
- **Per-zone SSH keys.** Ansible uses a separate key for each zone, so a compromised zone exposes only its own access.
- **Least-privilege API tokens.** Terraform and Packer each have their own Proxmox user and token, limited to the pools, storage and VM IDs they need. Terraform can clone into the lab pools but cannot touch the core VMs.
- **Terraform state** stays on ansible01, is never committed, and is backed up encrypted inside the KeePassXC database.
- **Bootstrap credentials are throwaway.** A clone starts with a bootstrap account; on first contact Ansible creates its own account, sets random passwords and removes the bootstrap one.

The internal PKI has two tiers. An offline root CA (P-384, valid to 2036) sits in a powered-off VM and only wakes up to sign. It is name-constrained to `lab.internal` and the lab's address space, so even a stolen key cannot mint a certificate my laptop would trust for a public site. Below it, ca01 runs step-ca as the online intermediate and issues 30-day certificates over ACME.

Active Directory Certificate Services will be a separate root inside the AD zone, trusted only by domain members. It is a target, deliberately misconfigured for the ESC attack family, and never part of the chain my laptop trusts.

## What I do with it

The lab exists to run Active Directory attacks and then see them from the defender's side. Most of this section is the plan the build is heading toward; the AD hosts and the SIEM are not built yet.

![Purple-team loop: attack, telemetry, SIEM, score, tune]({{ '/assets/img/homelab/purple-loop.svg' | relative_url }})

**Attacking AD.** Kali in the REDTEAM zone attacks `corp.lab.internal` across the firewall: Kerberoasting, AS-REP roasting, BloodHound collection, AD CS abuse with Certipy, and relaying. These are network-side attacks, so they work regardless of the antivirus on the workstation. AD has no internet egress, and C2 callbacks are only allowed back into REDTEAM, so my own payloads never call out from my home IP.

**Defense tiers.** ws01 can be set to different difficulty levels on two independent axes instead of one ladder:

| Axis | Levels |
| --- | --- |
| Prevention | Off, Defender defaults, + ASR rules and tamper protection, + hardening (LAPS, LSA protection, AppLocker/WDAC, SMB and LDAP signing, Protected Users) |
| Detection | None, Sysmon to SIEM, Aurora Lite to SIEM, Elastic Defend |

An Ansible variable `defense_level` selects the level, and each level gets a Proxmox snapshot so I can reset between runs. A run counts twice: did I reach Domain Admin, and did I do it with zero alerts? One caveat: with no egress, Defender runs without cloud protection, so a win means "beat offline Defender", not "beat Defender".

**Detection stack.** Elastic (self-managed, free Basic license) runs on docker01. Sysmon and Elastic Agent are added to Windows hosts by Ansible, not baked into the templates, so the templates stay neutral. OPNsense ships firewall logs, Unbound DNS queries and Suricata alerts. Suricata runs in IDS mode only (alert, not block): inline IPS on OPNsense relies on netmap, which Suricata maintainers advise against with VLAN interfaces and report as unreliable for FreeBSD guests on Proxmox, exactly my setup, and alerts are what this loop needs anyway.

**Known blind spot.** Traffic inside a zone, such as ws01 to dc01 during lateral movement or DCSync, stays on the virtual switch and never passes OPNsense or Suricata. Host telemetry covers it for now; a mirrored switch port feeding a sensor VM is on the list.

Other scenarios I want to run later: AD CS escalation with matching detections, before/after hardening comparisons, deception, a second forest with a trust, pivoting through a DMZ, and incident response on the results.

## Lessons learned

Most of the time so far went into small failures, and each one left a rule behind.

- **Proxmox checks VM-create permissions on the VM ID, not the pool.** A pool-scoped token cannot create a VM until it has rights on `/vms/<id>`, and destroying a VM silently deletes those rights. A small prep script now re-applies them before every build.
- **Signature checks earn their keep.** Checking every VirtIO driver and installer caught unsigned installers and test-signed drivers on an official ISO.
- **A hash proves integrity, not authenticity.** A checksum published next to an image only proves the download wasn't corrupted. Tamper evidence needs a signature or an independent channel.
- **A filename proves nothing.** A file named like a config snippet turned out to be an old archive. I now check the SHA-256 of anything I copy on the machine where it lands.
- **Decode odd exit codes before guessing.** Exit 267014 looked like a script error; it is `SCHED_S_TASK_TERMINATED`, meaning the scheduled task was killed. A stream of "shutdown in progress (1115)" errors was Packer's harmless reboot probe.
- **Keep the evidence.** Packer's "clean up" option after a failure deletes the VM and its event logs. I now abort and keep the VM, and long Windows steps log to a file inside the guest that I can read over SSH mid-build.
- **Extra keystrokes have side effects.** Spare key presses meant to skip "press any key to boot from CD" also stopped the countdown on the next boot menu.
- **Installed copies drift.** A script fixed in the repo was still broken on the host. Comparing hashes after every fix catches it.
- **A 2.5-hour build needs the host for 2.5 hours.** Shutting the server down mid-build killed Packer with no cleanup. Builds now run in tmux on the control node, and I plan around them.
- **Snapshots are not backups.** Every copy on the same array dies with the array.

## Roadmap

Next up is finishing the Windows templates, then standing up the domain; the SIEM follows once AD has something to log.

1. **Windows templates.** Finish and test-clone the Windows Server 2025 template, then build the Windows 11 Enterprise template (UEFI, Secure Boot, vTPM).
2. **AD zone.** Terraform support for Windows VMs, then dc01 and ws01; Ansible for credential rotation, DC promotion, AD firewall rules and DNS forwarding.
3. **Defense role.** The `defense_level` variable on ws01, starting at Defender defaults, with a snapshot per level.
4. **Storage and backups.** Decide on RAID 10, back up the offline root CA, and add a second backup medium.
5. **Housekeeping.** Rotate the Proxmox API tokens with expiry dates and work through the deferred-hardening list I keep alongside the build.
6. **PKI consumers.** Test ACME issuance and put lab certificates on Proxmox, the bastion and other services.
7. **Detection.** docker01 with Elastic, Sysmon and Elastic Agent via Ansible, Suricata IDS on OPNsense. Then the purple-team loop starts for real.
8. **Later.** More detection tiers (Aurora Lite, Elastic Defend), a Server Core DC, a mirrored-port network sensor for traffic inside zones, the CTF zone, and the isolated malware zone last.
