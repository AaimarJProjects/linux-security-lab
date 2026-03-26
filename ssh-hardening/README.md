# This lab deals with configuring and hardening an ssh server and fail2ban



## The machines that were used were:
- Windows 11 (SSH client)

- Ubuntu VM (target server)



## These are all the changes I made in the sshd\_config file :
- Changed the default port 22 to a custom port with a large number above 1023 but below 65535 like port 2222 . The reason I did this was to prevent attackers from trying to access sshd using the default port.
- Disabled root login because root login allows a client to login to the ssh server as root giving them root permission which allows them to do basically anything like delete files and see secret info like private keys, passwords, etc. Also if clients log in as root you won't know who did what in the logs as they would all have the same username, root. By disabling root clients must login as regular users and use sudo for admin permissions, this is better as it shows who did what in the logs.

- Disabled password authentication because it allows clients to log in using a password which can be cracked (figured out) using a brute force attack. Instead I used cryptographic keys to authenticate my client as this is safer because I don't have to enter a password and even if an attacker steals my public key they can't get into the remote machine or server because they still need my private key.

- Added AllowUsers which is a white list ( a list that says which usernames are allowed in) and prevents clients that ssh in with usernames that are not in AllowUsers from connecting to the ssh server.





## What are key pairs and how do they work

- Key pairs are a public and private key that are generated using an algorithm (a mathematical formula) that match each other. They are both different but are both mathematically connected. The private key stays on your machine and it should not be shared with anyone, but you can share your public key with anyone.



## What is challenge-response mechanism

- Challenge-response mechanism is the puzzle that is created using your public key from the server that can only be solved using your private key on the client and is then verified using your public key.



## Why does the private key never leave my machine.

- The private key never leaves my machine because it is used to solve the puzzle that is sent from the server. So if I share my private key with someone else like an attacker now they can solve the puzzle and get into the server with my account which is a big security risk as they have all my permissions.



## Fail2Ban

- Fail2ban is a tool that bans ip addresses by monitoring logs and inserting firewall rules to prevent a user from trying to brute force into the server or remote machine from the same ip.

- Findtime: is the time window in which failures are counted .
- Maxretry: is the maximum amount of failed attempts an ip address has in the Findtime window before it is banned.
- Bantime: is how long a specific ip address will be banned for. It could be a specific time like 10 mins or indefinitely (using -1).



## Firewall Notes

- Ubuntu 22.04+ uses nftables.

- Fail2Ban writes bans to nftables on modern Ubuntu.

- The command used to verify ip address ban in nftables "sudo nft list ruleset".

 

## Testing

- I connected to the ssh server using ssh username@ip and signed in using the password.

- Changed the ssh port from 22 to 2222 then I tried connecting to the ssh server with out specifying a port and I got a connection refused message because ssh was no longer listening on port 22.

- Disabled root login and then I tried to login as root but the ssh server just kept telling me the password was wrong even though it was correct to trick me into thinking that I could login as root. This is called obscuring failure reasons which is a deliberate security feature that hides the real reason for denial so attackers can't tell whether the username exists or if root login is disabled.

- Added an allowed users in sshd\_config, then I tried to login as a different username and I got a permission denied message in the terminal.

- Generated public and private key asymmetric  pair using the ed25519 algorithm. Sent the public key to the Ubuntu VM and logged in to the ssh server using my private key after my terminal asked me to enter the passphrase for my private key.

- I established multiple connection to the ssh server using my private key and as a safety net and also to ensure that my keys were working, left those connections open so that I don't lock my self out of the ssh server and then I disabled password authentication. I tried to force a password login into the ssh server and it told me permission denied (publickey).

- Configured fail2ban jail.local to ban an ip for 10 minutes after 2 failed attempt. I then tried to connect to the ssh server from windows and entered the wrong password 2 times and it kept telling me connection timed out and when I checked the jail for sshd in fail2ban and the nftables file it showed the banned ip.

- I unbanned the ip "sudo fail2ban-client unban 192.etc.." and checked the fail2ban-client status for sshd and the nftables rules and the banned ip was no longer there. To verify I logged in on the same ip that was banned and I got in successfully.

 
## What I Learned

By actually doing this lab the concepts below became clearer:

- I learned the difference between ssh (the client that goes out that tries to connect to the server) and sshd  (the background service that run on the ssh server that listen and accepts incoming connections)

- I learned by experimenting in the lab that if I change my ssh port number and then restart the ssh server so that it can run and listen for connections on the new port but I don't change the firewall to allow incoming traffic before doing so then I will lock myself out of the ssh server because the firewall will block traffic to the new port for the ssh server because there is no rule to allow traffic to that port. 

- I learned that fail2ban watches the logs and bans the ip by inserting rules into the firewall config file so that when the banned  ip tries to get into the ssh server the firewall denies it.

- I learned that an allowed list prevents any user that is not on it from being used to ssh into the server.

- I learned that most passwords can be cracked (figure out) easily using a brute force attack (a script that tries thousands even millions of passwords to figure out a password).

- I learned that cryptographic keys are way safer than passwords because they are mathematically complex and practically impossible to brute force unlike passwords.

- I learned that the server uses challenge response which is when the server sends an encrypted challenge that is decrypted and signed using my private key and then the server verifies the response using my public key that I sent to it.





## Things that surprised me and didn't go as expected in the lab:
- When changing ports in the sshd\_config file, allowing traffic from the changed port on the firewall and then restarting sshd, the port that ssh listened on didn't change to the new port ,and it was because of socket activation, where systemd was listening on the port  instead of sshd directly. So I had to disable it using "sudo systemctl disable --now ssh.socket".

- When using my keys to login to the ssh server it would try a couple private keys but it would never get to my private key that match with the public key that I sent to the server. So I had to use a specific command "ssh -o IdentitiesOnly=yes -i\~/.ssh/(theprivatekey) username@ip" to specify the private key so I could then enter my private key passphrase and get into the ssh server. This is a manual fix. I also created a config file for the specific host name and I just use the command "ssh hostname" to ssh into the server.

- I tried to check iptables to see if the ip was banned but since new ubuntu versions use nftables I had to go there to see the banned ip.







## All the commands I used to complete week 1 lab btw all the commands are in double quotes.


1. First I checked if ssh was installed using "which ssh" because it looks through your system and tells you where the ssh program lives. If it returns a file path ssh is installed and if it returns nothing ssh is not installed.
2. Ssh was automatically installed for me but if it wasn't. I would  install ssh using "sudo apt install ssh" this installs ssh and all of its packages. 
3. Then I checked the ssh status using "sudo systemctl status ssh" to see if the SSH service was both enabled (starts on boot) and active (running right now). This matters because a service can be installed but still be off.
4. Since ssh was disabled for me, I enabled it using  "sudo systemctl enable ssh" to make ssh automatically start on boot .
5. And since ssh was also inactive for me, I activated it using "sudo systemctl start ssh" to turn it on right now.
6. Then I just double checked the ssh status once more using "sudo systemctl status ssh" to ensure both commands worked ,and they did so now ssh is enabled (starts on boot) and active (is turned on right now).
7. So now that I got ssh up and running I needed to find out my username and ip so that I could use my windows 11 client which also had ssh automatically installed to connect to my ubuntu server .Since I was going to use "ssh username@ip" on my ubuntu VM I got my username by running "whoami" this shows the username of the account that my ssh server is running on and I also checked the ip address using "ip addr" and I looked for the line in section 2 that begins with the word inet and shows the ip like 192.168.x.x.
8. Then I connected to my ssh server from windows PowerShell in my windows 11 client using "ssh username@ip" and it asked me if I wanted to continue connecting and I typed yes since it was my ubuntu VM which I trusted but if I was in a production environment then I would verify the signature first to ensure that I was connecting to the correct or intended server. And it worked I was now connected to my ssh server and was able to use my ubuntu terminal as if I was using my ubuntu VM.
9. So now that I am connected I checked the sshd\_config file using "sudo nano /etc/ssh/sshd\_config" and I saw a bunch of commented lines like the port number, password authentication, rootlogin, pubkeyauthentication, etc. I wanted to change commented lines that I listed above so that I could make my server more secure.
10. First I started with changing the port number but before doing this I ensured that I had an active connection from windows client as a safety net to prevent me from locking myself out of the ssh server from my client (the windows 11 machine). I then removed the comment and changed the port number in my sshd\_config file to a port number above the range from 1 to 1023 because those are reserved system ports that require root privileges so I picked a port above 1023 but below 65535 . After I ran "sudo sshd -t" to check if my sshd\_config file had any syntax errors and it didn't output anything meaning the file had no syntax errors if it did then it would output the line number where there was a syntax error. Running "sudo sshd -t" is important as I wouldn't want to restart my ssh server with syntax errors and lock myself out because my server refuses to run as its file has a syntax error. 

11. Then I changed the UFW firewall rules to accept traffic from the new port using "sudo ufw allow newport/tcp" because if I didn't it would time out my connection and I will get locked out of the server. This is because the firewall decides what traffic is allowed to come in or out of my network so if I didn't add that rule, anytime I tried to connect to the ssh server on the new port the firewall would reject me ,and I would be locked out of my ssh server. Since my sshd\_config file has no syntax errors and my firewall accepts traffic from the new port I then restarted the ssh server using "sudo systemctl restart ssh" so that the ssh server runs and listens for connections on the new port.
12. When I restarted the ssh server for some reason ssh didn't change to the new port. Which I then found out that the reason was because socket activation which from what I understand uses systemd to listen for on a specific port for incoming connections and then hand those connection to sshd instead of sshd continuously listening for connections in memory. So I had to disable socket activation using "sudo systemctl disable --now ssh.socket". Finally ssh was now running and listening for connections on the new port. 
13. Now that ssh runs on the new port I tried to connect to it in a new PowerShell window in the terminal using "ssh -p(the new port) username@ip" and I made sure to include -p because if I didn't then it would assume I was trying to connect to the server on the default port (port 22) and because of that it wouldn't connect to the ssh server since it is not listening on port 22.
14. Secondly I wanted to add an allowed list to the sshd\_config file because an allowed list is a whitelist, which is a list that only allows users that are on it into the ssh server. So I opened the sshd\_config file using "sudo nano /etc/ssh/sshd\_config" and then I scrolled down to the bottom of the file and added a new line for AllowUsers then I added my username (the same username I use to connect to the ssh server).Also before continuing I would make sure to ensure the user  that you are connecting to the ssh server is on the allowed list so that you don't lock yourself out. I then saved the changes using "Ctrl O", then enter and exited the nano text editor using "Ctrl X". Then I checked if the file had any syntax errors using "sudo sshd -t" and there was no output so it didn't have any syntax errors. Then I restarted my server using "sudo systemctl restart ssh" to enforce the new changes.
15. To test this I then tried logging in to the ssh server as a different username "ssh -p{ssh server port number} differentusername@ip" and it wouldn't allow me to connect as that user was not on the allowed list.
16. Thirdly I wanted to connect using cryptographic keys instead of a password. This is more convenient as I don't have to remember a bunch of passwords and more secure because most passwords can be easily figured out using brute force attack (this is an attack that uses a script to try and guess your password by trying thousands or even millions of passwords until it figures out your password ).To create cryptographic keys I first needed to generate a public key (the key that I can send to anyone) and private key pair (the key that stays on my machine and needs to be kept private). Since I am connecting using my windows 11 client in PowerShell I generated a public and private key pair using the command "ssh-keygen -t ed25519 -f \[thefilename] -C \[You can put a comment here]".Also I could've ran this command without -f -c it would just generate a key with a default file name like id\_ed25519 and with no comment. I like to use both as I like to put the file name as the name of the username on the server that I am connecting to and the type of algorithm used to create the keys as it helps me remember who these cryptographic keys are for and what algorithm was used. For the comment I just put the username of the account that this key is used in.
17. Then I sent the file in PowerShell to my ubuntu VM (where my ssh server is running) using " Get-Content "$env:USERPROFILE\\.ssh\\id\_ed25519.pub" | ssh username@your\_vm\_ip "mkdir -p \~/.ssh \&\& chmod 700 \~/.ssh \&\& cat >> \~/.ssh/authorized\_keys \&\& chmod 600 \~/.ssh/authorized\_keys" ".
18. I checked the folder where the file was sent using "cat \~/.ssh/authorized\_keys" and I saw my public key. I used cat to display the contents in the authorized\_keys file in the terminal.
19. Now that the ssh server has my public key I tried to login with my private key using "ssh -p(the port number) username@ip", this didn't work for me because I found out that it was using other private keys and not the specific private key that matches my public key so by using "ssh -vvv -p(the port number) username@ip" it showed me that it was trying different private key file names. So I used "ssh -o IdentitiesOnly=yes -i \~/.ssh/nameofprivatekeyfile -p(the port number) username@ip" to specify the specific private key file I wanted to use and it worked as I was able to connect to the ssh server using my private key. I also created a config file in PowerShell in my windows client using "notepad \~\\.ssh\\config". In the config file on the first line I added the host ( the name I wanted to use to connect to the the ssh server), On all the other line below I indented them by four spaces I then added all of these on a new line: the host name and then a space with the ip address of my ubuntu VM, the name of the user account that my ssh server is running on, the port number, the identity file ,and a space with the file path to my private key ,and IdentitiesOnly=yes. Now I can connect to the server using "ssh (host name)" the host name is the name given for the host on the first non indented line. I like to use the config file as it is a shorter command and has all the info in one place.
20. Now that I can login using my cryptographic keys to my ssh server.I am now going to disable password authentication so that attackers can't use brute force attacks to get into my server. First I open an active connection using my keys from windows as a safety net so I don't lock my self out of the server and also to verify once again my keys are working before I disable password authentication.I go to the sshd\_config file using "sudo nano /etc/ssh/sshd\_config" I look for the line with PasswordAuthentication uncomment it (I uncomment the line so that the config file see it as a rule and not a comment) and change it from yes to no. Then I save the changes and exit nano text editor.Once again I use "sudo sshd -t" to ensure the sshd\_config file has no syntax errors and it didn't so I restarted the server using "sudo systemctl restart ssh".I ensured that password authentication was disabled by trying to force ssh to ask for a password using "ssh -o PreferredAuthentications=password -p(the port number) username@ip" and it told me permission denied (publickey) so forcing password login didn't work.
21. Lastly I installed fail2ban using "sudo apt install fail2ban" because fail2ban watches for repeated failed login attempts and automatically bans the ip addresses. I checked if fail2ban was enabled (starts automatically on boot) and active (running right now) using "sudo systemctl status fail2ban".I saw that it was both disabled and inactive and I enabled fail2ban using "sudo systemctl enable fail2ban" to ensure it automatically starts on boot and activated it using " sudo systemctl start fail2ban" to ensure that it is running right now.  
22. I changed the directory to fail2ban using "cd /etc/fail2ban" and I used "ls" to list all the files in fail2ban except for the hidden ones. I created a local file using "sudo nano jail.local". In the jail.local file I added all of the following on separate line : \[sshd], enabled = true, port = (the port number that ssh runs on), logpath = /var/log/auth.log, maxretry = 2, bantime = 10m, findtime = 10m.The reason I created a .local file is that .conf files belong to the software package and are overwritten during software updates. So what ever changes I make can be overwritten whereas a .local file belongs to me and won't be overwritten.
23. Before trying to ssh into the server I opened the logs using "sudo tail -f /var/log/fail2ban.log" so that I could see what was happening in real time. Then in powershell in my windows 11 machine I tried to connect to the server with the wrong username using "ssh -p(the port number) wrongusername@ip". After 2 attempts the connection was being time out instead of permission being denied meaning the firewall was denying traffic from my ip so it never got to reach the ssh server.In the logs I saw the ip banned after two previous login attempts and when I checked the fail2ban-client using "sudo fail2ban-client status sshd" I saw the banned ip in the banned IP list and I also checked if the ip was banned at the firewall level using "sudo nft list ruleset" and I saw the banned ip in the table inet f2b-table shown as elements = {banned ip} meaning the ip is banned at the firewall level so any traffic that comes in from that specific ip will be denied before it can reach the ssh server. I also tried to ssh in with the correct details using"ssh -o IdentitiesOnly=yes -i \~/.ssh/\[your private key name] -p(your ssh server port number) username@ip" and the connection was being timed out because the ip was still banned.
24. I unbanned the ip using "sudo fail2ban-client unban \[banned ip]".Then I checked the fail2ban-client status for sshd using "sudo fail2ban-client status sshd" to see if the ip was still banned in the  banned ip list and it wasn't. Then I checked if the ip was still banned at the firewall level using "sudo nft list ruleset" and I saw the ip was no longer there under table inet f2b-table so that means the ip was no longer banned. Lastly I tried to log in with the correct username using "ssh -o IdentitiesOnly=yes -i \~/.ssh/\[your private key name] -p(your port number) username@ip"and it worked meaning my ip was unbanned.

