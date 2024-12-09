  Jabsec Cybersecurity Porfolio

Homelab Setup
=============

I'm currently running Proxmox on an old Dell Workstation. It is underpowered but it still allows me to explore and practice in a contained envrionment.

I have multiple purpose-built VM's ready to be used when I need them: 

- FlareVM
- REMnux
- SIFT
- Kali
- CommandoVM

I also have an Active Directory environment with a domain controller and a windows workstation that is joined to the domain. 

Lastly, I have two "utility" VM's running at all times - one is OPNSense which handles the network segmentation and DHCP services, and the other is a jumpbox/bastion server that I connect to in order to access my homelab. 

Altogether, it looks something like [this](./network_diagram.md) 
