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

The Flare, REMnux, and SIFT VM's are on a very locked down subnet that doesn't get internet access and can't touch any other subnets in the lab (except for when I allow it)

The Active Directory machines are on their own subnet

Kali and CommandoVM are in a subnet that does get internet access. 

The OPNSense VM inherently touches all of the subnets. 

Altogether, it looks something like this: 

![homelab network diagram](https://github.com/jabsec/portfolio/blob/main/homelab/homelab%20diagram.png "Network Diagram")
