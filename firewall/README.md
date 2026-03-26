\# Firewall Configuration Documentation



\## Current UFW Rules

I ran this command to see my current ufw rules:"sudo ufw status verbose"



These were the rules that were displayed to me below:



Port 2222/tcp - LIMIT



LIMIT means allow connections but limit attempts from a user to 6 attempts within 30 seconds. In my case if the user has more than 6 attempts in the timespan of 30 seconds then the ip address is banned at the firewall level temporarily but it resets automatically. This prevents legitimate users from being permanently locked out if they mistype their password a few times quickly.



Port 80/tcp - ALLOW



Port 80 is the standard rule for HTTP web traffic and I added an allow rule for port 80 because I have an nginx server running on it. So without this rule any HTTP requests to my server would be silently dropped by default deny rule.



Port 443/tcp - ALLOW



I have no service running on port 443 but I used it in the lab as a test to see what happens when I have a port open but no service is listening on that port. I saw that the firewall allows the packet through but since nothing is listening the kernel sends back a RST to tell the client that nothing is there. RST stands for reset and I like to think of it as the network equivalent of hanging up the phone abruptly.





Port 23/tcp - DENY



I also have no service running on port 23 but I added this rule to see what happens when I have an explicit deny rule. I learned that with this deny rule the firewall drops the incoming traffic silently and the client trying to connect eventually times out because they got no response from server making the server appear invisible to the connecting user similar to the default deny behavior. I also saw that when I changed to a reject rule the firewall actively sent back a response telling the client it was declined.



Default incoming - Deny



This rule causes all incoming traffic that is not explicitly allowed into my server to be denied automatically.

Explicitly allowed means if there is no allow rule for that specific port then traffic to that port is denied meaning the traffic is dropped by the firewall.





Default outgoing - ALLOW



This rule allows traffic from my server to reach out to the internet freely without the firewall denying or blocking it. For example downloading updates with apt.





\## What each rule prevents:



Port 2222 LIMIT prevents:
Without this rule an attacker could run an automated script trying thousands of passwords per second. The limit slows this down enough to make brute force attacks a little more impractical.



Default deny incoming prevents:
Without this rule every port on my server would be reachable by default and an attacker could probe for vulnerable services running on unexpected ports.



\## The Layered Defense Stack

Every connection attempt to my ssh server passes through multiple checkpoints in order and if any checkpoint fails the connection is blocked and logged. This is called defense in depth.



Someone tries to connect to my ssh server

&#x20;                |

CHECKPOINT 1 - nftables firewall

Is this port allowed?

No -> dropped, logged in ufw.log

yes -> keep going

&#x20;                |

CHECKPOINT 2 - sshd checks AllowUsers

Is this username on the allowed list?

No -> rejected, logged in auth.log

yes -> keep going

&#x20;               |

CHECKPOINT 3 - sshd checks PermitRootLogin

Is this person trying to login as root?

Yes -> rejected, logged in auth.log

No -> keep going

&#x20;               |

CHECKPOINT 4 - key authentication

The ssh server asks does their private key match my public key?

No -> failure logged, Fail2Ban watching

yes -> connected

&#x20;               |

CHECKPOINT 5 - Fail2Ban

Has the IP failed too many times?

yes -> IP banned at firewall level



\## Log Locations



UFW blocks and allows:

/var/log/ufw.log



SSH authentication events:

/var/log/auth.log



Fail2Ban decisions:
/var/log/fail2ban.log



\## Verification Commands

sudo ufw status verbose -> see all ufw active rules

sudo nft list ruleset -> see what the kernel is actually enforcing

sudo fail2ban-client status sshd -> checks banned IPs



\## tcpdump



\### What is tcpdump

\-I like to think of tcpdump as a live security camera pointed at my network cable. Every packet that passes in and out of my machine get captured and shown to me in real time. Tcpdump lets me see exactly what is happening at the network level not just if a connection worked or not but it shows me the actual conversation between machines. Also just like a security camera tcpdump just records what's happening and shows me it doesn't stop or alter any packets that leave or enter my machine.



\### What I used it for

I used tcpdump to see the tcp three way handshake that occurs before a connection is established

&#x20;

### The TCP handshake I observed

For me to observe the TCP three way handshake I used my client to connect to my server on an open port with a listening service like my ssh server. First my client sent a SYN packet \[S] which means synchronize this is my client saying that it wants to connect. Then my server responded with a SYN-ACK packet \[S.] which means I hear you and I am ready, and then my client sent an ACK\[.] packet which means great let's go. After those three steps the connection was established and data could start flowing.



\### RST vs FIN

Now that I have a connection established I then exited to see what happens when I end the connection to my ssh server. I saw that my ssh session was closed by my client by sending a RST ACK \[R.] packet instead of a FIN ACK \[F.] packet that I was expecting because I taught connects close by sending FIN ACK \[F.] packets. I later learned that they both serve the same purpose which to end a connection but RST ACK\[R.] ends the connection abruptly and FIN ACK\[F.] ends the connection politely as bot sides sends a FIN ACK and then the client ends it with and ACK\[.].



\### Commands I used

\-sudo tcpdump -i any -n port 2222 -> capture ssh traffic only, on any network interface ,and -n for no hostname lookup as this is faster and sometimes it gives me the wrong hostname or doesn't work at all. 

\-sudo tcpdump -i network interface -n -w /tmp/filename.pcap -> capture ssh traffic on a specific network interface and save the output to a .pcap file to be later analyzed by Wireshark.

\-sudo tcpdump -n -r /tmp/filename.pcap displays the .pcap file contents in the terminal

\-sudo tcpdump





\### What tcpdump proved





\## Testing From Kali

I tested using ping first with the command "ping -c 4 ipaddress" to confirm that my Kali VM could reach my Ubuntu server at the network level before testing specific ports and it could as 4 packets were sent and 4 were received. If ping failed I would know that the problem was a network connectivity issue rather than a firewall issue. I used the -c flag so that ping only sends a certain amount of icmp echo requests and stops automatically without the need to be stopped manually.



I tested on my Kali VM using netcat. I used netcat because it sees if a connection can be made by sending tcp packets to an address and port and tells me the results. I used "nc -vz -w 3 ipaddress portnumber" to see  what response I would get from a specific port . The reason I used the -v flag which stands for verbose was so that netcat sends me the results with extra details, the -z flag which stands for zero I/O mode so that netcat just tells me if the connection can be made but doesn't send any data, the -w flag determines how long I want netcat to wait before timing out the connection in this case it was 3 seconds. I tried it on a few ports and netcat showed me open for ports that were allowed and also had a service listening on it, timed out for all the ports that I didn't have a firewall rule for. I even tested it on my 23/tcp deny in rule, and I saw that the connection got timed out, but when I changed it to a reject rule netcat showed me that the connection was refused. I learned that this was because when I add a firewall rule to deny traffic to a port the firewall drops the traffic and never responds causing the client to timeout whereas with a reject rule the firewall declines the traffic and sends a response so the client knows it was declined.







I also tested using Nmap which stands for Network Mapper. How Nmap works is it starts the three way tcp handshake by sending a syn packet to the server. I learned that this is called a half open scan or syn scan because Nmap just waits to see if it gets a response but never finishes the three way handshake.I carried out multiple scans using Nmap I will tell you about each one below:
-I first carried out a regular scan using "nmap ipaddress" and it showed me that I had 2 ports open 2222 and 80 and 2 ports closed 23 and 443 ,and the usual services that run on these ports but it didn't give me the exact service that I had running on those ports at that time.

\-Secondly I carried out an Operating system scan using "nmap -O ipaddress" and it gave me some guess of some operating systems that it thought that I was running but none of them were correct.

\-Thirdly I used "nmap -sV ipaddress" for a service version scan and nmap was able to tell me the exact version of services running on my ports. This matters because if an attacker knows the exact version of a service they can look up known vulnerabilities for that specific version and this is why I learned that keeping my services updated is important.

\-Lastly I did an aggressive Scan using "nmap -A ipaddress". The -A flag combines OS detection, version detection, script scanning and traceroute all in one scan and because of this when I ran this scan I had all the info given to me from all the previous scans and more. I learned that this is a scan an attacker would run to gather as much info as it could on a possible target.



By using all these different scans I learned the reason the port 443 showed up as closed even though it was open was because there was no service listening and, I saw just like with netcat that when port 23 had a deny rule it just was added to the filtered ports because the traffic was dropped ,and Nmap got no response but when it had a reject rule it showed that port 23 was closed because of the same reason as netcat.

