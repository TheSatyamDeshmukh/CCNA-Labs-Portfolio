# 09 — WAN Technologies & SD-WAN: Complete Reference

> **CCNA Exam Domain:** 3.0 IP Connectivity / 6.0 Automation  
> **Exam Weight:** ~10% of total exam  
> **Difficulty:** ⭐⭐⭐☆☆

---

## Table of Contents

1. [WAN Fundamentals](#1-wan-fundamentals)
2. [WAN Topology Types](#2-wan-topology-types)
3. [Traditional WAN Connection Types](#3-traditional-wan-connection-types)
4. [WAN Encapsulation Protocols — HDLC & PPP](#4-wan-encapsulation-protocols--hdlc--ppp)
5. [PPPoE — PPP over Ethernet](#5-pppoe--ppp-over-ethernet)
6. [MPLS — Multiprotocol Label Switching](#6-mpls--multiprotocol-label-switching)
7. [Internet-Based WAN Options](#7-internet-based-wan-options)
8. [GRE Tunnels](#8-gre-tunnels)
9. [IPSec over GRE — Secure Tunnels](#9-ipsec-over-gre--secure-tunnels)
10. [SD-WAN — Software Defined WAN](#10-sd-wan--software-defined-wan)
11. [Cisco SD-WAN Architecture (Viptela)](#11-cisco-sd-wan-architecture-viptela)
12. [WAN Troubleshooting](#12-wan-troubleshooting)
13. [Key Takeaways & Exam Tips](#13-key-takeaways--exam-tips)

---

## 1. WAN Fundamentals

### What Is a WAN and Why Does It Exist?

A **WAN (Wide Area Network)** is any network that spans geographically separate locations — typically connecting branch offices to a headquarters, or connecting an enterprise to the internet. While a LAN (Local Area Network) is something you own and fully control (your switches, cables, routers), a WAN almost always involves a **service provider** — a company that owns the infrastructure between your locations and leases you access to it.

This distinction is fundamental to understanding WAN design. Because you don't own the WAN infrastructure, you cannot control its internal workings. You hand traffic to the service provider at one end and receive it at the other, trusting that the provider's network carries it faithfully. This dependency on a third party drives many WAN design decisions around redundancy, monitoring, and security.

```
YOUR NETWORK                    SERVICE PROVIDER                YOUR NETWORK
┌──────────────┐                ┌─────────────────────┐        ┌──────────────┐
│  HQ Office   │                │                     │        │ Branch       │
│              │                │   SP Backbone        │        │ Office       │
│  [Router]────┼────[Last Mile]─┼─[PE]────[P]────[PE]─┼────────┼─[Router]     │
│              │                │                     │        │              │
└──────────────┘                └─────────────────────┘        └──────────────┘
   CPE (yours)      Local Loop    Provider Network                CPE (yours)
```

### Essential WAN Terminology

Understanding WAN terminology is important because exam questions often test whether you know which part of the network each term refers to.

**CPE (Customer Premises Equipment)** is the router or modem located at the customer's physical site. This is the equipment you — the customer — own or manage. The CPE is where your LAN ends and the WAN begins.

**The Local Loop** (sometimes called the "last mile") is the physical connection between your CPE and the service provider's nearest facility. This is typically the most expensive and slowest part of the WAN path, and it is the most common point of failure.

**CO (Central Office)** is the service provider's local facility where your local loop terminates. Think of it as the first "onramp" into the SP's network.

**POP (Point of Presence)** is a location where the SP's network is accessible. Large SPs have POPs in major cities worldwide, allowing customers to connect to their backbone.

**The CSU/DSU (Channel Service Unit / Data Service Unit)** is a device that terminates a digital leased line (like a T1) at the customer site. It converts between the LAN signal format and the WAN signal format. On modern Cisco routers, this functionality is often built into the WAN interface card, so you may not see a separate CSU/DSU device.

**DCE (Data Communications Equipment)** is the device that provides the clocking signal on a serial WAN link. In a real deployment, the SP's equipment is the DCE. In a lab with a back-to-back serial cable between two routers, one router must be manually configured as the DCE.

**DTE (Data Terminal Equipment)** is the customer's device (your router) that receives the clock signal from the DCE.

```cisco
! In a lab — determine which end of the serial cable is DCE
Router# show controllers serial 0/0/0
Interface Serial0/0/0
  Hardware is GT96K
  DCE V.35, clock rate 2000000    ← this router is DCE, must set clock rate

! Set clock rate on DCE end (lab only — not needed with real SP equipment)
Router(config)# interface Serial0/0/0
Router(config-if)# clock rate 128000   ! 128 kbps
```

---

## 2. WAN Topology Types

The way you connect your sites together creates different trade-offs between cost, redundancy, and complexity. Knowing these topologies is important both for the exam and for real-world design decisions.

### Hub-and-Spoke (Star Topology)

This is the most common and cost-effective WAN topology. One central site (the hub — typically HQ) connects to all remote sites (spokes — branches). All inter-branch traffic must pass through the hub.

```
              ┌─────────────┐
              │     HQ      │
              │   (Hub)     │
              └──────┬──────┘
           ┌─────────┼─────────┐
           │         │         │
      ┌────┴───┐ ┌───┴────┐ ┌──┴─────┐
      │Branch-A│ │Branch-B│ │Branch-C│
      │(Spoke) │ │(Spoke) │ │(Spoke) │
      └────────┘ └────────┘ └────────┘

Advantages:
  + Cheapest — each branch needs only ONE WAN connection
  + Simple to manage — all routing centers at HQ
  + Easy to add new branches

Disadvantages:
  - HQ is a single point of failure (HQ goes down → all branches isolated)
  - Branch-to-Branch traffic must "hairpin" through HQ (adds latency)
  - HQ router can become a bottleneck under heavy load
```

### Partial Mesh

Some sites are directly connected to each other, but not all. Direct connections exist where traffic volume or latency requirements justify the extra cost.

```
        ┌─────────────┐
        │     HQ      │
        └──┬───────┬──┘
           │       │
      ┌────┴──┐  ┌─┴─────┐
      │Branch-A│──│Branch-B│   ← direct link (high traffic between A and B)
      └───────┘  └───────┘
           │
      ┌────┴──┐
      │Branch-C│               ← no direct link to B, uses HQ as transit
      └───────┘

Advantages:
  + More redundancy than hub-and-spoke
  + Direct links where needed = lower latency for critical paths
  + Cost-effective compromise

Disadvantages:
  - More expensive than pure hub-and-spoke
  - More complex routing configuration
```

### Full Mesh

Every site connects directly to every other site. Maximum redundancy, maximum cost.

```
     HQ ─────────── Branch-A
      │  ╲         ╱    │
      │    ╲     ╱      │
      │      ╲ ╱        │
      │      ╱ ╲        │
      │    ╱     ╲      │
      │  ╱         ╲    │
   Branch-C ──── Branch-B

Number of links needed: n(n-1)/2
  4 sites = 4×3/2 = 6 links
  10 sites = 10×9/2 = 45 links  ← very expensive!

Advantages:
  + Any site can reach any other directly
  + No single point of failure
  + Lowest latency between any two sites

Disadvantages:
  - Very expensive (many WAN circuits)
  - Complex to manage
  - Doesn't scale well (link count grows exponentially)
  - Only practical for small number of critical sites
```

### Dual-Homed (Redundant Connections to SP)

A single site connects to the service provider via two separate paths — often to two different SP routers or even two different SPs. This protects against local loop failures and SP equipment failures.

```
          ┌──────────────────────────────────────┐
          │           Service Provider           │
          │    [PE-Router-1]    [PE-Router-2]    │
          └──────────┬──────────────┬────────────┘
            Primary Link           Backup Link
                     │              │
              ┌──────┴──────────────┴──────┐
              │        HQ Router           │
              │    (dual SP connections)   │
              └────────────────────────────┘
Primary fails → backup takes over automatically via routing protocol
```

---

## 3. Traditional WAN Connection Types

### Leased Lines (Dedicated Point-to-Point)

A leased line is a **dedicated private circuit** between two locations. The SP provisions a fixed amount of bandwidth that is yours alone — not shared with any other customer. This provides predictable, consistent performance but at significant cost.

```
Circuit Type    Bandwidth       Region          Use Case
────────────    ─────────────   ─────────────   ─────────────────────────
T1              1.544 Mbps      North America   Small branch offices
E1              2.048 Mbps      Europe/Intl     Small branch offices
T3 (DS3)        44.736 Mbps     North America   Data center connectivity
E3              34.368 Mbps     Europe          Data center connectivity
OC-3            155.52 Mbps     Backbone        SP interconnects
OC-12           622.08 Mbps     Backbone        Major SP links
OC-48           2.488 Gbps      Core backbone   Major SP links
OC-192          9.953 Gbps      Core backbone   Intercontinental links
```

The key characteristic of leased lines is that they are **always on** — there is no "call setup" time. The circuit exists 24/7. This makes them reliable and suitable for real-time traffic, but it also means you pay for the bandwidth continuously regardless of whether you are using it. A T1 running at 10% utilization still costs the same as one running at 100%.

### Circuit Switching (Legacy — ISDN, PSTN)

Circuit switching establishes a **dedicated physical path** for the duration of a call, then tears it down when the call ends. The PSTN (telephone network) is the classic example. ISDN (Integrated Services Digital Network) was a digital evolution of the PSTN used for data in the 1990s and 2000s.

```
ISDN BRI (Basic Rate Interface):
  2B + D channels
  B channels: 64 Kbps each (bearer — actual data)
  D channel:  16 Kbps (delta — signaling/control)
  Total usable: 128 Kbps (both B channels bonded)

ISDN PRI (Primary Rate Interface):
  North America: 23B + D = 23 × 64K = 1.472 Mbps (fits in T1)
  Europe:        30B + D = 30 × 64K = 1.920 Mbps (fits in E1)
```

ISDN is now completely obsolete in most countries. You will not deploy it, but you may see it referenced in older documentation or exam questions.

### Packet Switching (Frame Relay, ATM — Legacy)

Packet switching shares the SP's infrastructure among multiple customers. Your traffic competes for bandwidth with other customers. Frame Relay and ATM were the dominant packet-switching WAN technologies through the 1990s and 2000s before MPLS replaced them.

**Frame Relay** used virtual circuits (PVCs — Permanent Virtual Circuits) identified by DLCIs (Data Link Connection Identifiers). Each DLCI represented a virtual connection between two endpoints.

**ATM (Asynchronous Transfer Mode)** used fixed 53-byte cells. The fixed cell size made hardware-based switching very fast and predictable, which made ATM popular for voice and video. However, the fixed small cell size (48 bytes payload + 5 bytes header) was inefficient for data.

Both Frame Relay and ATM are now retired in virtually all networks, replaced by MPLS and broadband internet.

---

## 4. WAN Encapsulation Protocols — HDLC & PPP

When data travels over a serial WAN link, it needs a Layer 2 encapsulation protocol — just as Ethernet is the Layer 2 protocol for LAN links, serial links need their own protocol to frame the data for transmission.

### HDLC — High-Level Data Link Control

**HDLC** is the default encapsulation on Cisco serial interfaces. It is simple, lightweight, and efficient — which is why it is the default. However, Cisco's implementation of HDLC is **proprietary**, meaning it includes a "Type" field that standard HDLC does not have. This allows Cisco HDLC to carry multiple Layer 3 protocols (IP, IPX, etc.), but it also means **Cisco HDLC is incompatible with non-Cisco equipment**.

```
Standard HDLC Frame:
┌────────┬─────────┬──────────┬──────────────┬─────┐
│  Flag  │ Address │ Control  │     Data     │ FCS │
│ (0x7E) │         │          │              │     │
└────────┴─────────┴──────────┴──────────────┴─────┘

Cisco HDLC Frame (proprietary addition):
┌────────┬─────────┬──────────┬──────────┬──────────────┬─────┐
│  Flag  │ Address │ Control  │   Type   │     Data     │ FCS │
│ (0x7E) │         │          │ (Cisco)  │              │     │
└────────┴─────────┴──────────┴──────────┴──────────────┴─────┘
                                   ↑ proprietary field — incompatible!

Rule: Use HDLC ONLY when both ends are Cisco routers.
      If one end is non-Cisco, use PPP instead.
```

```cisco
! HDLC is the default — this is what you see if you don't configure anything
Router# show interfaces Serial0/0/0
Serial0/0/0 is up, line protocol is up
  Hardware is GT96K
  Encapsulation HDLC, loopback not set    ← default encapsulation

! Explicitly set HDLC (usually not needed — already default)
Router(config)# interface Serial0/0/0
Router(config-if)# encapsulation hdlc

! Verify
Router# show interfaces Serial0/0/0 | include Encapsulation
  Encapsulation HDLC
```

### PPP — Point-to-Point Protocol

**PPP** is the open standard WAN encapsulation protocol (RFC 1661). It works with any vendor's equipment, making it the right choice whenever connecting to non-Cisco devices, or whenever you need PPP's additional features that HDLC lacks.

PPP is structured as a family of sub-protocols, each handling a specific aspect of the connection. Understanding this layered architecture helps you understand how PPP negotiates and establishes a link.

```
PPP Sub-protocol Architecture:
┌─────────────────────────────────────────┐
│           Network Layer (IP, IPv6...)    │
├─────────────────────────────────────────┤
│     NCP (Network Control Protocol)      │  Configures Layer 3 protocol
│  IPCP (for IP), IPv6CP (for IPv6)...   │  e.g., negotiates IP addresses
├─────────────────────────────────────────┤
│     LCP (Link Control Protocol)         │  Establishes, configures,
│     - Negotiates link parameters        │  and tests the PPP link itself
│     - Authentication (PAP or CHAP)      │  Runs first, before NCP
│     - Compression, multilink           │
├─────────────────────────────────────────┤
│           Physical Layer (Serial)       │
└─────────────────────────────────────────┘

PPP connection setup sequence:
  Step 1: LCP negotiates link parameters (MRU, auth method, compression)
  Step 2: Authentication (PAP or CHAP) if configured
  Step 3: NCP configures Layer 3 (IPCP assigns/confirms IP addresses)
  Step 4: Data transfer begins
```

### PPP Authentication — PAP vs CHAP

PPP supports two authentication methods. Understanding the difference is essential because the exam tests this.

**PAP (Password Authentication Protocol)** sends the username and password in **plain text** across the link. This is a fundamental weakness — anyone capturing the link traffic can read the credentials directly. PAP performs authentication only once during link establishment.

**CHAP (Challenge Handshake Authentication Protocol)** never sends the password across the link at all. Instead, it uses a **challenge-response mechanism** based on MD5 hashing. The authenticator sends a random challenge value; the authenticatee hashes the challenge together with the shared secret using MD5 and sends back only the hash. The authenticator performs the same calculation locally and compares results. If they match, authentication succeeds. The password itself is never transmitted. CHAP also re-authenticates periodically during the connection, making it more robust against replay attacks.

```
PAP Authentication (WEAK — password sent in clear):
Router-A ──── "username admin, password cisco123" ────► Router-B
                ↑ plain text — visible to any sniffer!

CHAP Authentication (STRONG — password never sent):
Router-B ──── Challenge: "random_value_abc123" ────► Router-A
Router-A: MD5("random_value_abc123" + "shared_secret") = hash_result
Router-A ──── Response: hash_result ────────────────► Router-B
Router-B: MD5("random_value_abc123" + "shared_secret") = same_hash?
          YES → Authentication succeeds ✅
          NO  → Authentication fails ✗
```

```cisco
! PPP with CHAP authentication — configuration on both routers

! Router-A configuration
Router-A(config)# hostname Router-A                  ! hostname used in CHAP exchange
Router-A(config)# username Router-B password ChapSecret123   ! peer's hostname + shared secret
Router-A(config)# interface Serial0/0/0
Router-A(config-if)# encapsulation ppp
Router-A(config-if)# ppp authentication chap         ! use CHAP (not PAP)
Router-A(config-if)# ip address 10.0.0.1 255.255.255.252
Router-A(config-if)# no shutdown

! Router-B configuration
Router-B(config)# hostname Router-B
Router-B(config)# username Router-A password ChapSecret123   ! must be same secret!
Router-B(config)# interface Serial0/0/0
Router-B(config-if)# encapsulation ppp
Router-B(config-if)# ppp authentication chap
Router-B(config-if)# ip address 10.0.0.2 255.255.255.252
Router-B(config-if)# no shutdown

! Key points about CHAP configuration:
! 1. The "username" command uses the PEER's hostname (not your own)
! 2. The password must be IDENTICAL on both sides
! 3. Your own hostname (set with "hostname") is what you send as identity
! 4. Hostnames are case-sensitive!

! PPP with PAP authentication (avoid in production!)
Router-A(config-if)# ppp authentication pap
Router-A(config-if)# ppp pap sent-username Router-A password ClearTextPass

! Verify PPP
Router# show interfaces Serial0/0/0
  Encapsulation PPP, LCP Open      ← LCP negotiated successfully
  Open: IPCP                        ← IP configured via NCP
Router# show ppp all
```

### HDLC vs PPP Comparison

```
Feature               HDLC (Cisco)        PPP
────────────────────  ──────────────────  ──────────────────────────
Standard              Cisco proprietary   Open (RFC 1661)
Vendor compatibility  Cisco only          Any vendor
Authentication        None                PAP or CHAP
Compression           No                  Yes (via LCP)
Error detection       Yes (FCS)           Yes (FCS)
Multilink support     No                  Yes (PPP Multilink)
Dynamic IP assignment No                  Yes (IPCP)
Configuration         Minimal (default)   More options to configure
Overhead              Lower               Slightly higher
Use when              Both ends Cisco     Mixed vendor, need features
```

---

## 5. PPPoE — PPP over Ethernet

### Why PPPoE Exists

DSL (Digital Subscriber Line) broadband uses standard telephone copper wire to deliver internet access. The physical infrastructure uses Ethernet from the customer's router to the DSLAM (DSL Access Multiplexer) at the CO, but ISPs needed a way to authenticate subscribers and manage their sessions — something Ethernet alone cannot do.

The solution was to run PPP *inside* Ethernet frames. This gave ISPs the authentication, session management, and per-session IP addressing they needed, while using the ubiquitous Ethernet infrastructure.

```
Without PPPoE: Ethernet → DSL modem → ISP
  No authentication, no session management
  ISP can't identify or control individual subscribers

With PPPoE:
  [Router] ──PPPoE over Ethernet──► [DSLAM at CO] ──► [ISP Network]
  PPP inside Ethernet: username/password auth, session tracking, IP assignment
  ISP knows exactly which subscriber is which, can enforce quotas, billing
```

### How PPPoE Works

PPPoE operates in two stages. First, a **discovery stage** where the client broadcasts to find available access concentrators (the ISP's PPPoE server). Second, a **session stage** where a PPP session is established within the discovered session, exactly like a regular PPP connection over a serial link.

```
PPPoE Discovery Stage (find server):
  Client → PADI (PPPoE Active Discovery Initiation) → Broadcast
  Server → PADO (PPPoE Active Discovery Offer) → Unicast back
  Client → PADR (PPPoE Active Discovery Request) → Unicast to server
  Server → PADS (PPPoE Active Discovery Session-confirmation) → session ID assigned

PPPoE Session Stage (establish PPP):
  LCP negotiation (link parameters)
  CHAP authentication (username/password)
  IPCP (IP address assigned from ISP)
  Data flows
```

```cisco
! PPPoE Client Configuration on Cisco Router (home/branch DSL)

! Step 1: Create the virtual dialer interface (this is the PPP session)
Router(config)# interface Dialer1
Router(config-if)# ip address negotiated          ! ISP assigns IP via IPCP
Router(config-if)# ip mtu 1492                    ! 1500 - 8 bytes PPPoE overhead
Router(config-if)# encapsulation ppp
Router(config-if)# ppp authentication chap callin
Router(config-if)# ppp chap hostname subscriber@isp.com
Router(config-if)# ppp chap password ISPpassword
Router(config-if)# dialer pool 1                  ! link to physical interface pool
Router(config-if)# dialer-group 1

! Step 2: Configure physical Ethernet interface (connected to DSL modem)
Router(config)# interface GigabitEthernet0/0
Router(config-if)# no ip address                  ! no IP on physical interface
Router(config-if)# pppoe enable                   ! enable PPPoE on this interface
Router(config-if)# pppoe-client dial-pool-number 1   ! link to dialer pool 1
Router(config-if)# no shutdown

! Step 3: Default route via dialer interface (internet traffic goes through PPPoE)
Router(config)# ip route 0.0.0.0 0.0.0.0 Dialer1

! Verify PPPoE session
Router# show pppoe session
Router# show interfaces Dialer1
Router# show ip interface brief   ! Dialer1 should have IP from ISP
```

---

## 6. MPLS — Multiprotocol Label Switching

### Understanding MPLS: Why It Was Invented

Traditional IP routing requires each router to examine the destination IP address of every packet, perform a lookup in its routing table (which can have millions of entries), find the longest prefix match, and then forward the packet. This process, repeated at every hop across the network, was a significant performance bottleneck when internet traffic volumes exploded in the late 1990s.

MPLS was designed to solve this by introducing **label-based switching**. Instead of examining the IP destination address at each hop, MPLS routers look at a short, fixed-length **label** attached to the packet. A label lookup is dramatically faster than an IP routing table lookup because it is a simple exact match (not longest prefix), and the result is stored in a compact switching table.

### How MPLS Labels Work

Think of MPLS like a highway toll system. When you enter the highway (at the edge of the MPLS network), you are assigned a ticket (label) that tells every toll booth (router) exactly which lane to send you to. You don't need to show your driver's license (destination IP) at every booth — just your ticket. At the exit (far edge of the MPLS network), your ticket is taken (label removed) and you return to regular road navigation (IP routing).

```
Without MPLS (IP routing at each hop):
Packet: Dst = 192.168.5.10
  R1: Look up 192.168.5.10 in 800,000-entry table → forward via R2
  R2: Look up 192.168.5.10 in 800,000-entry table → forward via R3
  R3: Look up 192.168.5.10 in 800,000-entry table → deliver

With MPLS (label switching):
Packet enters MPLS network at edge:
  Ingress LER: Look up 192.168.5.10 → assign Label 100, forward to LSR1
  LSR1: Label 100 → swap to Label 200, forward to LSR2   (simple lookup!)
  LSR2: Label 200 → swap to Label 300, forward to Egress LER
  Egress LER: Remove label → deliver packet normally
```

### MPLS Label Structure

The MPLS label is a 32-bit field inserted between the Layer 2 header (Ethernet, PPP, etc.) and the IP packet. This is why MPLS is sometimes called a "Layer 2.5" protocol — it sits between L2 and L3.

```
MPLS Label Stack Entry (32 bits total):
┌─────────────────────┬─────┬───┬─────────┐
│    Label            │ EXP │ S │   TTL   │
│    (20 bits)        │(3 b)│(1)│ (8 bits)│
└─────────────────────┴─────┴───┴─────────┘

Label (20 bits): The actual label value (0–1,048,575)
EXP   (3 bits):  Experimental — used for QoS/Class of Service (CoS)
S     (1 bit):   Bottom of Stack — 1 means this is the last label
TTL   (8 bits):  Time To Live — decremented at each LSR

Labels can be stacked (multiple labels on one packet):
  Outer label → routed through SP backbone
  Inner label → identifies the customer VPN
  This is how MPLS L3VPN works!
```

### MPLS Network Roles

```
LER (Label Edge Router):
  Located at the EDGE of the MPLS network
  Ingress LER: receives unlabeled IP packets, looks up FEC,
               imposes (pushes) the label, forwards into MPLS core
  Egress LER:  receives labeled packets from core,
               removes (pops) the label, forwards as normal IP packet
  Also called PE (Provider Edge) router

LSR (Label Switch Router):
  Located in the CORE of the MPLS network
  Receives labeled packets
  Swaps the incoming label for an outgoing label
  Forwards based on the label — does NOT look at IP header
  Also called P (Provider) router

CE (Customer Edge) router:
  The customer's router that connects to the PE
  Not part of the MPLS network
  Runs regular routing protocols (OSPF, EIGRP, BGP) with the PE
  Completely unaware of MPLS labels
```

### MPLS L3VPN — Connecting Multiple Sites

The most common enterprise use of MPLS is L3VPN service, where the SP uses MPLS to create a private virtual network that connects multiple customer sites. From the customer's perspective, it looks like all their sites are directly connected on the same private network.

```
Customer A's view:
  Branch-A ──────────────────── HQ ──────────────── Branch-B
            (appears directly connected — customer doesn't see SP network)

What actually happens inside SP MPLS network:
  Branch-A's CE ──► PE-1 (assigns label stack: VPN label + transport label)
                           ──────────────────────────────────────►
                                   MPLS Core (P routers swap transport labels)
                           ◄──────────────────────────────────────
                    PE-2 (removes transport label, uses VPN label to find customer's VRF)
                         ──► HQ's CE (delivers IP packet normally)

VRF (Virtual Routing and Forwarding):
  PEs maintain separate routing tables per customer using VRFs
  Customer A's routes in VRF-A, Customer B's routes in VRF-B
  Complete traffic isolation between customers!
```

### Benefits of MPLS for Enterprise WAN

MPLS offers several compelling advantages over traditional leased lines and internet-based WAN solutions. First, it provides **any-to-any connectivity** — every site can reach every other site directly, unlike hub-and-spoke where all traffic must pass through HQ. Second, it supports **traffic engineering** — the SP can explicitly route different classes of traffic over different paths, ensuring voice traffic avoids congested links. Third, it includes **built-in QoS** — the EXP bits in the label allow SP to honor the customer's QoS markings and prioritize voice and video appropriately.

---

## 7. Internet-Based WAN Options

### The Shift Away from Private WAN

For decades, enterprises paid premium prices for private WAN services like leased lines and MPLS because they offered predictable performance, guaranteed bandwidth, and SLA-backed reliability. But as internet bandwidth became cheap, fast, and widely available, organizations began asking: "Why am I paying 10× more for MPLS when I could use the internet?"

The answer depends on requirements. MPLS still wins when you need guaranteed SLAs, strict QoS, and carrier-grade reliability for voice/video. But for many workloads — especially cloud applications — internet connectivity is more than adequate and dramatically cheaper.

### Broadband Internet Access Technologies

```
Technology          Speed Range              Medium           Latency    Notes
──────────────────  ───────────────────────  ───────────────  ─────────  ──────────────────
DSL (ADSL)          Up to 100 Mbps down      Copper phone     10-40ms    Asymmetric (up<down)
                    Up to 20 Mbps up         wire                        Distance-limited
Cable (DOCSIS 3.1)  Up to 1 Gbps down        Coaxial cable    5-30ms     Shared bandwidth
                    Up to 35 Mbps up                                     neighborhood node
Fiber (FTTH)        100 Mbps to 10 Gbps      Fiber optic      1-5ms      Best option
                    Symmetric                                             where available
4G LTE              Up to 150 Mbps           Cellular radio   20-60ms    Mobile, good backup
5G                  Up to 10 Gbps            Cellular radio   1-10ms     Emerging, expensive
Satellite (GEO)     25-100 Mbps              Satellite        500-700ms  High latency!
Satellite (LEO)     50-500 Mbps              Satellite        20-40ms    Starlink etc.
                    (Starlink etc.)
```

### Metro Ethernet

Metro Ethernet delivers Ethernet service across a SP's metropolitan-area network. The customer gets a standard Ethernet handoff (RJ-45 or fiber connector on familiar Ethernet port), which makes it simple to integrate with existing LAN infrastructure — no special WAN knowledge or equipment needed.

The Metro Ethernet Forum (MEF) defines three standard service types. An **E-Line** provides a point-to-point Ethernet connection between exactly two sites, similar to a leased line but using Ethernet. An **E-LAN** provides a multipoint-to-multipoint service where all connected sites share a common Ethernet segment, similar to being on the same VLAN. An **E-Tree** provides a hub-and-spoke structure where the root site can communicate with all leaf sites, but leaves cannot communicate directly with each other.

---

## 8. GRE Tunnels

### What GRE Does and Why It's Useful

**GRE (Generic Routing Encapsulation)** is a tunneling protocol that encapsulates one network layer packet inside another network layer packet, creating a virtual point-to-point link over an existing network. The most common use is creating an IP-in-IP tunnel — encapsulating an IP packet inside another IP packet to carry it across the internet.

The key insight is that GRE creates a **virtual interface** (tunnel interface) on the router. From the router's perspective, this is just another network interface it can route traffic out. Routing protocols, multicast, and any Layer 3 protocol can flow through a GRE tunnel exactly as they would through a physical interface. This is something IPSec VPNs cannot do by themselves — IPSec encrypts traffic but does not support routing protocol updates or multicast over the tunnel.

```
Without GRE tunnel (can't run OSPF over internet):
  HQ Router ──── Internet (IP only, no multicast) ──── Branch Router
  Cannot run OSPF over internet (OSPF uses multicast 224.0.0.5)

With GRE tunnel (virtual point-to-point link over internet):
  HQ Router ════ GRE Tunnel ════ Branch Router
           ↑ virtual direct connection ↑
  Now OSPF works! Multicast works! It's just like a direct Ethernet link.
  
GRE encapsulation adds headers:
  Original: [IP header: 10.0.0.1→10.0.0.2][OSPF Hello packet]
  After GRE: [IP header: 203.0.113.1→198.51.100.1][GRE header][IP header: 10.0.0.1→10.0.0.2][OSPF Hello]
              ↑ outer IP header carries packet across internet ↑
```

### GRE Tunnel Configuration

```cisco
! ════════ HQ ROUTER ════════

! Step 1: Configure the physical WAN interface (internet-facing)
HQ-Router(config)# interface GigabitEthernet0/1
HQ-Router(config-if)# description "Internet WAN Link"
HQ-Router(config-if)# ip address 203.0.113.1 255.255.255.252    ! public IP from ISP
HQ-Router(config-if)# no shutdown

! Step 2: Configure the GRE tunnel interface
HQ-Router(config)# interface Tunnel0
HQ-Router(config-if)# description "GRE Tunnel to Branch"
HQ-Router(config-if)# ip address 172.16.12.1 255.255.255.252    ! tunnel IP (private)
HQ-Router(config-if)# tunnel source GigabitEthernet0/1           ! local WAN interface
HQ-Router(config-if)# tunnel destination 198.51.100.1            ! branch public IP
HQ-Router(config-if)# tunnel mode gre ip                         ! GRE over IPv4 (default)
HQ-Router(config-if)# ip mtu 1476                                ! 1500 - 24 byte GRE overhead
HQ-Router(config-if)# ip tcp adjust-mss 1436                     ! fix TCP MSS for tunnel

! Step 3: Route internal networks over the tunnel
HQ-Router(config)# ip route 192.168.2.0 255.255.255.0 Tunnel0    ! branch LAN via tunnel

! ════════ BRANCH ROUTER ════════

Branch-Router(config)# interface GigabitEthernet0/1
Branch-Router(config-if)# ip address 198.51.100.1 255.255.255.252
Branch-Router(config-if)# no shutdown

Branch-Router(config)# interface Tunnel0
Branch-Router(config-if)# description "GRE Tunnel to HQ"
Branch-Router(config-if)# ip address 172.16.12.2 255.255.255.252
Branch-Router(config-if)# tunnel source GigabitEthernet0/1
Branch-Router(config-if)# tunnel destination 203.0.113.1          ! HQ public IP
Branch-Router(config-if)# tunnel mode gre ip
Branch-Router(config-if)# ip mtu 1476
Branch-Router(config-if)# ip tcp adjust-mss 1436

Branch-Router(config)# ip route 192.168.1.0 255.255.255.0 Tunnel0  ! HQ LAN via tunnel

! Verify the tunnel
Router# show interfaces Tunnel0
Tunnel0 is up, line protocol is up    ← both must be up
  Hardware is Tunnel
  Internet address is 172.16.12.1/30
  Tunnel source 203.0.113.1 (GigabitEthernet0/1), destination 198.51.100.1
  Tunnel protocol/transport GRE/IP
  Tunnel TTL 255, Fast tunneling enabled

Router# ping 172.16.12.2 source Tunnel0    ! ping across the tunnel
Router# ping 192.168.2.1 source 192.168.1.1   ! ping branch LAN from HQ LAN
```

### GRE Limitations

GRE provides encapsulation and tunneling but **no security**. Traffic inside the GRE tunnel is sent in plaintext over the internet — anyone capturing the traffic can read it. For this reason, GRE is almost always combined with IPSec encryption in production deployments, creating a **GRE over IPSec** tunnel that provides both the flexibility of GRE (routing protocols, multicast) and the security of IPSec (encryption, authentication).

---

## 9. IPSec over GRE — Secure Tunnels

When you need both routing protocol support (which requires GRE) and encryption (which requires IPSec), the solution is to combine them. There are two ways to do this:

**GRE over IPSec (Crypto Map approach):** The GRE tunnel is created first, then IPSec encrypts the GRE packets. This is the traditional method.

**IPSec Tunnel with GRE:** More modern approach where IPSec protects the GRE tunnel directly.

```
Traffic flow with GRE over IPSec:
  Original packet: [IP: 10.1.1.10→192.168.2.5][TCP][Data]
  After GRE wrap:  [IP: 203.0.113.1→198.51.100.1][GRE][IP: 10.1.1.10→192.168.2.5][TCP][Data]
  After IPSec:     [IP: 203.0.113.1→198.51.100.1][ESP: ENCRYPTED(GRE+original packet)]
                    ↑ outer IP (routable)          ↑ encrypted — nobody can read this!
```

The key advantage of combining GRE with IPSec is that OSPF or EIGRP can run over the GRE tunnel interface, dynamically learning routes. The IPSec layer ensures that all this traffic — including routing updates — is encrypted as it crosses the internet.

---

## 10. SD-WAN — Software Defined WAN

### The Problem SD-WAN Solves

To understand why SD-WAN was revolutionary, you need to understand the problems with traditional WAN management. Imagine a company with 500 branch offices. Each branch has a router. To push a new QoS policy, a network engineer must SSH into each of the 500 routers individually and type the same commands 500 times. If a configuration error is made on router 347 out of 500, finding and fixing it requires checking each device.

Add to this the complexity of managing multiple WAN transport types — some branches have MPLS, some have internet broadband, some have LTE backup — each requiring different configuration. The traditional approach became unmanageable at scale.

SD-WAN addresses this through three fundamental shifts in how WANs are built and operated.

### Three Fundamental SD-WAN Principles

**Centralized control** means that instead of configuring each router individually, all policies are defined once in a central controller and pushed automatically to every device. A change that previously took a team of engineers weeks now takes minutes.

**Transport independence** means the SD-WAN solution treats all WAN transports (MPLS, broadband internet, LTE) as equal resources and can use any combination simultaneously. The system monitors the real-time health of each path and routes traffic dynamically based on actual conditions.

**Application awareness** means the SD-WAN understands which application is generating each traffic flow and can apply different policies to different applications. Voice calls go over the low-latency MPLS link; file backups go over the cheap internet link; if the MPLS link becomes congested, voice automatically shifts to LTE.

```
Traditional WAN management:
  Engineer SSH → Router-1 → configure
  Engineer SSH → Router-2 → configure
  ...
  Engineer SSH → Router-500 → configure
  Time: weeks. Error risk: very high.

SD-WAN management:
  Engineer → SD-WAN Controller (web GUI or API)
             → define policy once
             → automatically pushed to all 500 routers
  Time: minutes. Consistent. Auditable.
```

### SD-WAN vs Traditional WAN: A Detailed Comparison

```
Feature                 Traditional WAN           SD-WAN
──────────────────────  ────────────────────────  ────────────────────────────
Provisioning time       Weeks (manual)            Hours (zero-touch)
Configuration           Per-device CLI            Central controller, push
Transport types         Usually single (MPLS)     Multiple (MPLS+Internet+LTE)
Path selection          Static routing             Dynamic, application-aware
Failover                Manual or slow protocol   Sub-second automatic
Visibility              Per-device show commands  Central dashboard, real-time
Cloud connectivity      Complex                   Built-in cloud on-ramp
Security                Separate (firewall, VPN)  Integrated (encryption default)
Cost                    High (MPLS-heavy)         Lower (internet + policy)
Scalability             Complex                   Simple (software policy)
```

---

## 11. Cisco SD-WAN Architecture (Viptela)

Cisco's SD-WAN solution — acquired from Viptela in 2017 — has become the industry-leading enterprise SD-WAN platform. Understanding its architecture is important for the CCNA exam.

### The Four Planes of Cisco SD-WAN

Cisco SD-WAN separates the functions of a WAN into four distinct planes, each handled by a specific component. This separation of concerns is what gives SD-WAN its flexibility and scalability.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MANAGEMENT PLANE                                  │
│                                                                      │
│   vManage (GUI Dashboard + REST APIs)                                │
│   • Configure all vEdge devices from one interface                   │
│   • Monitor network health, application performance                  │
│   • Push templates, policies, software updates                       │
│   • REST API for automation integration                              │
└────────────────────────────┬────────────────────────────────────────┘
                             │ NETCONF/HTTPS
┌────────────────────────────▼────────────────────────────────────────┐
│                    CONTROL PLANE                                     │
│                                                                      │
│   vSmart Controller                                                  │
│   • Distributes routing and policy information to vEdge devices      │
│   • Runs OMP (Overlay Management Protocol)                           │
│   • Calculates optimal paths, enforces centralized policies          │
│   • Does NOT forward actual data traffic                             │
│   • Multiple vSmart controllers for redundancy                       │
└────────────────────────────┬────────────────────────────────────────┘
                             │ OMP (Overlay Management Protocol)
┌────────────────────────────▼────────────────────────────────────────┐
│                    ORCHESTRATION PLANE                               │
│                                                                      │
│   vBond Orchestrator                                                 │
│   • First point of contact for new vEdge devices                    │
│   • Authenticates vEdge devices (using certificates)                 │
│   • Helps vEdge devices discover vManage and vSmart                  │
│   • Must have a public IP (directly internet-reachable)              │
└──────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                       DATA PLANE                                     │
│                                                                      │
│   vEdge Routers (at each branch/site)                                │
│   • Forward actual user data traffic                                 │
│   • Build encrypted IPSec tunnels between sites                      │
│   • Monitor transport link quality (latency, jitter, loss)           │
│   • Apply application-aware routing policies from vSmart             │
│   • Can be physical hardware or virtual (vEdge Cloud)                │
└─────────────────────────────────────────────────────────────────────┘
```

### OMP — Overlay Management Protocol

**OMP (Overlay Management Protocol)** is the SD-WAN control plane protocol — think of it as BGP for the SD-WAN overlay network. OMP runs between vEdge routers and vSmart controllers (not between vEdge routers directly). vSmart acts as a route reflector: it receives route advertisements from all vEdge devices and distributes them to all other vEdge devices, along with policy information.

OMP carries three types of information. **OMP Routes** are network prefixes — equivalent to BGP routes, they tell vEdge devices which networks are reachable at each site. **TLOC Routes** (Transport Locator) identify the physical WAN transports available at each site (MPLS endpoint, internet endpoint, LTE endpoint), allowing vEdge devices to build tunnels to each other. **Service Routes** advertise network services (like a firewall or WAAS appliance) available at a site so that traffic can be directed through them.

### Zero-Touch Provisioning (ZTP)

One of SD-WAN's most operationally valuable features is Zero-Touch Provisioning. When a new branch is opened, instead of having a network engineer travel to the site or manually configure the router, the process is fully automated.

```
ZTP Process:
  1. New vEdge arrives at branch — someone plugs it in (power + WAN cable)
  2. vEdge boots up, has no configuration
  3. vEdge contacts Cisco's cloud ZTP server using DHCP/DNS
  4. ZTP server redirects vEdge to the organization's vBond
  5. vBond authenticates vEdge (using pre-loaded certificates)
  6. vBond introduces vEdge to vSmart and vManage
  7. vManage pushes the correct configuration template to vEdge
  8. vEdge builds IPSec tunnels to other sites
  Total time: 15-30 minutes, with zero manual CLI configuration
```

### Application-Aware Routing

This is where SD-WAN delivers its most compelling value. Traditional WAN routing uses protocol metrics (OSPF cost, BGP attributes) to choose paths. SD-WAN uses **real-time application performance** measured on every available transport link.

```
Branch vEdge monitors all available links every second:
  MPLS link:     latency=8ms, jitter=1ms, packet_loss=0%    → excellent
  Internet link: latency=25ms, jitter=8ms, packet_loss=0.5% → acceptable
  LTE link:      latency=45ms, jitter=15ms, packet_loss=1%  → usable

Policy defined in vManage:
  "VoIP traffic (DSCP EF, ports 5060, RTP):"
  "  → Primary: MPLS if latency <20ms AND loss <1%"
  "  → Failover: LTE if MPLS fails or latency >20ms"
  
  "Web browsing (HTTPS):"
  "  → Use Internet link (save MPLS for critical apps)"
  
  "Backup/bulk data transfers:"
  "  → Use whichever link has spare capacity"

If MPLS link degrades (latency spikes to 80ms):
  vEdge detects degradation within seconds
  VoIP flows automatically moved to LTE (45ms < 80ms)
  Sub-second failover — users barely notice
```

### Cisco SD-WAN Verification

```cisco
! On vEdge device (Cisco IOS-XE with SD-WAN software):

! Check control connections (to vSmart and vManage)
Router# show sdwan control connections
                                          PEER                                          PEER                                          CONTROLLER
PEER    PEER PEER            SITE       DOMAIN PEER                                     PRIV  PEER                                     PUB                                     GROUP      PEER ID
TYPE    PROT SYSTEM IP       ID         ID     PRIVATE IP                               PORT  PUBLIC IP                                PORT         UPTIME                          
vbond   dtls 0.0.0.0        0          0      100.64.0.2                               12346 100.64.0.2                               12346        0:00:05:28
vsmart  dtls 1.1.1.2        100        1      10.0.0.2                                 12346 10.0.0.2                                 12346        0:00:05:15

! Check BFD sessions (tunnel health monitoring)
Router# show sdwan bfd sessions
                                      SOURCE TLOC      REMOTE TLOC
                                      COLOR            COLOR
SYSTEM IP    SITE ID  STATE  COLOR    RESTRICT  COLOR  RESTRICT         ENCAP  TX PKTS  RX PKTS
10.0.0.2     200      up     mpls     No        mpls   No               ipsec  45       45
10.0.0.2     200      up     biz-internet No    biz-internet No         ipsec  30       30

! Check OMP routes
Router# show sdwan omp routes

! Check application-aware routing policy
Router# show sdwan policy from-vsmart
```

---

## 12. WAN Troubleshooting

### Systematic WAN Troubleshooting Approach

WAN troubleshooting requires a methodical approach because there are multiple layers where failures can occur — physical, data link encapsulation, IP addressing, and routing. Always work from the bottom up (physical first) before assuming a higher-layer issue.

```
Layer 1 — Physical:
  Router# show interfaces Serial0/0/0
    Is the interface up/up?
    Check line status and protocol status
    Look for: input errors, CRC, carrier transitions
  
  If "Serial0/0/0 is down":
    Check physical cable connection
    Check CSU/DSU (if applicable)
    Verify SP has the circuit provisioned and active
    Check: Router# show controllers Serial0/0/0
           Look for "DCE" or "DTE" and clock rate

Layer 2 — Encapsulation:
  Router# show interfaces Serial0/0/0
    Check: "Encapsulation HDLC" or "Encapsulation PPP"
  Both ends MUST have same encapsulation!
  HDLC on one end, PPP on other = "line protocol is down"
  
  For PPP:
  Router# show ppp all             ! PPP negotiation state
  Router# show interfaces Serial0/0/0
    Look for: "LCP Open" (good), "LCP Closed" (problem)

Layer 3 — IP Addressing:
  Router# show ip interface brief  ! check IP assigned
  Router# show running-config interface Serial0/0/0  ! verify IP config
  Ping the directly connected IP (neighbor's address)
  If ping fails and physical/L2 is up → IP addressing issue

Layer 4+ — Routing:
  Router# show ip route            ! expected routes present?
  Router# ping [far-end-network]   ! end-to-end reachability
  Router# traceroute [destination] ! where does it stop?
```

### Common WAN Problems and Solutions

```
PROBLEM: Serial interface shows "down/down"
  Cause 1: No cable connected or faulty cable
  Cause 2: SP has not activated the circuit
  Cause 3: Remote end is administratively down
  Cause 4: Clock rate not configured (in lab with back-to-back cable)
  Fix:
    show controllers serial 0/0/0    ← shows DCE/DTE and clock
    On DCE end: ip address + clock rate + no shutdown
    Call SP to verify circuit is active

PROBLEM: Serial interface shows "up/down"
  Cause 1: Encapsulation mismatch (HDLC on one side, PPP on other)
  Cause 2: PPP authentication failure (wrong username/password)
  Cause 3: MTU mismatch
  Fix:
    show interfaces serial 0/0/0    ← check encapsulation
    debug ppp authentication        ← watch authentication attempt
    Ensure both sides same encapsulation and auth credentials

PROBLEM: PPP interface up/up but can't ping neighbor
  Cause 1: IP addresses on wrong subnet
  Cause 2: ACL blocking ICMP
  Fix:
    show ip interface serial 0/0/0  ← verify IP addresses
    show ip access-lists            ← check for deny entries
    ping [neighbor IP] from directly connected interface

PROBLEM: GRE tunnel shows "up/down"
  Cause: Tunnel destination IP is not reachable
         (the underlying internet path is broken)
  Fix:
    ping [tunnel destination public IP]  ← can you reach the far end?
    show ip route [tunnel dest]          ← is there a route to it?
    If no: fix underlying internet connectivity first

PROBLEM: GRE tunnel up but routing doesn't work
  Cause 1: Routes not configured to use tunnel
  Cause 2: Routing protocol not configured on tunnel interface
  Cause 3: MTU issue (GRE adds 24 bytes overhead)
  Fix:
    show ip route                    ← verify tunnel-based routes
    show interfaces tunnel 0         ← check MTU (should be 1476)
    ip tcp adjust-mss 1436          ← fix TCP MSS on tunnel interface
```

---

## 13. Key Takeaways & Exam Tips

### Key Takeaways

A WAN connects geographically separate sites using service provider infrastructure. The SP owns the WAN backbone; you own the CPE at each end. The local loop (last mile) is the physical connection between your site and the SP's CO and is the most common failure point.

**HDLC** is Cisco's default serial encapsulation but is proprietary — it only works between Cisco routers. **PPP** is the open standard alternative that works with any vendor and adds features like PAP/CHAP authentication, compression, and multilink. **CHAP** is significantly more secure than PAP because the password is never sent across the link — only an MD5 hash is exchanged. **PPPoE** wraps PPP inside Ethernet frames to allow ISPs to authenticate DSL subscribers.

**MPLS** uses 32-bit labels for fast packet switching inside the SP core, avoiding expensive IP routing table lookups at every hop. Ingress LERs (PE routers) impose labels; core LSRs (P routers) swap labels; egress LERs remove labels. MPLS L3VPN uses VRFs to provide customer traffic isolation.

**GRE** creates virtual point-to-point tunnels over IP networks, encapsulating any Layer 3 protocol. The key advantage of GRE over IPSec alone is that GRE supports routing protocols and multicast over the tunnel. However, GRE provides no encryption — combine it with IPSec for secure tunnels.

**SD-WAN** replaces per-device WAN management with centralized, application-aware, policy-driven orchestration. Cisco SD-WAN uses four components: **vManage** (management plane GUI/API), **vSmart** (control plane, runs OMP), **vBond** (orchestration/authentication), and **vEdge** (data plane, at each branch). **OMP** is the SD-WAN control plane protocol that distributes routes and policies.

### Exam Tips

> 🎯 **Most Tested Topics:**

1. **"What is the default serial encapsulation on Cisco routers?"** → **HDLC** — but it's proprietary (Cisco-only)
2. **"HDLC or PPP — which works with non-Cisco equipment?"** → **PPP** (open standard, any vendor)
3. **"Which PPP authentication method sends the password in plain text?"** → **PAP** — never use in production
4. **"Which PPP authentication is more secure and why?"** → **CHAP** — uses MD5 hash, password never transmitted
5. **"What does the 'S' bit in the MPLS label mean?"** → **Bottom of Stack** — set to 1 on the last (innermost) label
6. **"What is the difference between PE and P routers in MPLS?"** → **PE** (Provider Edge) is at the edge — adds/removes labels. **P** (Provider) is in the core — only swaps labels, never sees IP headers
7. **"What does GRE add that IPSec alone cannot provide?"** → **Routing protocol support and multicast** — GRE creates a virtual interface that routing protocols treat as a regular link
8. **"What port does GRE use?"** → GRE is **IP Protocol 47** (not TCP or UDP — it runs directly over IP)
9. **"What SD-WAN component provides the management GUI?"** → **vManage**
10. **"What SD-WAN component handles the control plane and distributes routes?"** → **vSmart** (uses OMP protocol)
11. **"What is Zero-Touch Provisioning in SD-WAN?"** → A new device automatically downloads its configuration from the controller upon first boot — no manual CLI needed
12. **"What is the overhead added by GRE encapsulation?"** → **24 bytes** — so MTU on tunnel interface should be set to 1476 (1500 - 24)
13. **"What does PPPoE use the Dialer interface for?"** → The Dialer interface is the virtual PPP session endpoint — it gets the IP address from the ISP and carries all traffic

---

✅ **Section 1–13: All sections complete.**

*← Previous: [08 — ACLs & Security](./08_ACLs_Security.md)*  
*Next: [10 — IP Services →](./10_IP_Services.md)*
