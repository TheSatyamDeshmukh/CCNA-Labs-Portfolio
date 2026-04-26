# 07 — Routing Fundamentals: Complete Reference

> **CCNA Exam Domain:** 3.0 IP Connectivity  
> **Exam Weight:** ~25% of total exam  
> **Difficulty:** ⭐⭐⭐⭐☆

---

## Table of Contents

1. [How Routers Work](#1-how-routers-work)
2. [Routing Table — Deep Dive](#2-routing-table--deep-dive)
3. [Administrative Distance (AD)](#3-administrative-distance-ad)
4. [Static Routing](#4-static-routing)
5. [Floating Static Routes](#5-floating-static-routes)
6. [Default Routes](#6-default-routes)
7. [Dynamic Routing Overview](#7-dynamic-routing-overview)
8. [OSPF — Open Shortest Path First](#8-ospf--open-shortest-path-first)
9. [EIGRP — Enhanced Interior Gateway Routing Protocol](#9-eigrp--enhanced-interior-gateway-routing-protocol)
10. [RIP — Routing Information Protocol](#10-rip--routing-information-protocol)
11. [Route Redistribution Basics](#11-route-redistribution-basics)
12. [Routing Troubleshooting](#12-routing-troubleshooting)
13. [Key Takeaways & Exam Tips](#13-key-takeaways--exam-tips)

---

## 1. How Routers Work

### Router's Core Job

A router receives packets on one interface and forwards them out another interface toward the destination. To do this, it needs three things:

```
1. ROUTING TABLE — knows which networks exist and how to reach them
2. FORWARDING DECISION — matches destination IP to best route
3. PACKET REWRITING — updates MAC addresses for each hop
```

### Packet Forwarding — Step by Step

```
PC-A (192.168.1.10) sends packet to Server (10.0.0.5)

STEP 1 — PC-A checks: is 10.0.0.5 in my subnet?
  My IP:  192.168.1.10/24 → my subnet = 192.168.1.0/24
  10.0.0.5 is NOT in 192.168.1.0/24 → send to default gateway

STEP 2 — PC-A sends Ethernet frame:
  Src MAC: PC-A's MAC
  Dst MAC: Gateway Router's MAC (learned via ARP)
  Src IP:  192.168.1.10
  Dst IP:  10.0.0.5

STEP 3 — Router receives frame on Gi0/0:
  Strip L2 header (Ethernet frame)
  Read L3 header: Dst IP = 10.0.0.5
  Look up 10.0.0.5 in routing table
  Find: 10.0.0.0/8 via 172.16.1.2 out Gi0/1
  Decrement TTL by 1 (e.g., 64 → 63)
  ARP for 172.16.1.2 MAC (next-hop)
  Build NEW Ethernet frame:
    Src MAC: Router's Gi0/1 MAC
    Dst MAC: Next-hop router's MAC
    Src IP:  192.168.1.10 (IP unchanged!)
    Dst IP:  10.0.0.5 (IP unchanged!)
  Send out Gi0/1

STEP 4 — Next router repeats process until packet reaches server

KEY INSIGHT: IP addresses NEVER change hop-to-hop
             MAC addresses change at EVERY hop
```

### TTL — Time To Live

```
TTL prevents packets from looping forever in the network.

  - Each router decrements TTL by 1
  - If TTL reaches 0 → router drops the packet
  - Router sends ICMP "Time Exceeded" (Type 11) back to source

Default TTL values by OS:
  Windows:  128
  Linux:    64
  Cisco IOS: 255

Traceroute uses this! Sends packets with TTL=1, 2, 3...
  TTL=1 → first router drops it, sends ICMP "TTL Exceeded" → reveals router 1 IP
  TTL=2 → second router drops, reveals router 2 IP
  TTL=N → destination reached → ICMP Echo Reply received
```

### Router vs Layer 3 Switch

```
┌────────────────────────┬──────────────────────┬──────────────────────┐
│ Feature                │ Router               │ Layer 3 Switch       │
├────────────────────────┼──────────────────────┼──────────────────────┤
│ Primary purpose        │ WAN/inter-network    │ LAN inter-VLAN       │
│ Forwarding mechanism   │ Software (process)   │ Hardware ASIC        │
│ Forwarding speed       │ Slower               │ Wire speed           │
│ Interface types        │ Many (serial, DSL..) │ Ethernet only        │
│ WAN support            │ Yes                  │ Limited              │
│ Routing protocols      │ All                  │ All                  │
│ Cost                   │ Higher per port      │ Lower per port       │
│ Best for               │ Edge, WAN, ISP       │ Campus/LAN core      │
└────────────────────────┴──────────────────────┴──────────────────────┘
```

---

## 2. Routing Table — Deep Dive

### Reading the Routing Table

```cisco
Router# show ip route

Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP

Gateway of last resort is 203.0.113.1 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 203.0.113.1
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.0.0.0/30 is directly connected, Serial0/0/0
L        10.0.0.1/32 is directly connected, Serial0/0/0
      192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.1.0/24 is directly connected, GigabitEthernet0/0
L        192.168.1.1/32 is directly connected, GigabitEthernet0/0
O     192.168.2.0/24 [110/2] via 10.0.0.2, 00:15:32, Serial0/0/0
D     172.16.0.0/16 [90/2172416] via 10.0.0.2, 00:15:32, Serial0/0/0
S     10.10.10.0/24 [1/0] via 10.0.0.2
R     10.20.20.0/24 [120/3] via 10.0.0.2, 00:00:25, Serial0/0/0
```

### Routing Table Entry Anatomy

```
O     192.168.2.0/24 [110/2] via 10.0.0.2, 00:15:32, Serial0/0/0
│     │              │   │       │          │          │
│     │              │   │       │          │          └─ Exit interface
│     │              │   │       │          └─ Time since last update
│     │              │   │       └─ Next-hop IP address
│     │              │   └─ Metric (OSPF cost = 2)
│     │              └─ Administrative Distance = 110 (OSPF)
│     └─ Destination network / prefix length
└─ Route source code (O = OSPF)

Format: [AD/metric]
  AD     = how trustworthy the source is (lower = better)
  metric = how far/good the path is (lower = better)
```

### Route Types Explained

```
C — Connected Route:
  Network directly attached to router interface
  Created automatically when interface has IP + is up/up
  AD = 0 (most trusted — directly connected)
  Example: C  192.168.1.0/24 is directly connected, Gi0/0

L — Local Route:
  The router's OWN interface IP address as a /32 host route
  Created automatically alongside connected route
  AD = 0
  Example: L  192.168.1.1/32 is directly connected, Gi0/0
  Why useful: Router knows exactly which IP belongs to itself

S — Static Route:
  Manually configured by administrator
  AD = 1 (very trusted, just below connected)
  Example: S  10.10.10.0/24 [1/0] via 10.0.0.2

S* — Default Static Route (candidate default):
  The "last resort" route — matches everything not in table
  Example: S* 0.0.0.0/0 [1/0] via 203.0.113.1

O — OSPF Route:
  Learned via OSPF routing protocol
  AD = 110
  Example: O  192.168.2.0/24 [110/2] via 10.0.0.2

D — EIGRP Route:
  Learned via EIGRP routing protocol
  AD = 90
  Example: D  172.16.0.0/16 [90/2172416] via 10.0.0.2

R — RIP Route:
  Learned via RIP routing protocol
  AD = 120
  Example: R  10.20.20.0/24 [120/3] via 10.0.0.2
```

### Longest Prefix Match — The Most Important Rule

When multiple routes match a destination IP, the router **always picks the most specific match** (longest prefix / highest subnet mask bits).

```
Routing Table:
  10.0.0.0/8       via Router-A    (matches any 10.x.x.x)
  10.1.0.0/16      via Router-B    (matches any 10.1.x.x)
  10.1.1.0/24      via Router-C    (matches any 10.1.1.x)
  10.1.1.128/25    via Router-D    (matches 10.1.1.128-255)
  0.0.0.0/0        via Router-E    (matches everything)

Destination packet: 10.1.1.200

Match analysis:
  0.0.0.0/0     ✓ matches (0 bits match)
  10.0.0.0/8    ✓ matches (8 bits match)
  10.1.0.0/16   ✓ matches (16 bits match)
  10.1.1.0/24   ✓ matches (24 bits match)
  10.1.1.128/25 ✓ matches (25 bits match) ← LONGEST = WINNER

Result: Forward to Router-D via 10.1.1.128/25 route

WHY THIS MATTERS:
  More specific route = more precise destination knowledge
  /32 host route always beats /24 network route
  /24 beats /16, /16 beats /8, /8 beats /0
```

### Routing Table Lookup Process

```
Packet arrives, Dst IP = 10.1.1.200

Step 1: Does exact /32 host route exist for 10.1.1.200?
        → No
Step 2: Does /25 match? 10.1.1.128/25?
        → YES → use it, STOP

If not found in any subnet:
Step N: Does 0.0.0.0/0 (default route) exist?
        → YES → use default route
        → NO  → drop packet, send ICMP Unreachable to source
```

---

## 3. Administrative Distance (AD)

### What Is AD?

AD measures how **trustworthy** a routing information source is. When a router learns the SAME network from multiple sources (e.g., both OSPF and EIGRP), AD decides which one goes in the routing table.

```
Router learns 192.168.5.0/24 from:
  OSPF  → AD = 110
  EIGRP → AD = 90   ← WINS (lower AD = more trusted)
  RIP   → AD = 120

Result: EIGRP route installed in routing table
        OSPF and RIP routes kept in their protocol databases
        (ready to take over if EIGRP route disappears)
```

### Complete AD Reference Table

```
Route Source                    AD Value    Notes
──────────────────────────────  ─────────  ──────────────────────────────
Directly Connected              0           Most trusted — physically attached
Static Route                    1           Administrator-defined
EIGRP Summary Route             5           Auto-summarized EIGRP
External BGP (eBGP)             20          Routes learned from other AS
EIGRP (Internal)                90          Within same AS
IGRP (legacy)                   100         Obsolete Cisco protocol
OSPF                            110         Open standard link-state
IS-IS                           115         Used by ISPs
RIP (v1 and v2)                 120         Oldest dynamic protocol
EIGRP (External)                170         Redistributed into EIGRP
Internal BGP (iBGP)             200         Within same AS (iBGP)
Unknown/Unusable                255         Route NEVER installed in table
```

> 💡 **Memory trick:** "**D**ogs **E**at **O**ld **I**nteresting **R**otten **E**ggs **I**nstead **B**eing **I**gnored"  
> **D**irectly=0, **E**IGRP=90, **O**SPF=110, **I**S-IS=115, **R**IP=120, **E**IGRP-ext=170, **i**BGP=200

### AD vs Metric — Critical Difference

```
AD:     Used to choose between DIFFERENT routing PROTOCOLS
        OSPF route vs EIGRP route → AD decides

Metric: Used to choose between routes from the SAME protocol
        OSPF has two paths → metric (cost) decides which is best

They work at different levels:
  First: AD filters which protocol's routes win
  Then:  Metric determines the best path within that protocol
```

### Verifying AD

```cisco
! View AD of a specific route
Router# show ip route 192.168.2.0
Routing entry for 192.168.2.0/24
  Known via "ospf 1", distance 110, metric 2, type intra area
  Last update from 10.0.0.2 on Serial0/0/0, 00:20:15 ago
  Routing Descriptor Blocks:
  * 10.0.0.2, from 10.0.0.2, 00:20:15 ago, via Serial0/0/0
      Route metric is 2, traffic share count is 1
```

---

## 4. Static Routing

### When to Use Static Routes

```
Good use cases for static routes:
  ✓ Stub networks (only ONE path in/out — no need for dynamic protocol)
  ✓ Default route to ISP (0.0.0.0/0)
  ✓ Small networks (2-3 routers — dynamic protocol overhead not worth it)
  ✓ Specific host routes (/32) for management or policy
  ✓ Backup paths (floating static routes)
  ✓ Summary routes to reduce routing table size

Bad use cases for static routes:
  ✗ Large networks (hundreds of routes = hours of admin work)
  ✗ Networks with frequent topology changes
  ✗ Networks requiring automatic failover
  ✗ Any network where you want automatic route learning
```

### Static Route Syntax

```cisco
ip route [destination-network] [subnet-mask] [next-hop-IP | exit-interface] [AD]

Parameters:
  destination-network  → network you want to reach
  subnet-mask          → mask of that network
  next-hop-IP          → IP of adjacent router's interface
  exit-interface       → local interface to send packet out
  AD                   → optional, default=1, higher=floating static
```

### Four Ways to Write a Static Route

```cisco
! Method 1: Next-hop IP only (most common for Ethernet)
Router(config)# ip route 192.168.2.0 255.255.255.0 10.0.0.2
  Advantage:  Clear and explicit — always performs ARP to resolve MAC
  Disadvantage: Extra ARP lookup required
  Best for:   Multi-access Ethernet networks

! Method 2: Exit interface only
Router(config)# ip route 192.168.2.0 255.255.255.0 GigabitEthernet0/1
  Advantage:  No ARP — directly sends out that interface
  Disadvantage: On Ethernet, router sends ARP for EVERY destination
                (proxy ARP needed — inefficient!)
  Best for:   Point-to-point serial links ONLY

! Method 3: Both exit interface AND next-hop IP (RECOMMENDED)
Router(config)# ip route 192.168.2.0 255.255.255.0 GigabitEthernet0/1 10.0.0.2
  Advantage:  Fully specified — most efficient
              No proxy ARP issue
              Recursive lookup avoided
  Best for:   All scenarios (especially Ethernet)

! Method 4: Next-hop IP with custom AD (floating static)
Router(config)# ip route 192.168.2.0 255.255.255.0 10.0.0.2 150
  Used as backup route — see Section 5
```

### Recursive Static Routes

```
Recursive lookup happens when next-hop is not directly connected:

Router# show ip route
  S  192.168.5.0/24 [1/0] via 10.0.0.2
  C  10.0.0.0/30 is directly connected, Gi0/1

To forward to 192.168.5.0/24:
  Step 1: Look up 192.168.5.x → matches static: via 10.0.0.2
  Step 2: Look up 10.0.0.2    → matches connected: via Gi0/1 ← found!
  Forward out Gi0/1 to 10.0.0.2

If 10.0.0.2 is ALSO a static route (not connected):
  Step 1: via 10.0.0.2
  Step 2: via 172.16.1.1 (another static)
  Step 3: via Gi0/0 (finally connected!)

Multiple recursive lookups = overhead. Use fully specified routes to avoid.
```

### Complete Static Routing Lab Example

```
Topology:
PC-A            R1              R2              PC-B
192.168.1.10 ── 192.168.1.1    10.0.0.2 ── 192.168.2.10
                 Gi0/0  Gi0/1──Gi0/0  Gi0/1
                        10.0.0.1

Networks:
  192.168.1.0/24 — R1 LAN (left side)
  10.0.0.0/30    — R1-R2 link
  192.168.2.0/24 — R2 LAN (right side)
```

```cisco
!════════════ R1 CONFIGURATION ════════════!

! Interface configuration
R1(config)# interface GigabitEthernet0/0
R1(config-if)# description "LAN - 192.168.1.0/24"
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown

R1(config)# interface GigabitEthernet0/1
R1(config-if)# description "Link to R2"
R1(config-if)# ip address 10.0.0.1 255.255.255.252
R1(config-if)# no shutdown

! Static route to R2's LAN
R1(config)# ip route 192.168.2.0 255.255.255.0 GigabitEthernet0/1 10.0.0.2
!                     network to reach             exit-iface       next-hop

!════════════ R2 CONFIGURATION ════════════!

R2(config)# interface GigabitEthernet0/0
R2(config-if)# description "Link to R1"
R2(config-if)# ip address 10.0.0.2 255.255.255.252
R2(config-if)# no shutdown

R2(config)# interface GigabitEthernet0/1
R2(config-if)# description "LAN - 192.168.2.0/24"
R2(config-if)# ip address 192.168.2.1 255.255.255.0
R2(config-if)# no shutdown

! Static route to R1's LAN
R2(config)# ip route 192.168.1.0 255.255.255.0 GigabitEthernet0/0 10.0.0.1

!════════════ VERIFICATION ════════════!

R1# show ip route
C    10.0.0.0/30 is directly connected, GigabitEthernet0/1
L    10.0.0.1/32 is directly connected, GigabitEthernet0/1
C    192.168.1.0/24 is directly connected, GigabitEthernet0/0
L    192.168.1.1/32 is directly connected, GigabitEthernet0/0
S    192.168.2.0/24 [1/0] via 10.0.0.2, GigabitEthernet0/1

R1# ping 192.168.2.10 source 192.168.1.1
!!!!!  ← 5 exclamation marks = 100% success

R1# traceroute 192.168.2.10
  1  10.0.0.2  (R2 Gi0/0)
  2  192.168.2.10 (PC-B)
```

### Static Route Verification Commands

```cisco
Router# show ip route static               ! only static routes
Router# show ip route 192.168.2.0          ! specific network detail
Router# show ip route 192.168.2.0 255.255.255.0  ! with mask
Router# show running-config | include ip route   ! all static routes in config

! Test connectivity
Router# ping 192.168.2.10                  ! basic ping
Router# ping 192.168.2.10 source Gi0/0    ! ping from specific interface
Router# ping 192.168.2.10 repeat 100      ! send 100 pings
Router# traceroute 192.168.2.10           ! path trace
Router# traceroute 192.168.2.10 source Gi0/0
```

---

## 5. Floating Static Routes

### What Is a Floating Static Route?

A **floating static route** is a static route with an AD **higher than the primary dynamic routing protocol**. It "floats" above the dynamic route and only becomes active when the dynamic route disappears.

```
Normal situation:
  OSPF learns 192.168.5.0/24 via 10.0.0.2 (AD=110)
  Static route: 192.168.5.0/24 via 10.0.0.3 (AD=150)

  Routing table shows:    O 192.168.5.0/24 [110/2] via 10.0.0.2   (OSPF wins)
  Floating static hidden: NOT in table (150 > 110, so OSPF route beats it)

OSPF link fails:
  OSPF route 192.168.5.0/24 removed from routing table
  Floating static (AD=150) NOW installs in routing table!
  Traffic continues via backup path!

OSPF link recovers:
  OSPF re-learns 192.168.5.0/24 (AD=110)
  110 < 150 → OSPF route wins again
  Floating static disappears from table again
  Automatic failover and failback!
```

### Floating Static Route Configuration

```cisco
! Topology:
!   R1 ──Primary OSPF link──► R2 (10.0.0.2)
!   R1 ──Backup 4G/LTE──────► R3 (172.16.1.1) ──► R2 (same destinations)

! Primary path: OSPF (AD=110) — dynamic, automatic
Router(config)# router ospf 1
Router(config-router)# network 10.0.0.0 0.0.0.3 area 0

! Backup path: Floating static with AD=150 (higher than OSPF=110)
Router(config)# ip route 192.168.5.0 255.255.255.0 172.16.1.1 150
!                                                              ↑ AD=150

! If OSPF = primary (AD=110):
!   Static with AD=150 is NOT in routing table (150 > 110, OSPF wins)
! If OSPF fails:
!   Static with AD=150 ENTERS routing table (only route available)
! If OSPF recovers:
!   Static with AD=150 LEAVES routing table (OSPF wins again)

! Verify — during normal operation:
Router# show ip route 192.168.5.0
Routing entry for 192.168.5.0/24
  Known via "ospf 1", distance 110, metric 2

! After OSPF fails:
Router# show ip route 192.168.5.0
Routing entry for 192.168.5.0/24
  Known via "static", distance 150, metric 0
  * 172.16.1.1
```

### Common Floating Static AD Values

```
Primary protocol      Primary AD    Floating Static AD (must be higher)
────────────────────  ──────────    ────────────────────────────────────
EIGRP internal        90            Use 91–109 (e.g., 100)
OSPF                  110           Use 111–119 (e.g., 115)
RIP                   120           Use 121+ (e.g., 130)
No dynamic routing    1             Use 2+ (e.g., 5)

Rule: Floating static AD must be HIGHER than primary protocol's AD
      to remain hidden when primary route exists
```

---

## 6. Default Routes

### What Is a Default Route?

The default route **0.0.0.0/0** matches **any and all** destinations not found in the routing table. It is the "route of last resort."

```
Routing Table (no default route):
  C  192.168.1.0/24 directly connected Gi0/0
  S  10.0.0.0/8 via 172.16.1.1
  O  172.16.0.0/12 via 10.0.0.2

Packet arrives for 8.8.8.8:
  Check 192.168.1.0/24 → no match
  Check 10.0.0.0/8 → no match
  Check 172.16.0.0/12 → no match
  No more routes → DROP! Send ICMP Unreachable to source

Routing Table (WITH default route):
  C  192.168.1.0/24 directly connected Gi0/0
  S* 0.0.0.0/0 via 203.0.113.1  ← catches everything else!

Packet for 8.8.8.8:
  192.168.1.0/24 → no match
  0.0.0.0/0 → MATCH! → forward to 203.0.113.1 (ISP router)
```

### Configuring Default Routes

```cisco
! Method 1: Static default route (most common — points to ISP)
Router(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1
!                         ↑ network ↑ mask  ↑ next-hop (ISP gateway)

! Method 2: Static default route via exit interface
Router(config)# ip route 0.0.0.0 0.0.0.0 GigabitEthernet0/1

! Method 3: Fully specified (recommended)
Router(config)# ip route 0.0.0.0 0.0.0.0 GigabitEthernet0/1 203.0.113.1

! Verify default route
Router# show ip route 0.0.0.0
Routing entry for 0.0.0.0/0, supernet
  Known via "static", distance 1, metric 0, candidate default path
  * 203.0.113.1, via GigabitEthernet0/1

Router# show ip route
Gateway of last resort is 203.0.113.1 to network 0.0.0.0
S*    0.0.0.0/0 [1/0] via 203.0.113.1

! The "Gateway of last resort" line confirms default route is set
```

### Advertising Default Route to Other Routers

```cisco
! In OSPF — advertise default route to all OSPF neighbors
Router(config)# router ospf 1
Router(config-router)# default-information originate
! This router generates and advertises Type 5 LSA for 0.0.0.0/0
! All OSPF routers learn the default route

! With "always" keyword — advertise even if no default route in table
Router(config-router)# default-information originate always

! In EIGRP — redistribute default route
Router(config)# router eigrp 100
Router(config-router)# redistribute static   ! redistributes all static routes incl. default

! In RIP
Router(config)# router rip
Router(config-router)# default-information originate
```

---

## 7. Dynamic Routing Overview

### Why Dynamic Routing?

```
STATIC ROUTING PROBLEMS at scale:
  100 routers × 99 routes each = 9,900 static route commands!
  One new network added = update 100 routers manually
  One link fails = manually update alternate paths
  Human error = routing loops, black holes, outages

DYNAMIC ROUTING SOLUTION:
  Routers exchange routing information AUTOMATICALLY
  New networks propagate to all routers in minutes
  Link failures detected → alternate paths calculated
  Administrator only configures the protocol — not every route
```

### Routing Protocol Categories

```
DYNAMIC ROUTING PROTOCOLS
│
├── IGP (Interior Gateway Protocol) — used WITHIN one AS
│   │
│   ├── DISTANCE VECTOR — share routing table with neighbors
│   │   ├── RIP v1/v2   (hop count metric, simple, legacy)
│   │   └── EIGRP       (composite metric, Cisco proprietary)
│   │
│   └── LINK STATE — build complete topology map
│       ├── OSPF        (cost metric, open standard, most common)
│       └── IS-IS       (cost metric, used by large ISPs)
│
└── EGP (Exterior Gateway Protocol) — used BETWEEN AS
    └── BGP             (path attributes, THE internet routing protocol)
```

### Distance Vector vs Link State

```
DISTANCE VECTOR ("routing by rumor"):
  ┌──────────────────────────────────────────────────────────────┐
  │ Each router only knows:                                      │
  │   • What its NEIGHBORS told it about distant networks        │
  │   • Direction (which neighbor to send traffic to)           │
  │   • Distance (hop count or metric to destination)           │
  │                                                              │
  │ Process:                                                     │
  │   R1 shares its routing table with R2                       │
  │   R2 shares its routing table with R3                       │
  │   R3 "hears" about R1's networks through R2                 │
  │   "I heard from R2 that R1 is 2 hops away"                  │
  │                                                              │
  │ Convergence: SLOW (must pass through every hop)             │
  │ Updates: Periodic (RIP every 30 seconds)                    │
  │ Prone to: Routing loops (must use split horizon, etc.)      │
  │ Memory: Low (only stores distance/direction)                │
  │ Examples: RIP, EIGRP (hybrid)                               │
  └──────────────────────────────────────────────────────────────┘

LINK STATE ("complete road map"):
  ┌──────────────────────────────────────────────────────────────┐
  │ Each router builds complete topology of the network:        │
  │   • Floods LSAs (Link State Advertisements) to ALL routers  │
  │   • Every router has IDENTICAL Link State Database (LSDB)   │
  │   • Each router runs Dijkstra SPF algorithm independently   │
  │   • Each router calculates its OWN best path to everywhere  │
  │                                                              │
  │ Convergence: FAST (triggered updates, not periodic)         │
  │ Updates: Triggered (only when topology changes)             │
  │ Less prone to: Routing loops (full topology known)          │
  │ Memory: Higher (stores full topology map)                   │
  │ CPU: Higher (runs SPF algorithm)                            │
  │ Examples: OSPF, IS-IS                                       │
  └──────────────────────────────────────────────────────────────┘
```

### Loop Prevention Mechanisms (Distance Vector)

```
SPLIT HORIZON:
  Rule: Do NOT advertise a route back out the interface you learned it from
  Prevents: Simple 2-router loops
  
  R1 learns 10.0.0.0/8 from R2 via Gi0/1
  Split horizon: R1 will NOT advertise 10.0.0.0/8 back out Gi0/1 to R2
  R2 cannot "hear about its own route" from R1 → no loop

ROUTE POISONING:
  When a route fails, advertise it with metric = INFINITY (unreachable)
  RIP: infinity = 16 hops
  Forces all routers to immediately remove the route

POISON REVERSE:
  After learning a route is unreachable via poisoning,
  immediately send back infinity metric to confirm removal

HOLDDOWN TIMER:
  After a route is marked unreachable, ignore updates about it for a period
  Prevents false "good news" from being accepted prematurely
  RIP holddown = 180 seconds
```

---

## 8. OSPF — Open Shortest Path First

### OSPF Key Facts

```
Feature              Value / Description
───────────────────  ───────────────────────────────────────────────
Standard             RFC 2328 (OSPFv2 for IPv4), RFC 5340 (OSPFv3 for IPv6)
Type                 Link-state IGP
AD                   110
Metric               Cost (based on interface bandwidth)
Algorithm            Dijkstra SPF (Shortest Path First)
Protocol number      IP Protocol 89 (not TCP or UDP — runs directly on IP)
Multicast addresses  224.0.0.5 (AllSPFRouters — all OSPF routers)
                     224.0.0.6 (AllDRouters — only DR and BDR)
Hello interval       10 seconds (broadcast/point-to-point networks)
                     30 seconds (NBMA networks)
Dead interval        4× Hello = 40 seconds (default)
Updates              Triggered (only on topology changes)
Authentication       Supported (MD5, SHA)
```

### OSPF Cost — The Metric

```
Cost = Reference Bandwidth / Interface Bandwidth

Default Reference Bandwidth = 100 Mbps (100,000,000 bps)

Interface         Bandwidth      Cost Calculation         Cost
──────────────    ──────────     ─────────────────────    ────
Serial (T1)       1.544 Mbps     100/1.544               64
FastEthernet      100 Mbps       100/100                  1
GigabitEthernet   1 Gbps         100/1000                 0.1 → rounded to 1
10GigEthernet     10 Gbps        100/10000                0.01 → rounded to 1

⚠️ PROBLEM: FastEthernet AND GigabitEthernet BOTH get cost=1!
   OSPF cannot differentiate between 100Mbps and 1Gbps links!
   This means it might choose a 100Mbps path over a 1Gbps path!

FIX: Increase reference bandwidth to match your fastest links:
Router(config-router)# auto-cost reference-bandwidth 1000
! Now: GigE = 1, FastEthernet = 10 — properly differentiated

! For 10G networks:
Router(config-router)# auto-cost reference-bandwidth 10000
! GigE = 10, 10GigE = 1

! IMPORTANT: Must set SAME reference bandwidth on ALL OSPF routers
!            Inconsistent values cause suboptimal routing
```

### OSPF Areas — Hierarchical Design

```
OSPF uses a TWO-LEVEL hierarchy:
  Area 0 (Backbone Area) — MUST exist
  Other areas — all must connect to Area 0

WHY AREAS?
  Reduces LSDB size (only intra-area LSAs flooded within area)
  Limits SPF recalculation scope (topology change stays in area)
  Summarization possible at ABR (Area Border Router)
  Scales to large networks (thousands of routers)

┌─────────────────────────────────────────────────────────────────┐
│                    Area 0 (Backbone)                            │
│   [R1]────[R2]────[R3]                                          │
│    │                │                                           │
│   ABR1             ABR2                                         │
└────┼────────────────┼────────────────────────────────────────────┘
     │                │
  Area 1           Area 2
  [R4]─[R5]       [R6]─[R7]─[ASBR]──External (BGP)
  [R6]─[R7]       [R8]─[R9]

Router Types:
  IR   (Internal Router)  — all interfaces in SAME area
  ABR  (Area Border Router) — interfaces in MULTIPLE areas
                              summarizes routes between areas
  ASBR (AS Boundary Router) — connects OSPF to EXTERNAL routing domains
                               (BGP, EIGRP, static routes redistribution)
  Backbone Router — at least one interface in Area 0

Virtual Links:
  If an area cannot connect directly to Area 0:
  Configure a "virtual link" through a transit area
  (Try to avoid — use proper topology instead)
```

### OSPF Router ID

```
Router ID (RID) = 32-bit number in dotted-decimal format
Uniquely identifies each OSPF router

Selection priority (highest wins):
  1. Manually configured: Router(config-router)# router-id 1.1.1.1
  2. Highest IP on any Loopback interface (up/up)
  3. Highest IP on any active physical interface (up/up)

BEST PRACTICE: Always manually configure Router-ID
  Use format: [area].[router-number].0.0 or 1.1.1.1, 2.2.2.2, etc.
  Loopback IPs are also popular choices

Router-ID impact:
  DR/BDR election on multi-access networks
  Identification in OSPF database
  Changing RID requires OSPF process restart!
```

### OSPF Neighbor States — Full Adjacency Process

```
State       Description
─────────   ──────────────────────────────────────────────────────
DOWN        No Hellos received from this neighbor
INIT        Hello received, but our RID not yet in neighbor's Hello
2-WAY       Our RID seen in neighbor's Hello — bidirectional comm
            DR/BDR election occurs here (on multi-access networks)
EXSTART     Determine Master/Slave for DBD exchange (higher RID = Master)
EXCHANGE    Exchange Database Description (DBD) packets
            Each router sends summary of its LSDB
LOADING     Send LSR (Link State Request) for missing LSAs
            Receive LSU (Link State Update) with missing LSAs
            Send LSAck (acknowledgement)
FULL ✅     Databases are synchronized — adjacency complete!
            Routers can now exchange routing information

Only FULL state means full adjacency — anything else = problem!
```

### DR and BDR — Designated Router Election

On **multi-access networks** (Ethernet), OSPF elects a DR and BDR to reduce LSA flooding overhead.

```
WITHOUT DR/BDR (5 routers on same subnet):
  Every router forms adjacency with every other router
  5 routers = 5×4/2 = 10 adjacencies
  Each topology change → 10 LSA updates

WITH DR/BDR:
  All routers form FULL adjacency with DR and BDR only
  Routers reach 2-WAY state with each other (not FULL)
  5 routers = only 4+4 = 8 adjacencies (to DR and BDR)
  Each topology change → DR floods to all → much more efficient

DR  = Designated Router (primary — receives and floods LSAs)
BDR = Backup DR (takes over if DR fails)
DROther = non-DR/BDR routers (2-WAY with each other, FULL with DR/BDR)

Multi-access types needing DR/BDR: Ethernet (most common)
Point-to-point: NO DR/BDR (only 2 routers — no need)
```

### DR/BDR Election

```
Election priority — HIGHEST wins:
  1. Highest OSPF interface priority (default = 1, range 0-255)
     Priority 0 = INELIGIBLE (will never become DR or BDR)
  2. Highest Router-ID (tiebreaker)

ELECTION IS NON-PREEMPTIVE:
  Once DR is elected, it stays DR even if a higher-priority
  router joins the network — DR only changes when it fails!

  This means:
    First router to come up (if same priority) often becomes DR
    Order of startup matters when priorities are equal

Setting priority:
Router(config-if)# ip ospf priority 100   ! high priority = more likely to be DR
Router(config-if)# ip ospf priority 0     ! ineligible for DR/BDR

Verify DR/BDR:
Router# show ip ospf neighbor
Neighbor ID   Pri  State         Dead Time  Address      Interface
10.0.0.2       1   FULL/DR       00:00:38   10.0.0.2     Gi0/1
10.0.0.3       1   FULL/BDR      00:00:38   10.0.0.3     Gi0/1
10.0.0.4       1   2WAY/DROTHER  00:00:38   10.0.0.4     Gi0/1
```

### OSPF Neighbor Requirements (Hello Must Match)

```
Two OSPF routers will NOT form adjacency if these don't match:
  ✓ Area ID (must be same on the shared link)
  ✓ Hello interval (default 10s)
  ✓ Dead interval (default 40s)
  ✓ Authentication (type and key)
  ✓ Stub area flag
  ✓ MTU (must be same — or use ip ospf mtu-ignore)
  ✓ Network type must be compatible

These CAN differ:
  Router ID
  OSPF process ID
  Interface cost
  Reference bandwidth (local significance only)
```

### OSPF Configuration — Classic Method

```cisco
!════════════ ROUTER 1 (R1) ════════════!

R1(config)# router ospf 1
! "1" = process ID — locally significant (doesn't need to match neighbors)

R1(config-router)# router-id 1.1.1.1       ! explicitly set RID
R1(config-router)# auto-cost reference-bandwidth 1000  ! adjust for GigE

! Advertise networks (using wildcard mask = inverse of subnet mask)
R1(config-router)# network 192.168.1.0 0.0.0.255 area 0
! Wildcard: 0.0.0.255 = 255.255.255.0 inverted
! Matches any interface with IP in 192.168.1.0/24 range

R1(config-router)# network 10.0.0.0 0.0.0.3 area 0
! Wildcard: 0.0.0.3 = 255.255.255.252 inverted
! Matches interfaces in 10.0.0.0/30 range

! Prevent OSPF Hellos on LAN interface (no OSPF neighbors there)
R1(config-router)# passive-interface GigabitEthernet0/0
! Network STILL advertised into OSPF — just no Hellos sent
! Prevents unauthorized routers from forming adjacency

! Advertise default route to all OSPF neighbors
R1(config-router)# default-information originate

!════════════ ROUTER 2 (R2) ════════════!

R2(config)# router ospf 1
R2(config-router)# router-id 2.2.2.2
R2(config-router)# auto-cost reference-bandwidth 1000
R2(config-router)# network 10.0.0.0 0.0.0.3 area 0
R2(config-router)# network 192.168.2.0 0.0.0.255 area 0
R2(config-router)# passive-interface GigabitEthernet0/1
```

### OSPF Configuration — Per-Interface Method (Modern, Preferred)

```cisco
! Modern approach: configure OSPF directly on interface
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip ospf 1 area 0     ! process 1, area 0
R1(config-if)# ip ospf priority 100  ! make this router preferred DR

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip ospf 1 area 0
R1(config-if)# ip ospf cost 10       ! manually set cost

! Router config still needed for global settings
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# auto-cost reference-bandwidth 1000
R1(config-router)# passive-interface GigabitEthernet0/0

! Advantages of per-interface method:
! Cleaner, more explicit
! Avoids wildcard mask mistakes
! No ambiguity about which interface joins OSPF
```

### Wildcard Mask Reference

```
Subnet Mask        Wildcard Mask      CIDR    Usage
─────────────────  ─────────────────  ──────  ──────────────────────────
255.255.255.255    0.0.0.0            /32     Specific host
255.255.255.252    0.0.0.3            /30     Point-to-point link
255.255.255.248    0.0.0.7            /29     8 addresses
255.255.255.240    0.0.0.15           /28     16 addresses
255.255.255.224    0.0.0.31           /27     32 addresses
255.255.255.192    0.0.0.63           /26     64 addresses
255.255.255.128    0.0.0.127          /25     128 addresses
255.255.255.0      0.0.0.255          /24     Standard subnet
255.255.0.0        0.0.255.255        /16     Class B network
255.0.0.0          0.255.255.255      /8      Class A network
0.0.0.0            255.255.255.255    /0      Any address (= "any")

Calculation: Wildcard = 255.255.255.255 - Subnet Mask
Example: 255.255.255.0 → 255.255.255.255 - 255.255.255.0 = 0.0.0.255
```

### OSPF Verification Commands

```cisco
! Check neighbor relationships (most important first step)
Router# show ip ospf neighbor
Neighbor ID  Pri  State     Dead Time  Address    Interface
2.2.2.2       1   FULL/DR   00:00:35   10.0.0.2   GigabitEthernet0/1
3.3.3.3       1   FULL/BDR  00:00:38   10.0.0.3   GigabitEthernet0/1
! All neighbors should show FULL state

! Check OSPF routes in routing table
Router# show ip route ospf
O     192.168.2.0/24 [110/2] via 10.0.0.2, 00:15:32, GigabitEthernet0/1
O     192.168.3.0/24 [110/3] via 10.0.0.2, 00:12:10, GigabitEthernet0/1

! Check OSPF process configuration
Router# show ip ospf
 Routing Process "ospf 1" with ID 1.1.1.1
 ...
 Reference bandwidth unit is 1000 mbps
 ...

! Check LSDB (Link State Database)
Router# show ip ospf database
Router# show ip ospf database router    ! Type 1 LSAs
Router# show ip ospf database network   ! Type 2 LSAs (DR generated)
Router# show ip ospf database summary   ! Type 3 LSAs (ABR generated)
Router# show ip ospf database external  ! Type 5 LSAs (ASBR external)

! Check interface OSPF settings
Router# show ip ospf interface GigabitEthernet0/1
GigabitEthernet0/1 is up, line protocol is up
  Internet Address 10.0.0.1/30, Area 0, Attached via Interface Enable
  Process ID 1, Router ID 1.1.1.1, Network Type BROADCAST, Cost: 1
  Transmit Delay is 1 sec, State DR, Priority 1
  Designated Router (ID) 1.1.1.1, Interface address 10.0.0.1
  Backup Designated router (ID) 2.2.2.2, Interface address 10.0.0.2
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
  oob-resync timeout 40
  Hello due in 00:00:05

! OSPF troubleshooting
Router# debug ip ospf hello         ! view Hello messages (use carefully!)
Router# debug ip ospf adj           ! view adjacency formation
Router# undebug all                 ! turn off all debug
```

### OSPF Troubleshooting — Common Issues

```
PROBLEM: Neighbors stuck in INIT state
  Cause:  Hello received but our RID not in neighbor's Hello
  Check:  Are they on same subnet? Same area? Firewall blocking?

PROBLEM: Neighbors stuck in EXSTART/EXCHANGE
  Cause:  MTU mismatch between routers
  Fix:    Match MTU or: Router(config-if)# ip ospf mtu-ignore

PROBLEM: Neighbors never form — no OSPF neighbor seen
  Check:
    1. Both interfaces up/up?         show interfaces
    2. Same area configured?          show ip ospf interface
    3. Hello/Dead intervals match?    show ip ospf interface
    4. Authentication matches?        show ip ospf interface
    5. Passive interface configured?  show ip ospf
    6. ACL blocking OSPF?             show ip access-lists

PROBLEM: Route learned but not the best path expected
  Check:
    1. Reference bandwidth set consistently? show ip ospf
    2. Interface costs correct?             show ip ospf interface
    3. Is a better route from other source? show ip route [network]
```

---

## 9. EIGRP — Enhanced Interior Gateway Routing Protocol

### EIGRP Key Facts

```
Feature              Value / Description
───────────────────  ────────────────────────────────────────────────────
Standard             Cisco proprietary (originally), RFC 7868 (2016 open)
Type                 Advanced Distance Vector (also called "hybrid")
AD (internal)        90
AD (external)        170 (redistributed from other protocols)
AD (summary)         5
Metric               Composite: Bandwidth + Delay (default)
                     Also considers: Reliability, Load, MTU (not default)
Algorithm            DUAL (Diffusing Update Algorithm)
Protocol number      IP Protocol 88
Multicast address    224.0.0.10 (AllEIGRPRouters)
Hello interval       5 seconds (default on most interfaces)
                     60 seconds (on slow interfaces < T1)
Hold interval        3× Hello = 15 seconds
Updates              Partial, triggered (not periodic full table)
Authentication       MD5, SHA-256 (named mode)
```

### EIGRP vs OSPF — When to Use Which

```
Use EIGRP when:
  ✓ Cisco-only environment (all routers are Cisco)
  ✓ Want simplest configuration
  ✓ Need unequal-cost load balancing
  ✓ Already using EIGRP in existing network

Use OSPF when:
  ✓ Multi-vendor environment (mixed Cisco, Juniper, etc.)
  ✓ Need standard-based solution
  ✓ Large hierarchical network needing area design
  ✓ Company policy requires open standards
```

### EIGRP Metric — How It's Calculated

```
Composite Metric Formula:
  Metric = [K1×BW + (K2×BW)/(256-load) + K3×delay] × [K5/(reliability+K4)]

Default K values: K1=1, K2=0, K3=1, K4=0, K5=0

With defaults, simplified to:
  Metric = (10^7 / minimum_bandwidth_kbps) + (sum_of_delays / 10)
  
  where:
    bandwidth = slowest link's bandwidth in kbps
    delay = sum of all interface delays in microseconds ÷ 10

Example calculation (path through 3 routers):
  Links: 100Mbps FastEthernet → 1Gbps GigEthernet → 100Mbps FastEthernet
  
  Bandwidth:
    Minimum bandwidth = 100 Mbps = 100,000 kbps
    BW component = 10^7 / 100,000 = 100
  
  Delay:
    FE delay = 100 μs, GigE delay = 10 μs, FE delay = 100 μs
    Total delay = 210 μs → divided by 10 = 21
  
  Metric = 100 + 21 = 121 (multiplied by 256 internally → 30,976)
  
Note: K values must match on ALL EIGRP neighbors or adjacency fails!
```

### EIGRP Terminology — DUAL Algorithm

```
SUCCESSOR:
  The BEST route to a destination network
  Installed in the routing table
  Must meet the Feasibility Condition

FEASIBLE DISTANCE (FD):
  The TOTAL metric from THIS router to the destination
  via the Successor route
  FD = local metric to neighbor + neighbor's metric to destination

REPORTED DISTANCE (RD) / ADVERTISED DISTANCE (AD):
  The metric a NEIGHBOR advertises to reach a destination
  = neighbor's cost to reach the destination from itself

FEASIBLE SUCCESSOR (FS):
  A BACKUP route that meets the Feasibility Condition
  Stored in TOPOLOGY TABLE (not routing table yet)
  Instantly usable if Successor fails — no DUAL recalculation!

FEASIBILITY CONDITION:
  A neighbor's Reported Distance (RD) MUST be LESS THAN
  the current Feasible Distance (FD) of the Successor
  
  RD < FD → qualifies as Feasible Successor ✅
  RD ≥ FD → does NOT qualify (might cause a loop)

WHY THIS IS POWERFUL:
  Normal failover: Successor fails → run DUAL (recalculate)
  With Feasible Successor: Successor fails → instantly use FS!
  Convergence = milliseconds (not seconds)
```

### EIGRP Topology Table

```
Router# show ip eigrp topology
EIGRP-IPv4 Topology Table for AS(100)/ID(1.1.1.1)
Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply,
       r - reply Status, s - sia Status

P 192.168.2.0/24, 1 successors, FD is 28160
        via 10.0.0.2 (28160/2816), GigabitEthernet0/1  ← SUCCESSOR (FD/RD)
        via 172.16.1.1 (30720/2816), GigabitEthernet0/2 ← FEASIBLE SUCCESSOR

P 192.168.3.0/24, 1 successors, FD is 30720
        via 10.0.0.2 (30720/5120), GigabitEthernet0/1  ← SUCCESSOR only
        via 172.16.1.1 (33280/5120), GigabitEthernet0/2 ← NOT FS! (33280 > 30720)

Explanation for first route:
  FD = 28160 (our total cost to 192.168.2.0)
  via 10.0.0.2: cost=28160, neighbor RD=2816 → FS? 2816 < 28160 ✅ (FEASIBLE SUCCESSOR)
  via 172.16.1.1: cost=30720, neighbor RD=2816 → FS? 2816 < 28160 ✅ (also FS!)

Explanation for second route:
  FD = 30720
  via 172.16.1.1: cost=33280, neighbor RD=5120 → FS? 5120 < 30720 ✅ (IS feasible)
  Wait — then why "NOT FS"?
  Recalculate: 5120 < 30720 → IS feasible. Example was for illustration.
  
P = Passive (stable), A = Active (DUAL recalculating — link failed)
```

### EIGRP Configuration — Classic Mode

```cisco
!════════════ ROUTER 1 ════════════!

R1(config)# router eigrp 100
! "100" = Autonomous System number — MUST match all EIGRP neighbors

R1(config-router)# eigrp router-id 1.1.1.1   ! set router ID

! Advertise networks (same wildcard mask syntax as OSPF)
R1(config-router)# network 192.168.1.0 0.0.0.255
R1(config-router)# network 10.0.0.0 0.0.0.3
R1(config-router)# network 172.16.0.0 0.0.255.255

! CRITICAL: Disable auto-summarization (enabled by default in classic mode!)
R1(config-router)# no auto-summary
! Without this: EIGRP summarizes to classful boundaries
! 192.168.1.0/24 becomes 192.168.0.0/16 — WRONG!

! Passive interface (no Hellos, but still advertise network)
R1(config-router)# passive-interface GigabitEthernet0/0

! Modify Hello and Hold timers (optional — must match neighbor)
R1(config-if)# ip hello-interval eigrp 100 5
R1(config-if)# ip hold-time eigrp 100 15
```

### EIGRP Named Mode Configuration (Modern, Recommended)

```cisco
! Named mode = single config block for all address families
! Better organized, supports IPv4 and IPv6 in same config

R1(config)# router eigrp ENTERPRISE_NETWORK   ! any name works

R1(config-router)# address-family ipv4 unicast autonomous-system 100
R1(config-router-af)# eigrp router-id 1.1.1.1
R1(config-router-af)# network 192.168.1.0 0.0.0.255
R1(config-router-af)# network 10.0.0.0 0.0.0.3

! Interface-level settings within named mode
R1(config-router-af)# af-interface GigabitEthernet0/0
R1(config-router-af-interface)# passive-interface
R1(config-router-af-interface)# exit-af-interface

R1(config-router-af)# topology base
R1(config-router-af-topology)# auto-summary       ! or no auto-summary
R1(config-router-af-topology)# exit-af-topology
R1(config-router-af)# exit-address-family
```

### EIGRP Unequal-Cost Load Balancing

```
EIGRP supports UNEQUAL-COST load balancing — unique feature!
(OSPF only supports equal-cost load balancing)

VARIANCE command:
  Allows load balancing across paths with metrics up to
  [variance × best_metric]

Example:
  Best path:   metric = 30000
  Backup path: metric = 60000

  variance 2 → load balance across paths where metric ≤ 2×30000 = 60000
  Backup path (60000) ≤ 60000 → QUALIFIES for load balancing!

Router(config-router)# variance 2    ! default = 1 (equal-cost only)

Traffic distribution:
  Path 1 (metric 30000): gets 2 shares (60000/30000 = 2)
  Path 2 (metric 60000): gets 1 share (60000/60000 = 1)
  Total: Path 1 carries 2/3 of traffic, Path 2 carries 1/3

Only Feasible Successors can participate in unequal-cost load balancing!
```

### EIGRP Verification Commands

```cisco
! Check neighbor relationships
Router# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H  Address          Interface   Hold Uptime   SRTT  RTO   Q   Seq
                                (sec)         (ms)        Cnt Num
0  10.0.0.2         Gi0/1       13  01:30:45  1    200   0   15
1  172.16.1.1       Gi0/2       11  01:30:40  2    200   0   12
! Q Cnt should be 0 (no pending updates)

! Check topology table
Router# show ip eigrp topology
Router# show ip eigrp topology all-links   ! shows all paths including non-FS

! Check EIGRP routes in routing table
Router# show ip route eigrp
D     192.168.2.0/24 [90/28160] via 10.0.0.2, 01:30:45, Gi0/1

! Check EIGRP traffic/statistics
Router# show ip eigrp traffic
Router# show ip eigrp interfaces
Router# show ip eigrp interfaces detail
```

---

## 10. RIP — Routing Information Protocol

### RIP Overview

```
RIP is the OLDEST and SIMPLEST dynamic routing protocol.
Used for small/legacy networks or study/lab environments.
Maximum hop count = 15. Metric = 16 = UNREACHABLE.

RIPv1 vs RIPv2:
┌─────────────────────┬────────────────────┬────────────────────────┐
│ Feature             │ RIPv1              │ RIPv2                  │
├─────────────────────┼────────────────────┼────────────────────────┤
│ Standard            │ RFC 1058           │ RFC 2453               │
│ Classless support   │ NO (classful only) │ YES (VLSM/CIDR)       │
│ Update type         │ Broadcast          │ Multicast (224.0.0.9)  │
│ Authentication      │ NO                 │ YES (plain or MD5)     │
│ Route tag support   │ NO                 │ YES                    │
│ Next-hop field      │ NO                 │ YES                    │
│ Subnet mask in update│ NO                │ YES                    │
│ AD                  │ 120                │ 120                    │
│ Metric              │ Hop count          │ Hop count              │
│ Max hops            │ 15                 │ 15                     │
│ Update interval     │ 30 seconds         │ 30 seconds             │
│ Invalid timer       │ 180 seconds        │ 180 seconds            │
│ Holddown timer      │ 180 seconds        │ 180 seconds            │
│ Flush timer         │ 240 seconds        │ 240 seconds            │
└─────────────────────┴────────────────────┴────────────────────────┘
```

### Why RIP Has Serious Limitations

```
PROBLEM 1: Maximum 15 hops
  Any destination 16+ hops away = unreachable
  Limits RIP to very small networks

PROBLEM 2: Slow convergence
  Updates every 30 seconds
  Invalid timer = 180s (6 updates missed before route invalid)
  Holddown = 180s
  Total convergence: up to 6 minutes in worst case!

PROBLEM 3: No bandwidth consideration
  Uses hop count ONLY
  1-hop T1 link (1.5Mbps) preferred over 3-hop Gigabit links!
  RIP picks the fewest hops, not the best bandwidth

PROBLEM 4: Bandwidth consumption
  Sends FULL routing table every 30 seconds
  On large networks = significant overhead

PROBLEM 5: Routing loops
  Distance vector problems — must use timers to prevent
  Convergence is slow enough that loops can temporarily form
```

### RIP Configuration

```cisco
! RIPv2 configuration (always use v2, never v1 in production)
Router(config)# router rip
Router(config-router)# version 2                    ! enable version 2
Router(config-router)# no auto-summary              ! CRITICAL! Disable classful summarization

! Advertise networks (RIP uses CLASSFUL network — no wildcard mask!)
Router(config-router)# network 192.168.1.0          ! /24 assumed
Router(config-router)# network 10.0.0.0             ! /8 assumed (class A)
Router(config-router)# network 172.16.0.0           ! /16 assumed (class B)

! Passive interface (suppress updates on user-facing interfaces)
Router(config-router)# passive-interface GigabitEthernet0/0

! Set maximum hops (default 15, can set 1-255 but 16+ = unreachable)
Router(config-router)# maximum-paths 4              ! load balancing across 4 paths

! Authentication (v2 only)
Router(config)# key chain RIP_KEYS
Router(config-keychain)# key 1
Router(config-keychain-key)# key-string RIPSecretKey

Router(config)# interface GigabitEthernet0/1
Router(config-if)# ip rip authentication mode md5
Router(config-if)# ip rip authentication key-chain RIP_KEYS

! Verification
Router# show ip route rip
R     192.168.2.0/24 [120/1] via 10.0.0.2, 00:00:12, GigabitEthernet0/1
!                     ↑  ↑ metric (1 hop)
!                     120 = AD

Router# show ip rip database
Router# show ip protocols
Router# debug ip rip                ! view RIP updates in real time
```

---

## 11. Route Redistribution Basics

### What Is Redistribution?

Redistribution allows routes from **one routing protocol to be shared into another**. Used when multiple protocols exist in the same network (migration, mergers, multi-vendor).

```
Scenario: Company merges two networks
  Network A uses OSPF (10.0.0.0/8)
  Network B uses EIGRP (172.16.0.0/16)

  Without redistribution: OSPF routers can't reach EIGRP networks and vice versa
  With redistribution: ASBR redistributes OSPF into EIGRP and EIGRP into OSPF
```

### Basic Redistribution Commands

```cisco
!════════ ASBR — Redistributing OSPF into EIGRP ════════!
Router(config)# router eigrp 100
Router(config-router)# redistribute ospf 1 metric 1000 100 255 1 1500
!                                          ↑↑↑↑↑     ↑   ↑  ↑  ↑  ↑
!                               OSPF process BW  delay  rel load MTU
! metric must be specified when redistributing into EIGRP
! (BW=1000kbps, delay=100, reliability=255, load=1, MTU=1500)

!════════ ASBR — Redistributing EIGRP into OSPF ════════!
Router(config)# router ospf 1
Router(config-router)# redistribute eigrp 100 subnets
!                                              ↑ subnets = include /24, /30 etc.
!                                              Without "subnets" = only classful routes!

! Redistributed routes get AD = 170 in EIGRP (external)
! Redistributed routes appear as O E2 in OSPF (external type 2)

!════════ Redistributing Static into OSPF ════════!
Router(config)# router ospf 1
Router(config-router)# redistribute static subnets
Router(config-router)# redistribute connected subnets   ! also redistribute connected routes

!════════ Verify redistribution ════════!
Router# show ip route
O E2  172.16.5.0/24 [110/20] via 10.0.0.2    ← OSPF external type 2 (redistributed)
D EX  10.5.5.0/24 [170/2174976] via 10.0.0.3  ← EIGRP external (redistributed)
```

---

## 12. Routing Troubleshooting

### Systematic Troubleshooting Approach

```
STEP 1: Check physical connectivity
  Router# show interfaces GigabitEthernet0/1
  → Is it up/up? Any errors?

STEP 2: Check IP addressing
  Router# show ip interface brief
  → Correct IPs? Any interfaces down?

STEP 3: Check routing table
  Router# show ip route
  → Expected routes present?
  → Correct next-hop? Correct exit interface?

STEP 4: Test reachability
  Router# ping [destination]
  Router# ping [destination] source [source-interface]
  Router# traceroute [destination]

STEP 5: Protocol-specific checks (if dynamic routing)
  OSPF: show ip ospf neighbor → all neighbors in FULL state?
  EIGRP: show ip eigrp neighbors → all neighbors listed?
  RIP: show ip rip database → expected routes present?

STEP 6: Check for ACLs blocking traffic
  Router# show ip access-lists
  → Any unexpected deny statements? High hit counts?

STEP 7: Debug (last resort — use carefully in production!)
  Router# debug ip routing              ! route add/remove events
  Router# debug ip ospf hello           ! OSPF hello messages
  Router# debug ip eigrp                ! EIGRP updates
  Router# undebug all                   ! ALWAYS turn off debug after!
```

### Common Routing Problems and Fixes

```
PROBLEM: Route missing from routing table
  Possible causes:
  1. Interface down → connected route removed
     Fix: show interfaces → check physical, cable, shutdown
  2. Neighbor down → dynamic routes removed
     Fix: show ip ospf neighbor OR show ip eigrp neighbors
  3. Route filtered by distribute-list/route-map
     Fix: show ip protocols → check filters
  4. AD conflict (hidden by lower-AD route to same dest)
     Fix: show ip route [network] → check which protocol won

PROBLEM: Traffic going wrong path
  Possible causes:
  1. Longest prefix match selecting unexpected route
     Fix: show ip route [exact-destination-IP] → see which route matches
  2. Suboptimal metric (e.g., OSPF cost not adjusted for GigE)
     Fix: Adjust reference bandwidth or manual costs
  3. Equal-cost load balancing distributing to slow path
     Fix: Adjust costs to make one path preferred

PROBLEM: Intermittent connectivity (up/down)
  Possible causes:
  1. Flapping interface (physical issue)
     Fix: show interfaces → check input errors, carrier transitions
  2. Hello/Dead timer mismatch → neighbor keeps going down
     Fix: show ip ospf interface → verify timers match both ends
  3. Routing loop (rare but devastating)
     Fix: traceroute → look for repeating hops

PROBLEM: "Destination unreachable" on ping
  Possible causes:
  1. No route in routing table (or no default route)
     Fix: show ip route → is destination there?
  2. Route exists but return path missing (asymmetric routing)
     Fix: ping from remote side → is return path also there?
  3. ACL blocking
     Fix: show ip access-lists → check deny counts increasing

PROBLEM: OSPF neighbors not forming
  Checklist:
  □ Both interfaces up/up?
  □ Same area configured?
  □ Same Hello/Dead timers?
  □ Same authentication?
  □ MTU matches (or ip ospf mtu-ignore)?
  □ Passive-interface blocking Hellos?
  □ ACL blocking IP protocol 89?
```

---

## 13. Key Takeaways & Exam Tips

### Key Takeaways

- Routers use **longest prefix match** — most specific route always wins
- **Administrative Distance** = trustworthiness of routing source; lower = preferred
- AD order: Connected(0) > Static(1) > EIGRP(90) > OSPF(110) > IS-IS(115) > RIP(120)
- **Static routes** use AD=1; **floating static** uses higher AD as backup
- Default route = **0.0.0.0/0** — matches all destinations not in routing table
- **OSPF** = link-state, AD=110, cost metric, Area 0 required, open standard
- **OSPF cost** = Reference BW / Interface BW — adjust with `auto-cost reference-bandwidth`
- **OSPF neighbors** must match: area, Hello/Dead timers, authentication, MTU
- **DR/BDR** elected on multi-access (Ethernet) networks — highest priority wins, non-preemptive
- **EIGRP** = advanced distance vector, AD=90, composite metric (BW+Delay), Cisco proprietary
- **EIGRP DUAL** — Feasible Successor = instant backup route (no recalculation)
- **RIP** = hop count metric, max 15 hops, 30-second updates, legacy protocol
- `passive-interface` = stops Hellos but still ADVERTISES the network
- `no auto-summary` = must disable in EIGRP (classic) and RIP for VLSM support

### Exam Tips

> 🎯 **Most Tested Topics:**

1. **"OSPF Administrative Distance?"** → **110**
2. **"EIGRP internal AD?"** → **90** | **EIGRP external AD?** → **170**
3. **"RIP AD?"** → **120** | **RIP max hops?** → **15** (16 = unreachable)
4. **"Which routing protocol uses hop count?"** → **RIP**
5. **"OSPF metric is called?"** → **Cost** = Reference BW / Interface BW
6. **"Default OSPF reference bandwidth?"** → **100 Mbps** — fix with `auto-cost reference-bandwidth 1000`
7. **"OSPF uses what multicast?"** → **224.0.0.5** (all routers), **224.0.0.6** (DR/BDR only)
8. **"OSPF neighbor adjacency — what state means fully synced?"** → **FULL**
9. **"OSPF DR election — what wins?"** → **Highest interface priority**, then highest Router-ID
10. **"What is a Feasible Successor in EIGRP?"** → Backup route where neighbor's RD < local FD
11. **"What does `passive-interface` do?"** → Stops sending Hello/update packets on that interface — network still advertised
12. **"What is a floating static route?"** → Static route with AD higher than primary dynamic protocol — acts as backup
13. **"EIGRP AS number — must it match?"** → **YES** — neighbors must have same AS number
14. **"What command fixes OSPF cost for GigabitEthernet?"** → `auto-cost reference-bandwidth 1000`
15. **"What does `no auto-summary` do in EIGRP?"** → Disables classful summarization — required for VLSM support

---

*← Previous: [06 — Switching & VLANs](./06_Switching_VLANs.md)*  
*Next: [08 — ACLs & Security →](./08_ACLs_Security.md)*
