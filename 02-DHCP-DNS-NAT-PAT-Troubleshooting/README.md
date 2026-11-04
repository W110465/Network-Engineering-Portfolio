Cisco 2921 — DHCP, DNS, and NAT/PAT Home Lab

Project Summary

I set up my Cisco 2921 router to run a separate network in my home lab. The router handles DHCP for my devices, provides DNS settings, and connects the lab to my existing home internet through NAT/PAT.

The main problem came during testing. My phone connected to the lab Wi-Fi and received an IP address, but websites would not load. I checked the network configuration until I found the NAT rule that was causing the issue.

Equipment

Cisco 2921 router, Cisco switch, wireless access point, home internet router, and smartphone.

Network Setup

Device / Interface

IP Address

Home router (upstream gateway)

10.0.0.1

Cisco G0/2 (outside)

10.0.0.160

Cisco G0/0 (inside)

192.168.1.1

Phone (DHCP client)

192.168.1.7
<img width="1440" height="1920" alt="HOMELAB" src="https://github.com/user-attachments/assets/9a3361e8-7be9-41e2-baad-5f0757f15e90" />

I connected G0/2 to my home router and G0/0 to my Cisco switch. G0/0 uses a static address because it is the gateway for my lab. G0/2 receives its address from my home router through DHCP.

interface GigabitEthernet0/0

 ip address 192.168.1.1 255.255.255.0

 no shutdown

interface GigabitEthernet0/2

 ip address dhcp

 no shutdown

I ran show ip interface brief and confirmed that both interfaces were up. G0/2 had received 10.0.0.160.

DHCP and DNS

I configured a DHCP pool called SERVER for the 192.168.1.0/24 network.

ip dhcp pool SERVER

 network 192.168.1.0 255.255.255.0

 default-router 192.168.1.1

 dns-server 1.1.1.1 8.8.8.8

Using show ip dhcp binding, I saw several assigned addresses. My phone received 192.168.1.7, and its gateway was 192.168.1.1.

NAT/PAT Setup

I marked G0/0 as the inside interface and G0/2 as the outside interface. I also created ACL 1 to match the lab subnet.

interface GigabitEthernet0/0

 ip nat inside

interface GigabitEthernet0/2

 ip nat outside

access-list 1 permit 192.168.1.0 0.0.0.255

The Connection Problem

My phone showed that it was connected to Wi-Fi, but I could not open websites. I first suspected DNS, so I checked the DHCP pool and set the DNS servers to 1.1.1.1 and 8.8.8.8. The problem remained.

I checked show ip interface brief and found both router interfaces up. Next, show ip route showed a default route pointing to my home router at 10.0.0.1.

I pinged 1.1.1.1 from the Cisco router and got a reply. I also pinged my phone at 192.168.1.7, and that worked. The router could reach both the internet and my phone, but the phone still could not browse.

I then checked NAT:

show ip nat translations

show ip nat statistics

The translation table was empty. The statistics showed:

Total active translations: 0

Peak translations: 0

Hits: 0

Misses: 0

ACL 1 was correct when I checked it with show access-lists 1. But the NAT statistics showed an Outside Destination mapping. I reviewed the NAT command and found that the inside-source PAT rule was missing or incorrect.

What Fixed It

I entered this command:

ip nat inside source list 1 interface GigabitEthernet0/2 overload

After that, I tried the phone again and websites loaded. The command let devices on the 192.168.1.0/24 network share G0/2’s address for outbound connections.

Result

The lab network now has working DHCP, DNS settings, and internet access through PAT. The most useful part of the project was the troubleshooting. I learned that a device can connect to Wi-Fi, get an IP address, and reach its gateway while still having no internet access. Checking the router, DHCP leases, routing table, ACL, and NAT statistics helped me find the actual problem instead of continuing to change DNS settings.
