# Cisco 2921 — DHCP, DNS, NAT/PAT Configuration and Troubleshooting

## What I Was Working On

I was configuring my physical Cisco 2921 router to provide internet access to devices connected to my home lab.

I wanted my Cisco router to assign IP addresses automatically using DHCP, provide DNS servers, and use NAT/PAT to allow multiple devices to access the internet through my existing home router.

I was able to connect my phone to the lab Wi-Fi, but the internet wasn't working. The phone received an IP address and showed that it was connected, but websites wouldn't load.

I had to troubleshoot the configuration to figure out what was missing.

## My Network Setup

I used the following equipment:

- Cisco 2921 router
- Cisco switch
- Wireless access point
- Home internet router
- Smartphone for testing

My network was configured as follows:

| Device / Interface | IP Address | Purpose |
|---|---|---|
| Home router | 10.0.0.1 | Internet gateway |
| Cisco G0/2 | 10.0.0.160 | Outside interface |
| Cisco G0/0 | 192.168.1.1 | Inside interface |
| Smartphone | 192.168.1.7 | DHCP client |

## Configuring the Router Interfaces

I assigned a static IP address to G0/0 because it was the gateway for my lab devices.

```cisco
configure terminal
interface GigabitEthernet0/0
ip address 192.168.1.1 255.255.255.0
no shutdown
exit
```

For G0/2, I used DHCP because it was connected to my home internet router.

```cisco
interface GigabitEthernet0/2
ip address dhcp
no shutdown
exit
```

I verified the configuration using:

```cisco
show ip interface brief
```

G0/0 was up with `192.168.1.1`, and G0/2 received `10.0.0.160` from my home router.

## Configuring DHCP and DNS

I configured a DHCP pool named `SERVER` to assign addresses to my lab devices.

```cisco
ip dhcp pool SERVER
network 192.168.1.0 255.255.255.0
default-router 192.168.1.1
dns-server 1.1.1.1 8.8.8.8
```

I checked the assigned addresses with:

```cisco
show ip dhcp binding
```

The router showed several DHCP leases, including `192.168.1.7`, which belonged to my phone.

I also checked the phone's network settings and confirmed that its default gateway was `192.168.1.1`.

This showed me that DHCP was assigning addresses correctly.

## Configuring NAT/PAT

I configured G0/0 as the NAT inside interface and G0/2 as the NAT outside interface.

```cisco
interface GigabitEthernet0/0
ip nat inside
exit

interface GigabitEthernet0/2
ip nat outside
exit
```

I created Access List 1 to identify the network that needed NAT translation.

```cisco
access-list 1 permit 192.168.1.0 0.0.0.255
```

The purpose of PAT was to allow multiple devices on my lab network to share the IP address assigned to G0/2 when accessing the internet.

## The Problem I Encountered

After configuring DHCP, DNS, and the router interfaces, my phone connected to the wireless network but couldn't load websites.

At first, I suspected DNS, so I checked the DHCP pool and configured:

```cisco
dns-server 1.1.1.1 8.8.8.8
```

The Wi-Fi connection appeared on my phone, but the internet still didn't work.

I realized I needed to check the network step by step instead of assuming DNS was the problem.

## How I Troubleshot the Problem

**First, I checked the router interfaces.**

```cisco
show ip interface brief
```

Both G0/0 and G0/2 were up and had the correct IP addresses.

**Next, I checked the routing table.**

```cisco
show ip route
```

I confirmed that my Cisco router had a default route pointing to my home router at `10.0.0.1`.

**Then, I tested internet connectivity from the Cisco router.**

```cisco
ping 1.1.1.1
```

The ping worked.

This told me that the Cisco router itself could reach the internet.

**I checked DHCP assignments.**

```cisco
show ip dhcp binding
```

My phone had received `192.168.1.7`, and its gateway was `192.168.1.1`.

I also pinged my phone from the router:

```cisco
ping 192.168.1.7
```

The ping succeeded, confirming that the router could communicate with my phone.

**Finally, I investigated NAT.**

```cisco
show ip nat translations
show ip nat statistics
```

The translation table was empty, and the statistics showed:

```text
Total active translations: 0
Peak translations: 0
Hits: 0
Misses: 0
```

I also verified Access List 1:

```cisco
show access-lists 1
```

The access list correctly permitted my `192.168.1.0/24` network.

One important clue was that my NAT statistics displayed an `Outside Destination` dynamic mapping instead of the inside-source PAT configuration I needed.

## How I Fixed It

After checking the interfaces, DHCP, routing, and ACL, I entered the correct inside-source PAT command:

```cisco
ip nat inside source list 1 interface GigabitEthernet0/2 overload
```

This command allows devices on my inside network to share the outside interface IP address using port translation.

After entering this command, **my phone was finally able to access the internet through my Cisco lab network.**

The issue was not the Wi-Fi connection or the DHCP-assigned gateway. The missing or incorrect inside-source PAT configuration was preventing the lab devices from accessing the internet.

## What I Learned

This project helped me understand the relationship between DHCP, DNS, routing, and NAT.

I learned that a device can successfully connect to Wi-Fi, receive an IP address, and communicate with its default gateway without having internet access.

I also learned that being able to ping the internet from the router doesn't automatically mean devices behind that router can access the internet.

The most valuable part of this project was troubleshooting. Instead of repeatedly changing settings, I checked each part of the network until I identified the problem.

Working with physical Cisco equipment helped me understand how these technologies work together and how to approach connectivity problems in a real network.
