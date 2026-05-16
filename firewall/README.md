# Firewall 
Without a firewall, every service on my server is open to anyone on the internet.  A firewall is a tool at the network level that enforces rules by watching network traffic and deciding what to do with it based on those rules. UFW acts like a translator because it takes the plain human readable firewall rules, and writes them in nftables language so you don't have to learn nftables syntax .

## Current UFW Rules
| Rule | Rule Info | Why |
| --- | --- | --- |
| Port 2200/tcp - LIMIT | LIMIT allows connections but limits attempts of a user to 6 attempts within 30 seconds.| So if the user has more than 6 attempts in the timespan of 30 seconds then the ip address is banned at the firewall level temporarily, but it resets automatically. This prevents legitimate users from being permanently locked out if they mistype their password a few times quickly.|
| Port 80/tcp - ALLOW | Allows traffic to port 80, the standard port for HTTP web traffic | I added an allow rule for port 80 because I have an nginx server running on it. So without this rule any HTTP requests to my server would be silently dropped by default deny rule. |
| Port 443/tcp - ALLOW | Allows traffic into port 443, the standard port for secure web traffic | This is a deliberate test configuration, I have no service running on port 443, this was done  to see the kernels response when a port is open but there is no service listening, but in a production environment I would close this port as it is making my attack surface bigger. |
| Port 23/tcp - REJECT | Rejects traffic going to Port 23, telnet, an unsecure version of ssh because any data that passes through it is sent in plain text.| This is a deliberate test configuration, I don't use telnet, I added this rule to see what happens when I have a reject rule but in a production environment I would not add a rule for this port as any traffic to it would be denied by default and an attacker would not know if there is anything on that port. |
| Default incoming - Deny | Drops all incoming traffic that is not explicitly allowed into my server. | This makes my system more secure because it only lets in traffic that is explicitly allowed in, like a security guard who only lets persons in a building that are on the guest list and declines everyone else entry. |
| Default outgoing - ALLOW | This rule allows traffic from my server to reach out to the internet freely without the firewall denying or blocking it. | It allows the server to send traffic out, so that it can reach out to external services like the package manager so that it can download updates with apt. |


## The Layered Defense Stack
Every connection attempt to my ssh server passes through multiple checkpoints in order and if any checkpoint fails the connection is blocked and logged. This is called defense in depth.

Someone tries to connect to my ssh server



1) CHECKPOINT 1 - nftables firewall

Is this port allowed?

- No -> dropped, logged in ufw.log

- Yes -> keep going



2) CHECKPOINT 2 - sshd checks AllowUsers

Is this username on the allowed list?

- No -> rejected, logged in auth.log

- Yes -> keep going



3) CHECKPOINT 3 - sshd checks PermitRootLogin

Is this person trying to login as root?

- Yes -> rejected, logged in auth.log

- No -> keep going


4) CHECKPOINT 4 - Key Authentication

The ssh server asks does their private key match my public key?

- No -> failure logged, Fail2Ban watching

- Yes -> connected



5) CHECKPOINT 5 - Fail2Ban

Has the IP failed too many times?

- Yes -> IP banned at firewall level

- No -> The connection attempt is established successfully


## Log Locations
``` bash
# UFW blocks and allows
tail -f /var/log/ufw.log

# SSH authentication events
tail -f /var/log/auth.log

# Fail2Ban decisions:
tail -f /var/log/fail2ban.log
```

## Verification Commands
```bash
# Shows all ufw active rules
sudo ufw status verbose  

# See what the kernel is actually enforcing
sudo nft list ruleset 

# Checks banned IPs
sudo fail2ban-client status sshd 

```

## Tcpdump

### What is Tcpdump

I like to think of tcpdump as a live security camera pointed at my network cable. Every packet that passes in and out of my machine get captured and shown to me in real time. Tcpdump lets me see exactly what is happening at the network level not just if a connection worked or not but it shows me the actual conversation between machines, and just like a live security camera tcpdump doesn't stop or alter any packets that leave or enter my machine.

### What I used it for

I used tcpdump to see all of the following: the tcp three way handshake that occurs before a connection is established, what it looks like when I do different nmap scans from the defenders point of view e.g. sV and -A, trying to connect to a blocked port and a port with a reject rule, how it looks when the firewall is dropping traffic, and when a connection is refused or a port is open but no service is listening on that port, to watch the payload in a http packet vs a ssh packet.

### The TCP Handshake that I observed

For me to observe the TCP three way handshake I used my client to connect to my server on an open port with a listening service like my ssh server. First my client sent a SYN packet [S] which means synchronize this is my client saying that it wants to connect. Then my server responded with a SYN-ACK packet [S.] which means I hear you and I am ready, and then my client sent an ACK[.] packet which means great let's go. After those three steps the connection was established and data could start flowing.

### RST vs FIN

Now that I have a connection established I then exited to see what happens when I end the connection to my ssh server. I saw that my ssh session was closed by my client by sending a RST ACK [R.] packet instead of a FIN ACK [F.] packet that I was expecting because I thought connects close by sending FIN ACK [F.] packets. I later learned that they both serve the same purpose which to end a connection but RST ACK[R.] ends the connection abruptly and FIN ACK[F.] ends the connection politely as bot sides sends a FIN ACK and then the client ends it with and ACK[.].

### Commands Used
```bash
# Capture ssh traffic only, on any network interface ,and -n for no hostname lookup as this is faster and sometimes it gives me the wrong hostname or doesn't work at all.
sudo tcpdump -i any -n port 2200 

# Capture ssh traffic on a specific network interface and save the output to a .pcap file to be later analyzed by Wireshark.
sudo tcpdump -i <network_interface> -n -w /tmp/filename.pcap 

# Displays the .pcap file contents in the terminal
sudo tcpdump -n -r /tmp/filename.pcap 

# Capture traffic on all interfaces and stop at a count of 50.
sudo tcpdump -i any -n -c 50 

```

### What Tcpdump Proved

Tcpdump taught me something I could not have learned just by running SSH commands because I could see the actual conversation happening at the packet level in real time. This is important because it tells me if a connection is falling because the firewall is dropping packets like if I see a SYN but not SYN-ACK back, if I see SYN RST back immediately that tells me that either the port exists but there is no service listening or it could mean that the port has a reject rule. This makes these two different situation indistinguishable from the defenders perspective because on my system for a reject rule and when there is no service listening both produce a TCP RST because my kernel is configured to reject with a tcp reset.

## Testing From Kali 

### Ping 
Ping tests connectivity so I used it to see my Kali VM could reach my Ubuntu server at the network level before testing specific ports .If ping failed I would know that the problem was a network connectivity issue rather than a firewall issue. I used the -c flag so that ping only sends a certain amount of icmp echo requests and stops automatically without the need to be stopped manually.

| Kali ping command | Response | Defenders Perspective (tcpdump)|
| --- | --- | --- |
|```ping -c 4 <target_machine_ip>``` | 4 packets were sent and 4 were received | I saw 4 ICMP echo request were sent and 4 ICMP echo replies were sent back this tell me that the machine can be reached on the network. |

### Netcat
Netcat sees if a connection can be made by sending tcp packets to an address and port and tells me the results. The reason I used the -v flag which stands for verbose was so that netcat sends me the results with extra details, the -z flag which stands for zero I/O mode so that netcat just tells me if the connection can be made but doesn't send any data, the -w flag determines how long I want netcat to wait before timing out the connection in this case it was 3 seconds. 

| Kali netcat command | Response | Defenders Perspective (tcpdump)|
| --- | --- | --- |
| ``` nc -vz -w 3 <target_machine_ip> 2200  ``` | netcat showed me the connection was open |I saw a tcp handshake being established so netcat sent a SYN[S] packet , a SYN ACK packet [S.] was sent back, and then netcat sent a ACK [.], netcat ending the connection with a RST [R]. |
| ``` nc -vz -w 3 <target_machine_ip> 5000  ``` | netcat showed me that the connection was timed out | I saw an attempt to start the tcp three way handshake so netcat sent a SYN[S] packet but the packets were dropped and it got no respose so it timed out. |
| ``` nc -vz -w 3 <target_machine_ip> 23  ``` | netcat showed me that the connection was refused | netcat sent a SYN[S] packet, the packet was rejected with a RST ACK [R.] from the kernel. |
| ``` nc -vz -w 3 <target_machine_ip> 443  ``` | netcat showed me that the connection was refused | netcat sent a SYN[S] packet to try and establish a connection but the packets were rejected and a RST ACK packet [R.] was sent back. |


### Nmap
I tested these ports using nmap in my Kali VM and watched what was happening on live on my network using tcpdump. Nmap stands for Network Mapper. How Nmap works is it starts the three way tcp handshake by sending a SYN [S] packet to the server. This is called a half open scan or syn scan because Nmap just waits to see if it gets a response but never finishes the tcp three way handshake.

| The command | Port  State |  Tcpdump Live Output | Reason |
| --- | --- | --- | --- |
|  nmap -p 2200 <target_ip_address>  | Open | nmap sent SYN [S] , a SYN ACK [S.] was sent back and RST [R.] was sent by nmap | A  SYN [S] packet was sent first  to start the tcp three way handshake, then a SYN ACK [S.] was sent back and then nmap ending the tcp handshake before the connection was formed with RST [R] successfully carrying out a half open scan. |
|  nmap -p 443 <target_ip_address>  | Closed |  nmap sends SYN [S], and gets back a RST ACK [R.] | A SYN is sent to start the tcp three way handshake but a RST ACK [R.] is sent back from the kernel telling nmap that there is no service listening on that port. |
|  nmap -p 5000 <target_ip_address>  | Filtered | Nmap send two SYN [S] packets and the connection times out | Nmap tries to establish to tcp three way handshake but it is being dropped by the firewall so it eventually times out because it does not get a response. |
|  nmap -p 23 <target_ip_address> | Closed | Nmap sends a SYN [S] and gets a response with RST ACK [R.] | Nmap starts the tcp three way handshake and the connection is rejected by the firewall and the kernel sends back a RST ACK [R.] nmap, because nmap determines port state by whether it got a response or not, both a reject rule and no service listening report as closed because both sends a RST response  back. |

### Nmap Scan Types and Why They Matter
1) An Operating system scan using ```nmap -O <target_machine_ip> ``` . Nmap outputted guesses of some operating systems it thought I was running but none of them were correct and because of this nmap cannot accurately identify my OS, so an attacker cannot look up OS specific vulnerabilities to exploit my operating system.

2) A service version scan using ``` nmap -sV <target_machine_ip> ```. Nmap was able to tell me the exact version of services running on my ports. This matters because if an attacker knows the exact version of a service they can look up known vulnerabilities for that specific version and this is why keeping my services updated is important.

3) An aggressive scan ```nmap -A <target_machine_ip>```. The -A flag combines OS detection, version detection, script scanning and traceroute all in one scan, and because of this when I ran this scan I had all the info given to me from all the previous scans and more. I learned that this is a scan an attacker would run to gather as much info as it could on a possible target. From a defenders perspective it shows you the importance of keeping things up to date and properly secure.

## Issues Encountered
I was unable to carry out the test to scan the open port my ssh server is  listening on at first because the ssh server was not able to start. The reason was because I need to create a directory  /run/sshd which I did but because directories and files in /run are not persistent by default they are lost during reboot so I would encounter the same problem again if the system reboots. So to fix this I created a sshd.conf file at the path /etc/tmpfiles.d/sshd.conf because that is where user custom files are stored, and added the text "d /run/sshd 0755 root root - -" to the file which means to create the directory if it doesn't exist at that file path with the permissions, owner, and group specified. When I rebooted my VM ssh started automatically and there was no problems. Also file in /etc/ override system defaults.

