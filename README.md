
# Linux Security Lab 

This is my home lab where I hardened my Ubuntu server, ssh, permissions, and firewall to protect my server from being compromised. I installed tools like tcpdump to watch network traffic from the defenders perspective. I attacked my own server from kali, noticed kali and windows produced different responses after being banned, investigated what the problem might be, tested it, and traced the real cause to VirtualBox's separate virtual network path. 


Each folder contains my notes, explanations, and evidence for each week.


## Week 1 - SSH Server Hardening and Fail2Ban

Week 1 I set up and hardened my SSH server on my Ubuntu VM. I changed the default port, disabled root login, set up key authentication,  disabled password authentication, and configured fail2ban to automatically ban IPs that fail to login or authenticate too many times.





## Week 2 - Linux User Management and Permissions

Week 2 I learned about Linux user management, file permissions, ownership, ACLs, and the file system attributes files that underlie everything. Includes intentional permission breaking and recovery.



## Week 3 - Firewall Configuration and Network Security

This week I learned about UFW configuration, iptables and nftables architecture, tcpdump packet capture analysis, and attack testing from my Kali Linux VM to verify defenses from the outside and watched attack patterns from a defenders perspective.





## Lab Environment

- Windows 11 (SSH client) - Windows host where I run these two VMs below using Oracle Virtual Box, ssh into my server, and update my GitHub repositories.

- Ubuntu VM (target server) -  Ubuntu  Server VM that I hardened and protected

- Kali Linux VM (attacker machine) -  Kali VM used to simulate being an attacker, and to see the holes in my defense from an attackers point of view, and make my server more secure. 

- Oracle Virtual Box (virtualization) 



## What I Learned and what surprised me 

- ARP traffic is constant on the wire because ip address are constantly changing as device leave and join the network. The ARP cache uses a TTL (time to live), and entries expire deliberately so the mapping stays accurate and device can communicate with each other.

- A closed nmap result had two different causes, one was because no service was listening so kernel sent TCP reset, and the other was because I had a firewall reject rule so the kernel also sent a TCP reset. These two cases where indistinguishable from the defenders perspective on the system as they both produce the same TCP RST.  






