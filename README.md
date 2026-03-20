## SSH Hardening and Linux Permissions Lab



## What this is

This is my hands on home lab where I practice cybersecurity concepts each week. I document everything I do, why I did it, and what I learned so that I can refer back to it and explain it to anyone.



## Environment

\-Windows 11 (SSH client)

\-Ubuntu VM (target server)

\-Kali Linux VM (attacker machine)



## Week 1 - SSH Server Hardening and Fail2Ban

This week I set up and hardened an SSH server on my Ubuntu VM. I changed the default port, disabled root login, set up key authentication,  disabled password authentication, and configured fail2ban to automatically ban IPs that fail to login too many times.



Full notes and commands -> week1-ssh-hardening.md



## Week 2 - Linux User Management and Permissions

This week I learned how Linux controls who can access what on a system. I learned how permission bits work, how special permissions like SetUID and sticky bits extend the standard model, how ACLs allow fine grained access control, and how to audit a system for dangerous permissions.



Full notes -> week2-permissions-notes.md



