Homelab Setup
=============

I recently purchased a used Dell Poweredge r630 from Amazon and have completely overhauled my homelab. The additional RAM and CPU cores has made a lot of things possible that were either impossible on my old equipment or were unbearably slow.

I built this new lab alongside Michael Taggrt's <i>excellent</i> Homelab Almanac v3.0 (https://taggart#tech.com/thav3) and have learned so much, particularly about DevOps tools like Terraform and Ansible. 

My new setup is divided into several different zones for different kinds of activities. I'll break each one of these sections out into their own page but below is a brief summary of each one: 

# Malware Analysis
- For malware analysis (duh). 
- Two Windows machines (one for the victim and one for analysis), one linux machine for analysis, and a SIEM. 
- More details [here](./malware_analysis.md) 
 
# Active Directory Lab
- To learn more about attacking and defending in an AD environment. 
- Currently just one domain with a domain controller, a file server, a SQL server, and a couple of Windows workstations. I have plans for expanding this into a whole forest with different trusts, etc. 
- More details [here](./active_directory.md) 

# CTF's
- Strictly for CTF's. 
- A Kali machine, a Windows machine, and a small BIND server to resolve CTF domains. 
- More details [here](./ctf.md) 
 
# Red Team Emulation
- I wanted a way of playing with "real" red team TTP's like Apache redirectors and reverse proxies. 
- A Windows workstation victim, a Kali box, a small LAMP stack, and a mail server. 
- More details [here](./redteam_emulation.md) 

# Infrastructure
- The heart of the whole thing. Pretty much everything that happens in any of the lab sections are able to happen because of the Infrastructure. 
- A Docker host, a WordPress server, a webserver running Caddy, a bastion host aka jumpbox, a Velociraptor server, a Zeek server, another Kali box, and a Bloodhound server.
- More details [here](./infrastructure.md) 

# Work
- Strictly for my work activities.
- A Debian machine, a Windows 11 machine, and a MISP instance on the way. 
- More details [here](./work.md) 

<br>

Altogether, it looks something like [this](./network_diagram.md) 
