# 10 — IP Services: NAT, DHCP, NTP, SNMP, Syslog & QoS

> **CCNA Exam Domain:** 4.0 IP Services  
> **Exam Weight:** ~10% of total exam  
> **Difficulty:** ⭐⭐⭐☆☆

---

## Table of Contents

1. [NAT — Network Address Translation](#1-nat--network-address-translation)
2. [NAT Types — Static, Dynamic, PAT](#2-nat-types--static-dynamic-pat)
3. [NAT Configuration & Verification](#3-nat-configuration--verification)
4. [DHCP — Dynamic Host Configuration Protocol](#4-dhcp--dynamic-host-configuration-protocol)
5. [NTP — Network Time Protocol](#5-ntp--network-time-protocol)
6. [SNMP — Simple Network Management Protocol](#6-snmp--simple-network-management-protocol)
7. [Syslog — System Logging](#7-syslog--system-logging)
8. [FTP and TFTP](#8-ftp-and-tftp)
9. [QoS — Quality of Service](#9-qos--quality-of-service)
10. [Key Takeaways & Exam Tips](#10-key-takeaways--exam-tips)

---

## 1. NAT — Network Address Translation

### The Problem NAT Solves

To understand NAT deeply, you need to understand the crisis that created it. IPv4 provides approximately 4.3 billion addresses (2³²). When the internet was designed in the 1970s and early 1980s, this seemed like an enormous, inexhaustible number. Nobody imagined that billions of devices would eventually need internet connectivity.

By the early 1990s, it became clear that IPv4 would run out — and it was running out fast. Two solutions were developed: the long-term solution was IPv6 (128-bit addresses, essentially unlimited), and the short-term solution was NAT. NAT allowed thousands or millions of devices to share a very small pool of public IP addresses, dramatically extending the life of IPv4.

The fundamental idea is simple. RFC 1918 designates three ranges of IP addresses as "private" — these addresses are not routable on the public internet, meaning internet routers will not forward packets with these source or destination addresses. Any organization can use these addresses internally without coordination with anyone:

```
RFC 1918 Private Address Ranges:
  10.0.0.0    – 10.255.255.255   (10.0.0.0/8)     → 16,777,216 addresses
  172.16.0.0  – 172.31.255.255   (172.16.0.0/12)  → 1,048,576 addresses
  192.168.0.0 – 192.168.255.255  (192.168.0.0/16) → 65,536 addresses

Your home uses 192.168.1.x.
Your neighbor also uses 192.168.1.x.
This is fine — they are both private and never routed on the internet.
```

NAT sits at the boundary between the private network and the internet. When a device with a private IP needs to reach the internet, NAT translates the private source address into a public address that IS routable on the internet. When the reply comes back, NAT translates it back to the private address and delivers it to the correct internal device.

### NAT Terminology — Four Key Terms

The NAT terminology is one of the most confusing parts of the topic because Cisco uses four specific terms that students often mix up. Let's define them precisely with a concrete example.

Imagine PC-A with IP 192.168.1.10 is visiting a web server at 8.8.8.8. Your router's public IP is 203.0.113.1.

**Inside Local** is the private IP address of the internal device as seen from inside the network — from the device's own perspective. In our example, PC-A's Inside Local address is **192.168.1.10**. This is the "real" IP the device thinks it has.

**Inside Global** is the public IP address that represents the internal device as seen from outside — from the internet's perspective. In our example, when PC-A's traffic leaves the router, it appears to come from **203.0.113.1**. The internet sees this address, not the private one.

**Outside Global** is the public IP of the external destination — how the internet and the destination server identify themselves. The web server's Outside Global is **8.8.8.8**.

**Outside Local** is how the external server appears from inside the network. In most NAT configurations this is the same as the Outside Global (8.8.8.8), but in double-NAT scenarios it can differ.

```
                    NAT Translation
Inside              Table              Outside
─────────────────────────────────────────────────────
PC-A                  ↓                Web Server
192.168.1.10  ──► 203.0.113.1  ──────► 8.8.8.8
(Inside Local)   (Inside Global)      (Outside Global)

The translation table entry:
  Inside Local    Inside Global     Outside Global   Protocol  Port
  192.168.1.10    203.0.113.1       8.8.8.8          TCP       80
```

---

## 2. NAT Types — Static, Dynamic, PAT

### Static NAT — One-to-One Permanent Mapping

Static NAT creates a **permanent, fixed, one-to-one mapping** between one specific private IP address and one specific public IP address. This mapping never changes and never expires.

The primary use case for static NAT is when you have an internal server — a web server, mail server, or FTP server — that needs to be reachable from the internet. The server needs a consistent public IP so that DNS records can point to it, and external clients can always find it. Without static NAT, the server's public IP could change, breaking all external access.

```
Static NAT — fixed mapping, always the same:
  Internal web server:  192.168.1.100 (private, not routable)
  Public IP assigned:   203.0.113.10  (public, internet-reachable)
  
  Internet request to 203.0.113.10 → NAT translates → forwards to 192.168.1.100
  Server replies from 192.168.1.100 → NAT translates → appears as 203.0.113.10
  
  This mapping exists 24/7 whether the server is communicating or not.
  One public IP is "consumed" per server — expensive but necessary for servers.
```

### Dynamic NAT — Pool-Based Mapping

Dynamic NAT maintains a **pool of public IP addresses** and assigns them to internal devices on a first-come, first-served basis as they need internet access. When the session ends, the public IP is returned to the pool for another device to use.

The critical limitation of dynamic NAT is that if all public IPs in the pool are in use, new internal devices cannot get internet access — they are simply denied translation. Dynamic NAT works when you have more internal devices than public IPs but not all devices need simultaneous internet access.

```
Dynamic NAT — pool of public IPs, assigned on demand:
  Public IP pool: 203.0.113.10 – 203.0.113.20  (11 public IPs)
  Internal devices: 192.168.1.1 – 192.168.1.254  (254 private IPs)
  
  PC-A (192.168.1.10) opens connection → gets 203.0.113.10 from pool
  PC-B (192.168.1.11) opens connection → gets 203.0.113.11 from pool
  ...
  12th device opens connection → pool EXHAUSTED → connection DENIED!
  
  When PC-A's session ends → 203.0.113.10 returned to pool → available again
```

### PAT — Port Address Translation (NAT Overload)

PAT, also called NAT Overload, is by far the most common form of NAT. This is what your home router uses. PAT maps **many private IP addresses to a single public IP address** by using unique **port numbers** to track each individual session.

To understand why this works, think about what makes a network session unique. A session is identified by the combination of source IP + source port + destination IP + destination port + protocol. If you change only the source IP (replacing private with public) but also assign a unique source port number for each translation, you can have thousands of concurrent sessions all sharing one public IP — because each one has a unique port number.

```
PAT — Many private IPs share ONE public IP using unique ports:

Inside Local         PAT Table (Inside Global)      Outside
─────────────────────────────────────────────────────────────
192.168.1.10:54231   203.0.113.1:10001    →    8.8.8.8:80  (PC-A browsing)
192.168.1.20:54232   203.0.113.1:10002    →    8.8.8.8:80  (PC-B browsing)
192.168.1.30:54233   203.0.113.1:10003    →    8.8.8.8:443 (PC-C HTTPS)
192.168.1.40:54234   203.0.113.1:10004    →    1.1.1.1:53  (PC-D DNS)
192.168.1.50:54235   203.0.113.1:10005    →    8.8.8.8:80  (PC-E browsing)

All 5 devices share ONE public IP: 203.0.113.1
Differentiated by unique port numbers: 10001, 10002, 10003, 10004, 10005

When web server at 8.8.8.8 replies to 203.0.113.1:10002:
  NAT looks up port 10002 in table → finds 192.168.1.20:54232
  Translates destination back to 192.168.1.20
  Forwards to PC-B ✅

Theoretical maximum: 65,535 concurrent sessions per public IP
Practical maximum: much lower (thousands typically)
```

---

## 3. NAT Configuration & Verification

### Identifying Inside and Outside Interfaces

Every NAT configuration begins by telling the router which interfaces face your internal private network (inside) and which face the public internet (outside). NAT only translates packets flowing between inside and outside interfaces — it has no effect on traffic staying entirely within the inside network.

```cisco
! The interface connected to your LAN = INSIDE
Router(config)# interface GigabitEthernet0/0
Router(config-if)# description "LAN Interface - Inside"
Router(config-if)# ip address 192.168.1.1 255.255.255.0
Router(config-if)# ip nat inside          ! ← mark as inside
Router(config-if)# no shutdown

! The interface connected to the internet/ISP = OUTSIDE
Router(config)# interface GigabitEthernet0/1
Router(config-if)# description "WAN Interface - Outside"
Router(config-if)# ip address 203.0.113.1 255.255.255.252
Router(config-if)# ip nat outside         ! ← mark as outside
Router(config-if)# no shutdown
```

### Configuring Static NAT

```cisco
! Create permanent one-to-one mapping
! Syntax: ip nat inside source static [inside-local] [inside-global]
Router(config)# ip nat inside source static 192.168.1.100 203.0.113.10
! 192.168.1.100 = web server's private IP
! 203.0.113.10  = public IP assigned to this server

! For multiple servers, add one line per server:
Router(config)# ip nat inside source static 192.168.1.101 203.0.113.11
Router(config)# ip nat inside source static 192.168.1.102 203.0.113.12

! Static NAT for a specific port (Port Forwarding):
! Forward only HTTP (port 80) from public IP to internal server
Router(config)# ip nat inside source static tcp 192.168.1.100 80 203.0.113.10 80
! Now only port 80 traffic is forwarded — other ports on 203.0.113.10 are not translated
```

### Configuring Dynamic NAT

```cisco
! Step 1: Define the pool of public IP addresses available for translation
Router(config)# ip nat pool PUBLIC_POOL 203.0.113.20 203.0.113.30 netmask 255.255.255.0
!                             pool name    start IP       end IP      subnet mask
! This creates a pool of 11 public IPs: .20 through .30

! Step 2: Define which internal addresses are eligible for translation (ACL)
Router(config)# access-list 1 permit 192.168.1.0 0.0.0.255
! Only devices in 192.168.1.0/24 will be NAT'd

! Step 3: Link the ACL to the pool
Router(config)# ip nat inside source list 1 pool PUBLIC_POOL
! "Translate addresses matching ACL 1 using addresses from PUBLIC_POOL"
```

### Configuring PAT (NAT Overload) — Most Common

```cisco
! Method 1: PAT using the router's own outside interface IP (most common for home/small office)
! Step 1: Define which internal hosts get NAT'd
Router(config)# access-list 1 permit 192.168.1.0 0.0.0.255

! Step 2: Configure PAT using the outside interface IP
Router(config)# ip nat inside source list 1 interface GigabitEthernet0/1 overload
!                                                      ↑ outside interface    ↑ overload = PAT!
! The router's WAN IP (203.0.113.1) becomes the shared public IP for all translations

! Method 2: PAT using a pool of public IPs (when you have more than one public IP)
Router(config)# ip nat pool PAT_POOL 203.0.113.50 203.0.113.55 netmask 255.255.255.0
Router(config)# ip nat inside source list 1 pool PAT_POOL overload
! "overload" keyword = PAT (port overloading)
```

### NAT Verification Commands

```cisco
! View current active NAT translations
Router# show ip nat translations
Pro  Inside global      Inside local       Outside local      Outside global
tcp  203.0.113.1:10001  192.168.1.10:54231 8.8.8.8:80        8.8.8.8:80
tcp  203.0.113.1:10002  192.168.1.20:54232 8.8.8.8:80        8.8.8.8:80
---  203.0.113.10       192.168.1.100      ---                ---       ← static entry (no port)

! View NAT statistics (hit counts, misses, translation table size)
Router# show ip nat statistics
Total active translations: 42 (1 static, 41 dynamic; 41 extended)
Peak translations: 150, occurred 00:05:32 ago
Outside interfaces: GigabitEthernet0/1
Inside interfaces: GigabitEthernet0/0
Hits: 18420  Misses: 3
Expired translations: 1200

! Clear NAT translation table (useful when troubleshooting)
Router# clear ip nat translation *              ! clears ALL dynamic translations
Router# clear ip nat translation inside 192.168.1.10   ! clears specific entry

! Real-time NAT debugging (use carefully — generates lots of output!)
Router# debug ip nat
NAT: s=192.168.1.10->203.0.113.1, d=8.8.8.8 [45231]    ← outbound translation
NAT: s=8.8.8.8, d=203.0.113.1->192.168.1.10 [45231]    ← inbound translation
Router# undebug all                                       ! always turn off after!
```

### NAT Troubleshooting

The most common NAT problems are misconfigured inside/outside interface assignments, the wrong ACL identifying which traffic to translate, and the "ip nat outside" or "ip nat inside" missing from an interface. If NAT is not working, the first thing to check is whether traffic is actually matching the NAT policy by watching the hit counters in `show ip nat statistics`. If "Misses" is increasing but "Hits" is not, packets are reaching the NAT router but not matching any translation rule.

---

## 4. DHCP — Dynamic Host Configuration Protocol

### What DHCP Does and Why It Matters

Imagine manually configuring the IP address, subnet mask, default gateway, and DNS server on every single device in a network of 500 computers. Every time a laptop moves between offices, its configuration would need updating. When the default gateway IP changes, every device needs reconfiguration. DHCP eliminates all of this by allowing devices to automatically request and receive their complete network configuration from a central server.

DHCP is a client-server protocol built on UDP. The client doesn't know its IP yet, so it cannot use TCP (which requires a reliable session). Instead, the client broadcasts (destination 255.255.255.255) from source 0.0.0.0 (it has no IP yet) and the server responds.

### The DORA Process — How DHCP Works Step by Step

The DORA acronym describes the four-message sequence that a client and server exchange to assign an IP address. Understanding each step is important because exam questions frequently ask about this process.

```
CLIENT                                              SERVER
(no IP yet)                                    (192.168.1.1)
    │                                               │
    │── DISCOVER (broadcast) ──────────────────────►│
    │   Src: 0.0.0.0, Dst: 255.255.255.255          │
    │   "Is there a DHCP server? I need an IP!"     │
    │                                               │
    │◄── OFFER (broadcast or unicast) ──────────────│
    │   "I can offer you 192.168.1.50"              │
    │   Includes: IP, mask, gateway, DNS, lease time│
    │                                               │
    │── REQUEST (broadcast) ───────────────────────►│
    │   Src: 0.0.0.0, Dst: 255.255.255.255          │
    │   "I accept 192.168.1.50! (broadcasting so    │
    │    other DHCP servers know I chose this one)" │
    │                                               │
    │◄── ACK (broadcast or unicast) ────────────────│
    │   "Confirmed! 192.168.1.50 is yours"          │
    │   Lease duration confirmed                    │
    │                                               │
    Client configures itself:
      IP:      192.168.1.50
      Mask:    255.255.255.0
      Gateway: 192.168.1.1
      DNS:     8.8.8.8
      Lease:   86400 seconds (24 hours)
```

The REQUEST is sent as a broadcast even though the client already received an OFFER. This is because there might be multiple DHCP servers on the network, and broadcasting the REQUEST lets all of them know which server was chosen — the unchosen servers can then release their offered addresses back to their pools.

### Cisco Router as DHCP Server

A Cisco router can act as a DHCP server for the local subnet, eliminating the need for a dedicated DHCP server in small branch deployments.

```cisco
! Step 1: Exclude addresses that should NOT be assigned by DHCP
! (Reserved for routers, servers, printers with static IPs)
Router(config)# ip dhcp excluded-address 192.168.1.1 192.168.1.20
! Excludes 192.168.1.1 through 192.168.1.20 — these won't be given out

! Step 2: Create a DHCP pool (one pool per subnet)
Router(config)# ip dhcp pool LAN_POOL
Router(dhcp-config)# network 192.168.1.0 255.255.255.0   ! the subnet
Router(dhcp-config)# default-router 192.168.1.1           ! default gateway
Router(dhcp-config)# dns-server 8.8.8.8 8.8.4.4          ! DNS servers (up to 8)
Router(dhcp-config)# domain-name company.local            ! DNS search domain
Router(dhcp-config)# lease 7                              ! 7 days lease (default = 1 day)
Router(dhcp-config)# lease 0 12                           ! or: 0 days, 12 hours

! For multiple VLANs, create separate pools:
Router(config)# ip dhcp pool VLAN20_POOL
Router(dhcp-config)# network 192.168.20.0 255.255.255.0
Router(dhcp-config)# default-router 192.168.20.1
Router(dhcp-config)# dns-server 8.8.8.8

! Verify DHCP operation
Router# show ip dhcp pool                    ! pool configuration and statistics
Router# show ip dhcp binding                 ! current IP-to-MAC assignments
Router# show ip dhcp conflict               ! IPs that caused conflicts
Router# clear ip dhcp binding *             ! clear all bindings (forces re-negotiation)
```

### DHCP Relay — ip helper-address

A critical real-world scenario: the DHCP server is on a different subnet than the clients. Since DHCP DISCOVER is a broadcast, and routers do not forward broadcasts by default, the DHCP server would never receive client requests.

The solution is the **DHCP relay agent**, configured on the router interface closest to the clients using the `ip helper-address` command. When the router receives a DHCP broadcast on an interface with `ip helper-address` configured, it converts the broadcast into a unicast packet addressed directly to the DHCP server, and forwards it. The server receives the request, allocates an address from the appropriate pool (identified by the router's interface IP, called the "giaddr" field in the DHCP packet), and sends the reply back to the router, which then delivers it to the client.

```
                              DHCP Server
                              10.0.0.50
                                 │
[Client] ──[Switch]──[Router Gi0/0]──[Router Gi0/1]──[DHCP Server]
           VLAN 10                   10.0.0.0/30
           192.168.10.0/24

Client sends DHCP Discover (broadcast on 192.168.10.0 segment)
Router Gi0/0 has ip helper-address 10.0.0.50 configured:
  Router intercepts the broadcast
  Converts to unicast: sends to 10.0.0.50
  Adds giaddr = 192.168.10.1 (Router's Gi0/0 IP — tells server which subnet)
  
DHCP Server receives unicast request:
  Sees giaddr = 192.168.10.1 → allocates from 192.168.10.0/24 pool
  Sends reply to 192.168.10.1 (the router)
  
Router forwards reply to client ✅
```

```cisco
! Configure ip helper-address on the interface facing clients
Router(config)# interface GigabitEthernet0/0
Router(config-if)# description "LAN - Client VLAN"
Router(config-if)# ip address 192.168.10.1 255.255.255.0
Router(config-if)# ip helper-address 10.0.0.50    ! DHCP server IP
Router(config-if)# no shutdown

! ip helper-address forwards EIGHT UDP services by default:
! DHCP (67/68), TFTP (69), DNS (53), TACACS (49),
! Time (37), NetBIOS-NS (137), NetBIOS-DGM (138), BootP (67)
```

---

## 5. NTP — Network Time Protocol

### Why Accurate Time Is Critical for Networks

You might wonder why network engineers care so much about the time on routers and switches. The answer is that accurate, synchronized time is foundational to several critical network functions that break when clocks are wrong.

Log correlation is perhaps the most operationally important. When investigating a network incident, engineers look at syslog messages from multiple devices — routers, switches, firewalls, servers — and try to piece together a timeline. If Router-A thinks an event happened at 10:00:00 and Router-B's clock is 5 minutes off, its log says the related event happened at 10:05:00. The logs appear unrelated even though they describe the same incident. With NTP-synchronized clocks, correlation becomes trivial.

Security protocols depend on time accuracy. Kerberos authentication (used in Active Directory environments) requires clocks to be within 5 minutes of each other — a larger skew causes authentication failures. TLS/SSL certificates have validity periods and are rejected if the checking device's clock is outside the certificate's valid timeframe.

Digital signatures and audit trails also depend on accurate timestamps to prove that an action occurred at a specific time. A network device with a wrong clock undermines the integrity of its own security logs.

### NTP Stratum Hierarchy

NTP uses a hierarchical structure to propagate accurate time from primary sources to all network devices. The **stratum number** indicates how many hops away a device is from a reference clock. Lower stratum = closer to the original time source = more accurate.

```
Stratum 0 — Reference Clocks (NOT on the network):
  Atomic clocks, GPS receivers, radio time signals
  These are the primary reference — accuracy within nanoseconds
  They are connected directly to Stratum 1 servers

Stratum 1 — Primary Time Servers:
  Directly connected to Stratum 0 reference
  Examples: time.nist.gov, pool.ntp.org servers
  Accuracy: within microseconds of UTC

Stratum 2 — Secondary Servers:
  Synchronized from Stratum 1 via NTP
  Typical enterprise NTP servers
  Most organizations run Stratum 2 internally

Stratum 3 — Tertiary:
  Synchronized from Stratum 2
  Typical network devices (routers, switches)
  
Stratum 15 — Maximum usable stratum
Stratum 16 — Unsynchronized (device does not trust its own clock)

The further from Stratum 0, the greater the potential time drift.
For most enterprise use, Stratum 3 or 4 is more than accurate enough.
```

### NTP Configuration on Cisco IOS

```cisco
! ════ CONFIGURE ROUTER AS NTP CLIENT (sync from external server) ════

! Primary NTP server (Stratum 1 — Google's public NTP)
Router(config)# ntp server 216.239.35.0 prefer    ! "prefer" = use this first

! Backup NTP server
Router(config)# ntp server 216.239.35.4

! Additional public NTP servers (redundancy)
Router(config)# ntp server 0.pool.ntp.org
Router(config)# ntp server 1.pool.ntp.org

! Set timezone (for correct local time display in logs)
Router(config)# clock timezone IST 5 30            ! IST = UTC+5:30 (India)
Router(config)# clock timezone EST -5              ! EST = UTC-5 (US Eastern)

! Enable summer time (daylight saving) if applicable
Router(config)# clock summer-time EDT recurring

! ════ CONFIGURE ROUTER AS NTP SERVER (for internal devices) ════

! This router becomes an NTP server at Stratum 3
Router(config)# ntp master 3
! Stratum number = what this router will advertise to its clients
! Internal switches and devices can then sync from this router

! ════ NTP AUTHENTICATION (prevent spoofing attacks) ════
Router(config)# ntp authenticate                          ! enable NTP auth
Router(config)# ntp authentication-key 1 md5 NTPKey@2024 ! define key
Router(config)# ntp trusted-key 1                        ! trust this key

! On client syncing from this server:
Client(config)# ntp server 192.168.99.1 key 1
Client(config)# ntp trusted-key 1
Client(config)# ntp authenticate

! ════ SOURCE INTERFACE (important for multi-homed routers) ════
! Ensure NTP packets always come from the same IP (for ACL/firewall rules)
Router(config)# ntp source Loopback0

! ════ VERIFICATION ════
Router# show ntp status
Clock is synchronized, stratum 3, reference is 216.239.35.0
nominal freq is 250.0000 Hz, actual freq is 250.0001 Hz, precision is 2**10
ntp uptime is 120000 (1/100 of seconds), resolution is 4000
reference time is E8A3C520.8F5C28F6 (10:15:28.560 IST Mon Apr 26 2026)
clock offset is 0.2345 msec, root delay is 14.52 msec
root dispersion is 7.81 msec, peer dispersion is 0.43 msec
loopback(0): stratum 1, offset -0.23 msec

Router# show ntp associations
  address         ref clock       st   when   poll reach   delay   offset    disp
*~216.239.35.0    .GPS.            1     42     64   377   14.523   0.234   0.125
+~216.239.35.4    .GPS.            1     38     64   377   15.821   0.189   0.098
* = current best source, + = candidate, ~ = configured

Router# show clock detail
10:15:30.285 IST Mon Apr 26 2026
Time source is NTP              ← confirms NTP is syncing the clock
```

---

## 6. SNMP — Simple Network Management Protocol

### What SNMP Does

SNMP is the standard protocol for **monitoring and managing network devices**. It allows a centralized Network Management Station (NMS) to collect performance statistics, receive alerts about problems, and even modify device configuration — all without having to SSH into individual devices.

Think of SNMP as a reporting system built into every network device. The device continuously collects data about itself (CPU usage, interface bandwidth, error counts, memory utilization) and stores it in a database called the MIB (Management Information Base). An NMS can query this database at any time to retrieve current or historical data, generate graphs and reports, and set thresholds that trigger alerts.

### SNMP Architecture

```
┌──────────────────────────────────────────────────────────────┐
│           Network Management Station (NMS)                   │
│                                                              │
│  SolarWinds / Cisco Prime / PRTG / Zabbix / Nagios          │
│  • Polls devices for statistics                              │
│  • Displays graphs and dashboards                            │
│  • Receives and processes traps                              │
│  • Generates alerts when thresholds exceeded                 │
└───────────────┬─────────────────────┬────────────────────────┘
                │                     │
         SNMP Get/Set            SNMP Traps
         (NMS initiates)        (Device initiates)
                │                     │
┌───────────────▼─────────────────────▼────────────────────────┐
│              Managed Device (Router/Switch/Server)            │
│                                                              │
│  SNMP Agent: responds to queries, sends traps                │
│  MIB (Management Information Base): data about the device   │
│  OID (Object Identifier): address of each MIB variable      │
└──────────────────────────────────────────────────────────────┘
```

### SNMP Operations

Each SNMP operation serves a specific purpose in the monitoring and management workflow.

**Get** is the most common operation — the NMS requests the current value of a specific OID (e.g., "what is the current CPU utilization?"). **GetNext** retrieves the next OID in the MIB tree, allowing the NMS to "walk" through the MIB sequentially. **GetBulk** (SNMPv2+) retrieves multiple OIDs in a single request — far more efficient than individual Gets when collecting many statistics. **Set** allows the NMS to modify a device's configuration by writing a new value to a writable OID — this is powerful but also a security risk if not properly controlled.

**Trap** is fundamentally different from the other operations because the **device initiates it** without being asked. When a significant event occurs (interface goes down, CPU exceeds 90%, a fan fails), the device proactively sends a Trap to the NMS. The problem with Traps is they are "fire-and-forget" — if the Trap packet is lost in transit, the NMS never knows about it. **Inform** (SNMPv2+) solves this by requiring the NMS to send an acknowledgement — if the device doesn't receive the ack, it retransmits.

### SNMP Versions — A Security Evolution

The three SNMP versions represent an evolution in security approach, from essentially no security to robust encryption and authentication.

**SNMPv1** uses a "community string" as its only security mechanism. This string is sent in plain text in every packet — it is literally a shared password written in the clear for anyone capturing network traffic to read. SNMPv1 is completely insecure by modern standards. Avoid it entirely except in isolated lab environments.

**SNMPv2c** keeps the same insecure community string approach but adds performance improvements including the GetBulk operation and 64-bit counters (important for high-speed interfaces where 32-bit counters wrap too quickly). Still insecure because the community string is still plaintext.

**SNMPv3** completely rearchitects security with three configurable levels. **noAuthNoPriv** still has no security (similar to v1/v2c). **authNoPriv** adds MD5 or SHA authentication — you know the packet came from a legitimate source. **authPriv** adds both authentication AND encryption (DES or AES) — the payload is encrypted so even someone capturing the traffic cannot read the SNMP data. Always use SNMPv3 with authPriv in production environments.

### SNMP Configuration

```cisco
! ════ SNMPv2c (common but insecure — avoid in production) ════

! Read-only community string (NMS can read but not change device config)
Router(config)# snmp-server community PUBLIC_RO ro
! NMS uses "PUBLIC_RO" as password to read data

! Read-write community string (NMS can read AND change device config)
Router(config)# snmp-server community PRIVATE_RW rw
! NEVER use "public" and "private" — these are the most-scanned defaults!

! Restrict which NMS IP can use SNMP (important security measure even with v2c)
Router(config)# access-list 99 permit 10.0.0.100    ! NMS IP
Router(config)# snmp-server community PUBLIC_RO ro 99   ! apply ACL to community

! Configure trap destination (where to send SNMP alerts)
Router(config)# snmp-server host 10.0.0.100 version 2c PUBLIC_RO
Router(config)# snmp-server enable traps                ! enable all traps
Router(config)# snmp-server enable traps snmp linkdown linkup  ! specific traps
Router(config)# snmp-server enable traps config          ! config change traps

! ════ SNMPv3 (secure — recommended for production) ════

! Create an SNMP group with authentication and privacy (authPriv)
Router(config)# snmp-server group MGMT_GROUP v3 priv
!                                               ↑ version 3
!                                                   ↑ priv = authPriv level

! Create an SNMP user in that group
Router(config)# snmp-server user NMS_ADMIN MGMT_GROUP v3
  auth sha AuthPassword@2024          ! SHA authentication
  priv aes 128 PrivPassword@2024      ! AES-128 encryption

! Configure trap destination for SNMPv3
Router(config)# snmp-server host 10.0.0.100 version 3 priv NMS_ADMIN

! ════ SNMP SYSTEM INFORMATION ════
Router(config)# snmp-server location "Server Room 3 - Rack 2A"
Router(config)# snmp-server contact "netops@company.com"

! ════ VERIFICATION ════
Router# show snmp
Router# show snmp community           ! view configured communities
Router# show snmp user                ! view SNMPv3 users
Router# show snmp group               ! view SNMPv3 groups
Router# show snmp host                ! view trap destinations
```

---

## 7. Syslog — System Logging

### Why Centralized Logging Matters

Every Cisco router and switch generates log messages constantly — interface status changes, authentication attempts, configuration changes, routing adjacency changes, security violations. By default, these messages appear on the console and disappear forever when the session ends. Even if stored in the device's RAM buffer (`logging buffered`), they are lost on reboot and limited in size.

In a real network, you need all these messages stored in a central, permanent, searchable location. This is what a syslog server provides. When you have 200 devices all sending their logs to one syslog server, you can search across all logs simultaneously, correlate events from different devices, and retain logs for compliance purposes (many regulations require 90 days or more of log retention).

### Syslog Severity Levels

Syslog defines eight severity levels numbered 0 through 7. This numbering is counterintuitive — level 0 is the MOST severe, not the least. Think of it as a priority scale where 0 is the highest priority emergency.

```
Level  Keyword        Meaning                     Example Scenario
─────  ─────────────  ─────────────────────────   ──────────────────────────────────
0      Emergency      System is unusable          Kernel panic, hardware failure
1      Alert          Immediate action needed     Temperature critical, disk dying
2      Critical       Critical conditions         Hardware failure, software crash
3      Error          Error conditions            Interface error, auth failure
4      Warning        Warning conditions          Memory low, config error
5      Notice         Normal but significant      Interface up/down, config change ← most common
6      Informational  General information         Login success, route added
7      Debug          Debug-level detail          Packet traces, protocol exchanges

Memory aid: "Every Awesome Cisco Engineer Will Need Daily Insights"
  Emergency(0) Alert(1) Critical(2) Error(3) Warning(4) Notice(5) Debug(6—wait, 7)
  
Better: 0=Emergency, 1=Alert, 2=Critical, 3=Error, 4=Warning, 5=Notice, 6=Info, 7=Debug
```

When you configure `logging trap [level]`, the router sends that level AND all levels MORE severe (lower numbers). So `logging trap warnings` sends levels 0, 1, 2, 3, and 4 — not just level 4.

### Syslog Configuration

```cisco
! ════ CONFIGURE SYSLOG SERVER ════
Router(config)# logging host 10.0.0.200               ! syslog server IP
Router(config)# logging host 10.0.0.201               ! backup syslog server

! Set minimum severity to send to syslog server
! "informational" (level 6) sends 0-6, missing only debug (7)
Router(config)# logging trap informational
! Most common choice — captures everything except debug messages

! ════ SOURCE INTERFACE (important!) ════
! Syslog messages should always come from a stable IP
Router(config)# logging source-interface Loopback0
! Without this, messages might come from different IPs on a multi-homed router
! Making it hard for the syslog server to identify the device

! ════ TIMESTAMPS (essential for correlation) ════
Router(config)# service timestamps log datetime msec localtime show-timezone
! Adds: Apr 26 2026 10:15:30.285 IST to every log message
! "msec" = millisecond precision (important for correlating rapid events)
! "localtime" = shows local time zone (not UTC)
! "show-timezone" = includes timezone abbreviation in message

! ════ CONSOLE LOGGING ════
Router(config)# logging console warnings    ! show level 4 and above on console
! Set lower to reduce console output (debug fills the screen with output)
! Default is debugging (level 7) — too verbose for production

! ════ BUFFER LOGGING (store in RAM) ════
Router(config)# logging buffered 65536 informational  ! 64KB buffer, level 0-6
! Viewable with: show logging
! Lost on reboot — use for recent events only

! ════ MONITOR LOGGING (for SSH/Telnet sessions) ════
Router(config)# logging monitor notifications  ! level 5 and above
! Then in your SSH session:
Router# terminal monitor    ! enable log display in your session
Router# terminal no monitor ! disable when done

! ════ VERIFICATION ════
Router# show logging
Syslog logging: enabled (0 messages dropped, 0 flushes, 0 overruns)
    Console logging: level warnings, 125 messages logged
    Monitor logging: level notifications, 0 messages logged
    Buffer logging:  level informational, 4523 messages logged
    Logging to 10.0.0.200, 4523 message lines logged

*Apr 26 2026 10:15:28.420 IST: %LINK-3-UPDOWN: Interface GigabitEthernet0/1, changed state to up
*Apr 26 2026 10:15:29.110 IST: %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/1, changed state to up
```

---

## 8. FTP and TFTP

### What These Protocols Are For

In the context of network devices, FTP and TFTP are primarily used for two purposes: transferring IOS firmware images to and from devices (for upgrades or backups), and transferring configuration files (saving a config backup to a server, or loading a config from a server onto a new device).

Understanding the differences between these two protocols helps you choose the right one for each task.

### FTP — Feature-Rich but Complex

FTP (File Transfer Protocol) is a full-featured file transfer protocol that provides authentication (username and password), directory navigation, file management commands (create, delete, rename directories and files), and both active and passive connection modes. The credentials and data are sent in plain text (which is a security concern), but FTP is still useful when you need its richer functionality.

FTP uses two separate TCP connections — a control connection on port 21 that carries commands and responses, and a data connection on port 20 (active mode) or a negotiated port (passive mode) that carries the actual file data. This two-channel design can sometimes cause problems with firewalls and NAT.

```cisco
! Copy IOS image from FTP server to router flash
Router# copy ftp flash
Address or name of remote host? 10.0.0.100
Username? ftpuser
Password? FTPpass123
Source filename? c2900-universalk9-mz.SPA.157-3.M8.bin
Destination filename [c2900-universalk9-mz.SPA.157-3.M8.bin]? [Enter]
Accessing ftp://ftpuser:FTPpass123@10.0.0.100/c2900...
Loading c2900-universalk9-mz.SPA.157-3.M8.bin
!!!!!!!!!!!!!!!!!!!!  (each ! = 1KB transferred)
[OK - 73214008 bytes]

! Copy running config to FTP server (backup)
Router# copy running-config ftp
Address or name of remote host? 10.0.0.100
Username? ftpuser
Password? FTPpass123
Destination filename? router1-config-backup-20260426.txt

! Configure FTP credentials in config (so you don't have to type them each time)
Router(config)# ip ftp username ftpuser
Router(config)# ip ftp password FTPpass123
```

### TFTP — Simple but Limited

TFTP (Trivial File Transfer Protocol) is the stripped-down, simple alternative. It has no authentication, no directory navigation, and no file management — it can only GET a file or PUT a file. It runs over UDP port 69, which makes it lightweight and fast. Because there is no authentication, you must ensure TFTP is only accessible from trusted networks.

Despite its simplicity (or because of it), TFTP is widely used for network device operations. It is simpler to set up a TFTP server than an FTP server, and the absence of authentication is not a concern when the server is on a secure management network.

```cisco
! Copy IOS image from TFTP server to flash
Router# copy tftp flash
Address or name of remote host? 10.0.0.100
Source filename? c2900-universalk9-mz.SPA.157-3.M8.bin
Destination filename [c2900-universalk9-mz.SPA.157-3.M8.bin]? [Enter]
Accessing tftp://10.0.0.100/c2900-universalk9-mz.SPA.157-3.M8.bin...
Loading c2900-universalk9-mz.SPA.157-3.M8.bin from 10.0.0.100:
!!!!!!!!!!!!!!!!!
[OK - 73214008 bytes]

! Copy startup config to TFTP (save config backup)
Router# copy startup-config tftp
Address or name of remote host? 10.0.0.100
Destination filename? router1-backup-20260426.cfg

! Copy TFTP config to running config (restore or apply config)
Router# copy tftp running-config
Address or name of remote host? 10.0.0.100
Source filename? router1-backup-20260426.cfg

! Useful IOS file commands
Router# show flash:                  ! list files in flash
Router# show flash: detail           ! detailed listing with sizes
Router# delete flash:old-ios.bin     ! delete old IOS image
Router# verify flash:new-ios.bin     ! verify file integrity (MD5 hash)
```

### FTP vs TFTP Comparison

```
Feature            FTP                         TFTP
────────────────   ─────────────────────────   ──────────────────────────
Transport          TCP                         UDP
Ports              21 (control) + 20 (data)    69
Authentication     Yes (username + password)   No
Encryption         No (plain text)             No
Directory browse   Yes (ls, cd, mkdir, etc.)   No (single file only)
File management    Yes (rename, delete, etc.)  No
Reliability        TCP provides reliability    App-level ACK/retransmit
Use case           Full-featured file ops      Simple IOS/config transfer
Server complexity  More complex to set up      Very simple
Security           Credentials in plain text   No auth at all
```

---

## 9. QoS — Quality of Service

### Why QoS Is Necessary

In a world with unlimited bandwidth, every packet would be delivered instantly with no delay and no loss — QoS would be unnecessary. But networks have finite bandwidth, and when more traffic arrives at a router or switch than it can forward in that instant, something must wait in a queue. The question QoS answers is: **when traffic must wait, which traffic should be served first?**

Without QoS, all traffic is treated equally — a large file backup competes for bandwidth with a VoIP phone call on equal terms. But a file backup can tolerate delays of seconds or even minutes without any noticeable impact. A VoIP call cannot tolerate even 150 milliseconds of one-way delay before voice quality degrades significantly. Equal treatment of unequal needs leads to poor outcomes.

QoS allows you to define policies that recognize traffic types and give preferential treatment to traffic that needs it — ensuring that voice calls are always served promptly even when the link is busy with background traffic.

### The Three Requirements for Real-Time Traffic

Voice and video applications have specific requirements that differ fundamentally from data applications. Understanding these requirements helps you understand why QoS is designed the way it is.

**Bandwidth** is the amount of data per second an application needs. A G.711 VoIP call requires approximately 87 Kbps including all headers. This must be available reliably — if the link is 100% utilized with file transfers, the VoIP call has no bandwidth left and the call drops.

**Latency (delay)** is the one-way time for a packet to travel from sender to receiver. The ITU-T G.114 standard recommends one-way delay of no more than 150 milliseconds for voice. Beyond 300ms, the conversation feels like a satellite call — awkward and difficult. Delay has two sources: propagation delay (the speed of light limit over physical distance — unchangeable) and queuing delay (packets waiting in a queue — QoS can address this).

**Jitter** is the variation in delay between consecutive packets. Even if average delay is acceptable, if some packets arrive after 10ms and others after 100ms, the receiving phone cannot reconstruct smooth audio. Jitter buffers help compensate, but they add latency. Consistent delay (low jitter) is just as important as low average delay.

```
VoIP Requirements (ITU-T G.114 + Cisco recommendations):
  One-way delay:  < 150ms   (absolute max 300ms)
  Jitter:         < 30ms
  Packet loss:    < 1%
  Bandwidth:      ~87 Kbps per G.711 call (codec dependent)

Video Conference Requirements:
  One-way delay:  < 150ms
  Jitter:         < 30ms
  Packet loss:    < 0.1%
  Bandwidth:      384 Kbps to 4+ Mbps (resolution dependent)
```

### The QoS Model — Classify, Mark, Act

QoS works in three stages. First you must **identify** what each packet is (classification). Then you **label** it with a marking that other devices can act on (marking). Then you apply **actions** — queuing, scheduling, policing, shaping — based on those markings.

**Classification** is the process of examining packets and assigning them to a class. You can classify based on many criteria: source/destination IP address, TCP/UDP port numbers, DSCP markings already in the packet, the physical interface the packet arrived on, or deep packet inspection (NBAR — Network-Based Application Recognition) that can identify applications by their behavior patterns.

**Marking** is writing a value into the packet header that downstream devices can read to identify the class without having to reclassify. The most important marking fields are DSCP (Differentiated Services Code Point) in the IP header and CoS (Class of Service) in the 802.1Q VLAN tag.

```
DSCP Field (6 bits in IP header, values 0-63):
  Value 46  (binary 101110) = EF (Expedited Forwarding) — VoIP payload
  Value 34  (binary 100010) = AF41 — Video conferencing
  Value 26  (binary 011010) = AF31 — Call signaling (SIP, H.323)
  Value 18  (binary 010010) = AF21 — Transactional data
  Value 10  (binary 001010) = AF11 — Bulk data
  Value 8   (binary 001000) = CS1  — Scavenger (low priority)
  Value 0   (binary 000000) = BE   — Best Effort (default, unmarked traffic)

CoS (Class of Service):
  3 bits in the 802.1Q Ethernet frame header
  Values 0-7 (7 = highest priority)
  CoS 5 = Voice payload (matches DSCP EF)
  CoS 3 = Video
  CoS 0 = Best effort (default)

Trust Boundary:
  IP phones mark their own traffic with DSCP EF (voice) and AF31 (signaling)
  Switches should be configured to trust these markings from phones
  But NOT from PCs — PCs could fake high-priority markings
  The switch port connecting an IP phone = the trust boundary
```

### Queuing and Scheduling

When a router's output queue is full (congestion), the scheduler decides which packet to send next. Different queuing mechanisms make different choices.

**FIFO (First In, First Out)** is the simplest — packets are sent in the order they arrived. There is no prioritization whatsoever. This is the default on high-speed interfaces where congestion is rare. When congestion occurs with FIFO, all traffic suffers equally.

**LLQ (Low Latency Queuing)** is the recommended queuing method for networks carrying voice and video traffic. It combines a **strict priority queue** (for voice) with **class-based weighted fair queuing** (CBWFQ) for other traffic. The strict priority queue is served first, always. Voice packets in this queue are sent before any other packet, regardless of how many data packets are waiting. This guarantees low latency and low jitter for voice. CBWFQ then provides bandwidth guarantees to other classes from the remaining capacity.

```
LLQ Conceptual Operation:
  Output buffer with multiple queues:
  
  Priority Queue (VoIP):  [VoIP][VoIP][VoIP]     ← always served FIRST
  Queue 1 (Video):        [Vid][Vid]               ← 30% of remaining BW
  Queue 2 (Data):         [Data][Data][Data][Data] ← 20% of remaining BW
  Queue 3 (Bulk):         [Bulk][Bulk][Bulk]       ← 10% of remaining BW
  Default Queue:          [Best Effort]            ← whatever remains
  
  Scheduler always drains Priority Queue first.
  Then distributes remaining capacity per CBWFQ weights.
  Result: VoIP gets < 10ms queuing delay regardless of congestion.
```

### Policing vs Shaping

Both policing and shaping are used to enforce a rate limit on traffic. They differ in what they do when traffic exceeds the limit.

**Policing** measures traffic against a defined rate and **drops or re-marks** packets that exceed the limit. It acts instantly and has no memory of past traffic — if a burst arrives, policing drops packets immediately when the rate is exceeded. Policing is used by service providers to enforce contracted rates and is typically applied at network ingress points.

**Shaping** measures traffic and **buffers** packets that exceed the defined rate, releasing them smoothly at the allowed rate. It "smooths out" traffic bursts by absorbing them into a buffer and releasing at a controlled rate. Shaping is used by customers to match traffic to their contracted SP rate, ensuring they don't send faster than the SP will accept.

### QoS Configuration with MQC

Cisco's **Modular QoS CLI (MQC)** provides a three-step framework for QoS configuration that cleanly separates classification (what), policy (how), and application (where).

```cisco
! ════ STEP 1: CLASS MAPS (define what traffic belongs to each class) ════

! VoIP class — match on DSCP EF marking
Router(config)# class-map match-any VOICE_CLASS
Router(config-cmap)# match dscp ef                ! match DSCP 46 (VoIP)
Router(config-cmap)# description "Voice payload traffic"

! Video class
Router(config)# class-map match-any VIDEO_CLASS
Router(config-cmap)# match dscp af41              ! DSCP 34 (video conference)
Router(config-cmap)# match dscp af42              ! DSCP 36 (also video)

! Call signaling class
Router(config)# class-map match-any SIGNALING_CLASS
Router(config-cmap)# match dscp af31              ! DSCP 26 (call setup)
Router(config-cmap)# match dscp cs3               ! DSCP 24 (also signaling)

! Critical data class — match by ACL (source/destination based)
Router(config)# access-list 101 permit tcp 192.168.30.0 0.0.0.255 host 10.1.1.5 eq 1433
Router(config)# class-map match-all CRITICAL_DATA
Router(config-cmap)# match access-group 101       ! SQL Server traffic

! ════ STEP 2: POLICY MAP (define what to DO with each class) ════

Router(config)# policy-map WAN_QOS_POLICY

! VoIP gets strict priority — always served first
Router(config-pmap)# class VOICE_CLASS
Router(config-pmap-c)# priority 500              ! 500 Kbps strict priority
! "priority" = LLQ strict priority queue
! Guarantees 500 Kbps with < 1ms queuing delay for voice

! Video gets guaranteed bandwidth
Router(config-pmap)# class VIDEO_CLASS
Router(config-pmap-c)# bandwidth 2000            ! guarantee 2 Mbps for video
! "bandwidth" = CBWFQ — guaranteed minimum, can borrow if others idle

! Signaling gets small guaranteed bandwidth
Router(config-pmap)# class SIGNALING_CLASS
Router(config-pmap-c)# bandwidth 128             ! 128 Kbps for call setup

! Critical data gets guaranteed bandwidth
Router(config-pmap)# class CRITICAL_DATA
Router(config-pmap-c)# bandwidth 1000            ! 1 Mbps for database traffic

! Everything else (class-default catches all unmatched traffic)
Router(config-pmap)# class class-default
Router(config-pmap-c)# fair-queue                ! WFQ among best-effort flows

! ════ STEP 3: SERVICE POLICY (apply to interface) ════
Router(config)# interface GigabitEthernet0/1     ! WAN interface (toward ISP)
Router(config-if)# service-policy output WAN_QOS_POLICY
! Apply outbound only — QoS scheduling affects outgoing traffic
! (Inbound can use policing to enforce received rates)

! ════ VERIFICATION ════
Router# show policy-map interface GigabitEthernet0/1
Router# show class-map VOICE_CLASS
Router# show policy-map WAN_QOS_POLICY
```

---

## 10. Key Takeaways & Exam Tips

### Key Takeaways

NAT solves the IPv4 address exhaustion problem by allowing private RFC 1918 addresses to be used internally while sharing a small pool of public addresses for internet access. The three NAT types serve different needs: static NAT provides permanent one-to-one mapping for servers that must be reachable from the internet; dynamic NAT assigns pool addresses on demand but fails when the pool is exhausted; and PAT (NAT Overload) uses port numbers to allow thousands of devices to share a single public IP simultaneously. The `ip nat inside` and `ip nat outside` interface commands are mandatory — without them, no translation occurs.

DHCP automates IP configuration using the DORA process (Discover, Offer, Request, Acknowledge). The `ip helper-address` command on a router interface converts DHCP broadcasts into unicast packets directed at a remote DHCP server, enabling centralized DHCP for multiple subnets. NTP synchronizes device clocks to ensure accurate log timestamps and security protocol operation — Stratum 0 is the reference clock, and stratum numbers increase with distance from the source.

SNMP provides centralized network monitoring through an NMS that polls device MIBs. Only SNMPv3 with authPriv provides real security. Syslog centralizes log messages using severity levels 0 (Emergency) through 7 (Debug), where lower numbers are more severe — `logging trap informational` sends levels 0–6.

QoS provides preferential treatment to latency-sensitive traffic. The three stages are classify (identify traffic), mark (label with DSCP/CoS), and act (queue, schedule, police, shape). LLQ is the recommended queuing method — it uses a strict priority queue for voice (ensuring < 10ms queuing delay) combined with CBWFQ for other traffic. VoIP uses DSCP EF (value 46) and requires latency < 150ms, jitter < 30ms, and loss < 1%.

### Exam Tips

> 🎯 **Most Tested Topics:**

1. **"What NAT type uses port numbers to share one public IP among many devices?"** → **PAT (NAT Overload)** — configured with the `overload` keyword
2. **"What is the Inside Local address in NAT?"** → The **private IP** of the internal device before translation
3. **"What command forwards DHCP broadcasts to a remote server?"** → **`ip helper-address [server-IP]`** on the interface facing clients
4. **"What are the four steps of DHCP?"** → **DORA** — Discover, Offer, Request, Acknowledge
5. **"What is NTP Stratum 0?"** → The **reference clock** (atomic clock, GPS) — not actually on the network; Stratum 1 servers connect to it
6. **"Which SNMP version provides full encryption?"** → **SNMPv3** with authPriv (authentication + AES encryption)
7. **"What SNMP operation does the device initiate without being asked?"** → **Trap** (Inform requires acknowledgement)
8. **"What is the most severe syslog level?"** → **Level 0 — Emergency** (not level 7, which is Debug — the least severe)
9. **"What does `logging trap warnings` send?"** → Levels **0 through 4** (Emergency, Alert, Critical, Error, Warning) — it sends all messages at that level and MORE severe
10. **"What protocol does TFTP use and on what port?"** → **UDP port 69**
11. **"What is the DSCP value for VoIP traffic?"** → **EF = 46** (Expedited Forwarding)
12. **"What are the three VoIP QoS requirements?"** → **Latency < 150ms, Jitter < 30ms, Loss < 1%**
13. **"What QoS queuing method guarantees lowest latency for voice?"** → **LLQ (Low Latency Queuing)** — uses strict priority queue for voice
14. **"What is the difference between policing and shaping?"** → Policing **drops** excess traffic immediately; shaping **buffers** and delays excess traffic to smooth it out
15. **"What command applies a QoS policy to an interface?"** → **`service-policy output [policy-name]`**

---

✅ **Module 10 — All 10 Sections Complete.**

*← Previous: [09 — WAN & SD-WAN](./09_WAN_SDWAN.md)*  
*Next: [11 — Network Automation & Programmability →](./11_Automation.md)*
