# Week 1 - SSH Hardening
I hardened and deployed an SSH server and configured fail2ban on Ubuntu. The goal was to replace insecure defaults like password auth, root login, default port with a layered defence stack that holds up under real attack conditions.


## The machines used were:
- **Windows 11** - (SSH client)
- **Ubuntu VM**  - SSH server (target) 


## Hardening Changes Made to sshd_config file :
| Change | Why |
|---|---|
|Port 22 to port 2200 | Reduces automated bot noise targeting the default port |
|PermitRootLogin no | Prevent root access and forces name accountability in the logs.
|PasswordAuthentication no | Disables password login to prevent brute force attacks |
|AllowUsers \<username\>| A whitelist that determines what user account can be used to ssh into the server |


## Cryptographic Keys

### What are key pairs and how do they work
Key pairs are a public and private key that are generated using an algorithm (a mathematical formula) that match each other. They are both different but are both mathematically connected. The private key stays on your machine and it should not be shared with anyone, but you can share your public key with anyone.


### What is challenge-response mechanism
Challenge-response mechanism is the puzzle that is created using your public key from the server that can only be solved using your private key on the client and is then verified using your public key.


### Why does the private key never leave my machine.
The private key never leaves my machine because it is used to solve the puzzle that is sent from the server. So if I share my private key with someone else like an attacker now they can solve the puzzle, and get into the server with my account. This is a big security risk as they have all my permissions.

## Fail2Ban
Fail2ban monitors authentication logs and bans IP addresses that exceed the failure threshold by inserting rules directly into the firewall.
- Findtime: is the time window in which failures are counted .
- Maxretry: is the maximum amount of failed attempts an ip address has in the Findtime window before it is banned.
- Bantime: is how long a specific ip address will be banned for. It could be a specific time like 10 mins or indefinitely (using -1).
 
## What I Learned

- The difference between ssh (the client that goes out that tries to connect to the server) and sshd  (the actual ssh server program that listen and accepts incoming connections).

- By experimenting learned that if I change my ssh port number and then restart the ssh server so that it can run and listen for connections on the new port but I don't change the firewall to allow incoming traffic before doing so then I will lock myself out of the ssh server because the firewall will block traffic to the new port for the ssh server because there is no rule to allow traffic to that port. 

- Since fail2ban watches the logs and bans ip addreses by inserting rules into the firewall config file. When the banned ip tries to get into the ssh server the firewall denies it.

- An allowed list prevents any user that is not on it from being used to ssh into the server.

- Most passwords can be cracked (figured out) easily using a brute force attack (a script that tries thousands even millions of passwords to figure out a password).

- Cryptographic keys are way safer than passwords because they are mathematically complex and practically impossible to brute force unlike passwords.

- The server uses challenge response which is when the server sends an encrypted challenge that is decrypted and signed using my private key and then the server verifies the response using my public key that I sent to it.
  
## Unexpected Behavior and Fixes :
- When changing to a custom port in the sshd_config file, allowing traffic from the changed port on the firewall and then restarting sshd, the port that sshd listened on didn't change to the new port. It was because of socket activation, where systemd was listening on the port instead of sshd directly. So I had to disable it using "sudo systemctl disable ssh.socket" and stop it from running using "sudo systemctl stop ssh.socket".

- When using my keys to login to the ssh server it would try a couple private keys but it would never get to my private key that match with the public key that I sent to the server. So I had to use a specific command "ssh -o IdentitiesOnly=yes -i\~/.ssh/<private_key_file_name> username@ip" to specify the private key so I could then enter my private key passphrase and get into the ssh server. This is a manual fix. I also created a config file for the specific host name and I just use the command "ssh hostname" to ssh into the server.

- I tried to check iptables to see if the ip was banned but since new ubuntu versions use nftables I had to go there to see the banned ip.


## Command Reference 
```bash
# "which sshd" looks through your system and tells you where the ssh program lives. If it returns a file path ssh is installed and if it returns nothing ssh is not installed.
which sshd 

# Installs the openssh-server, the tool that lets other machines connect to my machine.  
sudo apt install openssh-server

# Shows if the SSH service was both enabled (starts on boot) and active (running right now). This matters because a service can be installed but still be off.
sudo systemctl status ssh 

# Enables sshd which makes it automatically start on boot.
sudo systemctl enable ssh

# Starts sshd which turns sshd on right now.
sudo systemctl start ssh

# Ssh into using the username and ip address with the default port 
ssh <username@ip>

# Shows the username of the account I am logged in as in my Ubuntu VM.
whoami

# "hostname -I" shows the ip address. The ip address that I use in my ssh command.
hostname -I 

# Opens nano text editor with my sshd_config file which contains all my sshd configurations. This is where I change the rules to harden my ssh server.
sudo nano /etc/sshd/sshd_cofig

# Checks if the sshd_config file had any syntax errors. Do this every time you make changes to the sshd_config file before reloading or restarting so that you don't lock yourself out of the ssh server. If there output with a line number the file has syntax error but if there is no output the file doesn't have syntax errors.
sudo sshd -t

# restarts my ssh server so that it listens for connections on the new port
 sudo systemctl restart ssh 

# Disables ssh.socket. If ssh.socket activation is enabled then when you restart the ssh server, the ssh port wouldn't change to the new port because socket activation which from what I understand uses systemd to listen for on a specific port for incoming connections, and then hand those connection to sshd instead of sshd continuously listening for connections in memory. So I had to disable socket activation using the command below, and then stop it .
sudo systemctl disable ssh.socket

# Stops ssh.socket from running
sudo systemctl stop ssh.socket

# Connects to the ssh server on the custom port. If you change your port include this flag or else
it would assume you were trying to connect to the server on the default port (port 22).
ssh -p<the_new_port_2200> <username@ip>

# Generates a cryptographic key pair and -t is the algorithm used to create the keys. This command can run without -f which is used to specify a file name, and -c which is used to add a comment, without those two flags it would just generate a key with a default file name like id_ed25519 and with no comment.
ssh-keygen -t ed25519 -f <the_file_name> -C "<You_can_put_a_comment_here>" 

# Sends the public key file to the server (PowerShell).
type "$HOME\.ssh\<public_key_file>" | ssh -p <sshd_port_number> <username@your_vm_ip> "mkdir -p ~/.ssh & cat >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys"

#  Outputs the public key in the terminal that was sent to the server. Cat displays the contents in the authorized_keys file in the terminal.
cat ~/.ssh/authorized_keys

# Shows the ssh connection attempt with the most detail which I used to see why my key authentication was not working at first which was because it was trying different private key file names, and not the specific private key to match my public key. 
ssh -vvv -p<the_port_number> <username@ip>

# Specify the specific private key file I wanted to use so that I could connect to the ssh server using my private key. Used this to solve the problem above.
ssh -o IdentitiesOnly=yes -i ~/.ssh/<name_of_private_key_file> -p <the_port_number> <username@ip>

# This forces a password to login to the ssh server. I used this to ensure password auth was disabled.
ssh -o PreferredAuthentications=password -p<the_port_number> <username@ip>

# Installs fail2ban which watches for repeated failed login attempts and automatically bans the ip addresses.
sudo apt install fail2ban

# Checks the fail2ban status to see if it is start on boot and is currently running.
sudo systemctl status fail2ban

# Makes fail2ban automatically starts on boot 
sudo systemctl enable fail2ban

# Makes fail2ban run right now
sudo systemctl start fail2ban

# Creates a .local file. The reason I created a .local file is because .conf files belong to the software package and are overwritten during software updates. So what ever changes I made can be overwritten whereas a .local file belongs to me and won't be overwritten.
sudo nano /etc/fail2ban/jail.local

# shows the fail2ban logs live so I can see what is happening in real time.
sudo tail -f /var/log/fail2ban.log

# shows the status for jail: sshd so I could see the banned ip addresses.
sudo fail2ban-client status sshd

# Unban a specific IP address
sudo fail2ban-client unban <ip>

# Show me the nft firewall rules. Verifies the ban at the firewall level.
sudo nft list ruleset
```


