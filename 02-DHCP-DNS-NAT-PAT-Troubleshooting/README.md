
# Cisco 2921 Router — DHCP, DNS, NAT/PAT and Network Troubleshooting

## Project Overview

I set up my Cisco 2921 router to manage a separate network in my home lab. The goal was to have the router assign IP addresses through DHCP, provide DNS settings, and allow connected devices to access the internet using NAT/PAT.

I connected the Cisco router to my existing home router, then connected a Cisco switch and wireless access point to the lab network.

During testing, I ran into an issue where my phone connected to Wi-Fi and received an IP address, but websites would not load. I worked through the router configuration to find out what was preventing internet access.

## Lab Equipment

- Cisco 2921 router
- Cisco switch
- Wireless access point
- Home internet router
- Smartphone for testing

## Network Configuration

| Device / Interface | IP Address | Role |
|---|---|---|
| Home router | 10.0.0.1 | Upstream gateway |
| Cisco G0/2 | 10.0.0.160 | Outside interface |
| Cisco G0/0 | 192.168.1.1 | LAN gateway |
| Smartphone | 192.168.1.7 | DHCP client |

I configured G0/0 with a static IP address of `192.168.1.1/24` for the lab network. G0/2 was configured to receive an address through DHCP from my home router, which assigned it `10.0.0.160/24`.

```cisco
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown

interface GigabitEthernet0/2
 ip address dhcp
 no shutdown
```

## DHCP and DNS Configuration

I created a DHCP pool named `SERVER` so devices on my lab network could receive their network settings automatically.

```cisco
ip dhcp pool SERVER
 network 192.168.1.0 255.255.255.0
 default-router 192.168.1.1
 dns-server 1.1.1.1 8.8.8.8
```

I used `show ip dhcp binding` to verify that the router was assigning addresses. Several devices received leases, including my phone at `192.168.1.7`.

I also checked the phone's network settings and confirmed that its default gateway was `192.168.1.1`.

## NAT/PAT Configuration

I configured G0/0 as the inside interface and G0/2 as the outside interface.

```cisco
interface GigabitEthernet0/0
 ip nat inside

interface GigabitEthernet0/2
 ip nat outside
```

I created an access list to identify the addresses on my lab network that needed NAT translation.

```cisco
access-list 1 permit 192.168.1.0 0.0.0.255
```

The goal was to allow devices on the `192.168.1.0/24` network to share the IP address on G0/2 when accessing the internet.

## Connectivity Problem

After completing the initial configuration, my phone connected to the wireless network, but I couldn't browse the internet.

I initially thought DNS might be the issue, so I checked the DHCP pool and made sure the DNS servers were configured correctly.

The phone still couldn't load websites.

I decided to check the network one component at a time to identify where the connection was failing.

## Troubleshooting

I started by checking the router interfaces with `show ip interface brief`. Both G0/0 and G0/2 were up, and each had the expected IP address.

Next, I checked the routing table using `show ip route`. The router had a default route pointing to `10.0.0.1`, which was the address of my home router.

I then ran `ping 1.1.1.1` from the Cisco router. The ping succeeded, confirming that the router itself could reach the internet.

To check the LAN connection, I reviewed the DHCP bindings and confirmed that my phone had received `192.168.1.7`. I also pinged the phone from the router, and that test succeeded.

At that point, I knew the router could reach the internet and communicate with my phone, but the phone still couldn't access external websites.

I moved on to NAT and ran:

```cisco
show ip nat translations
show ip nat statistics
```

The translation table was empty. The NAT statistics also showed zero active translations and zero hits.

```text
Total active translations: 0
Peak translations: 0
Hits: 0
Misses: 0
```

I checked Access List 1 and confirmed that it permitted the `192.168.1.0/24` network.

I also noticed that the NAT statistics showed an `Outside Destination` mapping. This led me to review the NAT rule more closely.

## Resolution

The problem was resolved after I configured the correct inside-source PAT rule:

```cisco
ip nat inside source list 1 interface GigabitEthernet0/2 overload
```

This allowed devices on my lab network to use G0/2's IP address for outbound connections through PAT.

After applying the command, I tested the connection again from my phone. Websites started loading, and the phone could access the internet through my Cisco lab.

The issue was caused by an incorrect or missing inside-source PAT rule, rather than a failure of DHCP or the router's upstream connection.

## What I Learned

This project gave me practical experience configuring DHCP, DNS, routing, ACLs, and NAT/PAT on a physical Cisco router.

The biggest lesson came from troubleshooting the connectivity problem. My phone had a valid IP address, the correct gateway, and a working connection to the Cisco router, but that wasn't enough to provide internet access.

I also learned that successfully pinging an external address from the router doesn't prove that devices behind it can reach the internet.

By checking the interfaces, routing table, DHCP leases, ACL, and NAT statistics, I was able to narrow down the problem and correct the configuration.

This lab helped me become more comfortable troubleshooting an actual network rather than relying only on commands learned during CCNA study.
