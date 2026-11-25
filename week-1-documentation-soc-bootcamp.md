# WEEK 1 DOCUMENTATION: SOC bootcamp with cybercarerfair

**pyth490ras** · 7 min read · Oct 14, 2026

Tags: Home Lab, Soc

---

Just completed week one of my SOC bootcamp with cyber career fair, and here’s my documentation for the first week’s task.

Started by building my lab, which comprises of 3 virtual machines; a windows endpoint — which will serve as the target, a kali linux machine — which will serve as the target, and an ubuntu server — which will be used to collect and analyze log data, it’ll host a SIEM tool.

Oracle virtualbox is my preferred hypervisor. And since I want VM-VM communication, as well as VM-internet communication, I used the “NAT network” network adapter. Below is an image of all the various adapters in virtualbox and their features.

![NOTE: This is based on default settings. NAT and NAT Network can allow host-to-VM access via port forwarding.](https://miro.medium.com/v2/resize:fit:1400/1*KULdAJpOdxg-Y5lMccS-pQ.jpeg)
*NOTE: This is based on default settings. NAT and NAT Network can allow host-to-VM access via port forwarding.*

Since I only have 3 VMs, I decided to use /29 subnet, that provides 2³-2=6 usable IP addresses; one gateway, 3 VMs. /29 is just the perfect network to be in.

Having estblished how I want to assign these IP addresses, it is time to actually assign it.

Gteway → 192.168.0.1, Windows →192.168.0.2, Ubuntu → 192.168.0.3, Kali → 192.168.0.4, DNS → 8.8.8.8.

On linux, nmcli will get the job done by statically configuring the IP address permanently, the command is:

<em>sudo nmcli connection add type ethernet \<br>
 ifname eth0 con-name bootcamp-static \<br>
 ipv4.method manual ipv4.addresses 192.168.0.3/29 \<br>
 ipv4.gateway 192.168.0.1 ipv4.dns 8.8.8.8 \<br>
 ipv4.ignore-auto-dns yes ipv6.method disabled</em>

Followed by:

*nmcli connection up bootcamp-static*

For windows, *win+R*, then typing in *ncpa.cpl* gets it done. Right clicked Ethernet and selected properties, double-clicked on ipv4 and filled in the stuffs.

At the end, I checked to ensure that the configuration worked using:

![](https://miro.medium.com/v2/resize:fit:1400/1*DpOMfgt7X5qKqyi2w10Bag.png)

Now, it was time to reboot the VMs and test connectivity between all 3 using ping. And so, I had to ping 6 different ways: kali →ubuntu, ubuntu →kali, kali →windows, windows →kali,ubuntu →windows, windows →ubuntu. That is because successful ping in one direction does not guarantee success in the other direction.

*NB: I had to add firewall rules in microsoft before I could get it to receive inbound packets.*

And as a result, my network diagram looks like:

![](https://miro.medium.com/v2/resize:fit:1254/1*Y6oM1JvwTUcKsy8doRYdpA.jpeg)

Having completed my lab setup, the next thing I had to do is establish a baseline for what normal activity looks likeby inspecting 5 process: wininit.exe, services.exe, lsass.exe, svchost.exe, explorer.exe. These aren’t just random processes, they’re common targets, and some hold critical data. Identified the path the executable is in, the expected prent process, and a few more attributes. Using:

*wmic process get name, processid, parentprocessid, executablepath*

in cmd.

![](https://miro.medium.com/v2/resize:fit:1400/1*cJkOw4tzDHZtT3axQHPndA.jpeg)

Correlated the output with the output of task manager to get a clearer picture.

![These are just 5 of tens of processes that run by default](https://miro.medium.com/v2/resize:fit:1400/1*qPNQgesIYEi1z5eXD1GDAA.jpeg)
*These are just 5 of tens of processes that run by default*

I then looked at services. Picked 4services that use ports for communication and documented.

*netsat -ano* will list all active connections and listening ports, along with the PID of the process using the port.

![](https://miro.medium.com/v2/resize:fit:1400/1*7OK_GYnCC0C5jWrdhqF9Kg.jpeg)

And now, to my Ubuntu server. After setting up ssh, I created an ssh key because I don’t want to keep typing password each time. An ssh key can be created using:

*ssh-keygen -t ed25519 -C “username@domain”*

I then copied the public key to the server, uncommented this line:

*UsePAM yes*

And added the name of my key file to this line:

*AuthorizedKeysFile .ssh/authorized_keys .ssh/id_ed25519.pub*

And now, I can run: *ssh user1@192.168.0.3 -i ~/.ssh/id_ed25519*

Now that that’s done, it is time for some linux admiistration. Decided to create two users (analyst1 and analyst2) and a group (socanalyst).

![](https://miro.medium.com/v2/resize:fit:1400/1*_H8E13zzkdMZ--SBJet1qA.jpeg)

Used adduser because it handles things like password and user home directory creation for me, as opposed to useradd.

I then demonstrated linux permission by creating a file in /var/tmp (used to store temporary files that need to persist across reboots, unlike /tmp which does not persist across reboot). Modified the group ownership of the file to soc_analyst and the permission to 640. 6 (4+2=read+write)represents the permission of the owner, 4 (read) represents the permission of the group, while 0 (nothing) represents that of other. I then tried to read the content of the file as both analyst1 and analyst2. And as expected, analyst2 got “permission denied”, this is because they’re not a member of soc_analyst group. Also, analyst1 can not edit the file, since they only have read permission.

![](https://miro.medium.com/v2/resize:fit:1400/1*ZuErmyCIuCVjUOH3jYgb2g.jpeg)

Now, it is time for some failed logins, tried a few failed logins on the system too.

![](https://miro.medium.com/v2/resize:fit:1040/1*6rvOK935AjzgCko0Ce_d1g.png)

And now, it is time to hunt for logs.

/var/log/auth.log is one very resourceful source for authentication logs, and events relating to sudo. However, I prefer jornalctl

To check for authentication logs using

<em>sudo journalctl _SYSTEMD_UNIT=ssh.service _SYSTEMD_UNIT=sshd.service + SYSLOG_IDENTIFIER=su — grep=”Accepted|Failed|Invalid|session opened|authentication failure”</em>

![](https://miro.medium.com/v2/resize:fit:1400/1*2LElaYHiDcgT7VXyu34AdQ.jpeg)

And now, we can see that an IP tried to first login as an invalid user john, then root, then user1 which happens to be correct. After gaining access to ***user1*** account, the attacker moved to ***analyst1*** account using *su analyst1*.

Similarly, I also have some logs in windows event viewer. That can be opened using *win+R*, *eventvwr.msc.*

![Filtered for event ID 4624,4625 (events correlating to authentication)](https://miro.medium.com/v2/resize:fit:1400/1*-p5cx6DJz_EV0PQgZIfnFw.jpeg)
*Filtered for event ID 4624,4625 (events correlating to authentication)*

Now, I had to capture and analyze network traffic. Selected protocols include: ICMP, DNS, TCP, TLS.

![](https://miro.medium.com/v2/resize:fit:1400/1*0w70pcvHIu2NuZcm7h79JA.png)

And here’s captured ICMP packet. Of course without a source/destination port (ICMP). We also have some flags in there, MAC addresses, and a 40 byte data (which is ised to calculate the RTT, receiver returns the exact same data). So now we know what a real ICMP packet looks like.

Next is DNS, since DNS uses UDP (and TCP), it has source and destination port.

![](https://miro.medium.com/v2/resize:fit:1400/1*4ws6LezmKZcTQl32s9J7WQ.png)

And for DNS, we equally have lots of flags as well. Note that the transaction ID are the same for the query and response, and the response only contains Queries and Answers. A DNS server sending responses whose transaction ID cannot be mapped to any query might be a sign of malicious activity.

![](https://miro.medium.com/v2/resize:fit:1400/1*Eoso2bgRW2rxN-XpEelVbw.jpeg)

ANd for TCP, as usual, we establish a connection first (3 way handshake) before the specific protocol we want to use takes over (in this case, TLS). Disconnection also follows a 4-way handshake.

To wrap it all up, I decided to conduct a port scan on my ubuntu server, the scan came back with “22/tcp open” as expected. But proving a scan was conducted is the real goal. I sshed into my server and tried finding logs that can backup my claim, found none. Turns out linux doesn’t log network connection requests by default. I then turned UFW on, but it couldn’t log at the rate at which nmap was sending packets. I resorted into using tcpdump.

![](https://miro.medium.com/v2/resize:fit:1400/1*CQ-ixPH1ni_h_st3vpdc1Q.jpeg)

Getting these SYN packets on multiple random ports which we don’t even host a service on within a fraction of a second is a sure sign of scan. SOmeone is trying to knock on all doors to see which one we answer to.

Finally, comparing the normal baseline with the activity after my controlled activity left a few indicators of compromise:

1. Getting a few failed login is normal, but getting tens with invalid user account tried is a red flag.
2. Requests are normal, but getting hundreds of them per second, from one IP isn’t.
3. Processes creation follows a specific pattern, and they start at a specific location. Any change in that should be looked into.
4. Linux, by default, does not log activities like network connection attempts, unpriviledged command execution, activities that don’t involve a change in user/permission.
