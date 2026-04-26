# 06 — Switching & VLANs: Complete Reference

> **CCNA Exam Domain:** 2.0 Network Access  
> **Exam Weight:** ~20% of total exam  
> **Difficulty:** ⭐⭐⭐☆☆

---

## Table of Contents

1. [Ethernet & LAN Fundamentals](#1-ethernet--lan-fundamentals)
2. [How Ethernet Switches Work](#2-how-ethernet-switches-work)
3. [MAC Address Table (CAM Table)](#3-mac-address-table-cam-table)
4. [Switch Port Modes, Duplex & Speed](#4-switch-port-modes-duplex--speed)
5. [VLANs — Virtual Local Area Networks](#5-vlans--virtual-local-area-networks)
6. [VLAN Configuration (Cisco IOS)](#6-vlan-configuration-cisco-ios)
7. [VLAN Trunking — IEEE 802.1Q](#7-vlan-trunking--ieee-8021q)
8. [DTP — Dynamic Trunking Protocol](#8-dtp--dynamic-trunking-protocol)
9. [Inter-VLAN Routing](#9-inter-vlan-routing)
10. [VTP — VLAN Trunking Protocol](#10-vtp--vlan-trunking-protocol)
11. [STP — Spanning Tree Protocol (802.1D)](#11-stp--spanning-tree-protocol-8021d)
12. [RSTP — Rapid Spanning Tree (802.1w)](#12-rstp--rapid-spanning-tree-8021w)
13. [STP Enhancements](#13-stp-enhancements--portfast-bpdu-guard-root-guard)
14. [EtherChannel / Link Aggregation](#14-etherchannel--link-aggregation)
15. [Port Security](#15-port-security)
16. [Switch Hardening Best Practices](#16-switch-hardening-best-practices)
17. [Key Takeaways & Exam Tips](#17-key-takeaways--exam-tips)

---

## 1. Ethernet & LAN Fundamentals

### Ethernet Frame Structure

```
┌──────────┬──────┬──────────┬──────────┬───────────┬────────────┬──────┬─────┐
│ Preamble │ SFD  │ Dst MAC  │  Src MAC │EtherType/ │    Data    │      │ FCS │
│ 7 bytes  │ 1 B  │ 6 bytes  │  6 bytes │  Length   │  46-1500B  │      │ 4 B │
│10101010..│1011  │          │          │  2 bytes  │            │      │(CRC)│
└──────────┴──────┴──────────┴──────────┴───────────┴────────────┴──────┴─────┘
  Min frame size: 64 bytes (below = "runt" — collision fragment)
  Max frame size: 1518 bytes (standard), 9018 bytes (jumbo frames)
```

| Field | Size | Purpose |
|-------|------|---------|
| Preamble | 7 bytes | 10101010... pattern — clock synchronization |
| SFD (Start Frame Delimiter) | 1 byte | 10101011 — marks frame start |
| Destination MAC | 6 bytes | Recipient hardware address |
| Source MAC | 6 bytes | Sender hardware address |
| EtherType/Length | 2 bytes | Protocol (≥1536) or payload length |
| Data/Payload | 46–1500 bytes | Packet from Network layer |
| FCS | 4 bytes | CRC-32 error detection |

### MAC Address Deep Dive

```
Format:  AA:BB:CC:DD:EE:FF  (colon notation — Linux/IEEE)
         AABB.CCDD.EEFF      (Cisco dot notation)
         AA-BB-CC-DD-EE-FF   (Windows dash notation)

┌──────────────────────┬─────────────────────────────┐
│  OUI (3 bytes / 24b) │  Device Identifier (3 bytes)│
│  Assigned by IEEE to │  Unique per device from      │
│  each manufacturer   │  that manufacturer           │
└──────────────────────┴─────────────────────────────┘

Bit meanings in FIRST byte:
  Bit 0 (LSB): 0 = Unicast, 1 = Multicast/Broadcast
  Bit 1:       0 = Globally unique (OUI-assigned)
               1 = Locally administered (manually set)

Special addresses:
  FF:FF:FF:FF:FF:FF → Broadcast (all devices)
  01:00:5E:xx:xx:xx → IPv4 multicast
  33:33:xx:xx:xx:xx → IPv6 multicast
  00:00:00:00:00:00 → Unspecified (invalid for use)

Example OUIs:
  00:00:0C → Cisco
  00:50:56 → VMware
  08:00:27 → Oracle VirtualBox
  DC:A6:32 → Raspberry Pi
```

---

## 2. How Ethernet Switches Work

### Switch vs Hub vs Router

```
┌─────────────────────────────────────────────────────────────────┐
│  HUB (Layer 1)                                                  │
│  • Receives bit stream on one port → repeats out ALL ports      │
│  • No MAC learning — zero intelligence                          │
│  • ONE collision domain for entire hub                          │
│  • ONE broadcast domain                                         │
│  • Half-duplex only (CSMA/CD required)                          │
│  • Obsolete — not used in modern networks                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  SWITCH (Layer 2)                                               │
│  • Forwards frames based on DESTINATION MAC address             │
│  • Each port = its own collision domain (full-duplex capable)   │
│  • Without VLANs: ONE broadcast domain for entire switch        │
│  • Builds and maintains MAC address table (CAM table)           │
│  • Supports wire-speed forwarding via ASIC hardware             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  ROUTER (Layer 3)                                               │
│  • Forwards packets based on DESTINATION IP address             │
│  • Each interface = separate collision domain                   │
│  • Each interface = separate broadcast domain                   │
│  • Connects different networks/subnets                          │
│  • Uses routing table for path decisions                        │
└─────────────────────────────────────────────────────────────────┘
```

### Complete Switch Forwarding Logic

```
Frame arrives on Port Gi0/1 (Source MAC: AAAA, Dst MAC: BBBB)
                    │
          ┌─────────▼────────────────────────────────────────────┐
          │ STEP 1 — LEARN (Source MAC)                          │
          │ Record: AAAA → Gi0/1 → VLAN 10 in CAM table         │
          │ Reset aging timer if entry already existed            │
          └─────────┬────────────────────────────────────────────┘
                    │
          ┌─────────▼────────────────────────────────────────────┐
          │ STEP 2 — FORWARDING DECISION (Destination MAC)       │
          │                                                       │
          │  Is Dst = FF:FF:FF:FF:FF:FF (Broadcast)?             │
          │  YES → FLOOD all ports except source                 │
          │                                                       │
          │  Is Dst a multicast address?                         │
          │  YES → FLOOD (unless IGMP snooping configured)       │
          │                                                       │
          │  Look up Dst MAC in CAM table...                     │
          │                                                       │
          │  FOUND, same port as source?                         │
          │  YES → FILTER (drop — both devices on same segment)  │
          │                                                       │
          │  FOUND, different port?                              │
          │  YES → FORWARD out that specific port only           │
          │                                                       │
          │  NOT FOUND (unknown unicast)?                        │
          │  YES → FLOOD all ports except source                 │
          └──────────────────────────────────────────────────────┘
```

### Collision Domains vs Broadcast Domains

```
Example topology:

[PC1]─┐                    ┌─[PC5]
[PC2]─┤──[HUB]──┬──[SW1]──┤
[PC3]─┘         │          └─[PC6]──[ROUTER]──[PC7]
                 │                      │
              [PC4]                   [PC8]

Collision Domains (each switch port + hub = one):
  HUB segment (PC1+PC2+PC3+SW1 port): 1 collision domain
  SW1-PC4: 1 collision domain
  SW1-PC5: 1 collision domain
  SW1-PC6: 1 collision domain
  Router-PC7: 1 collision domain
  Router-PC8: 1 collision domain
  Total: 6 collision domains

Broadcast Domains:
  Left side of router (HUB+SW1+PC1-6): 1 broadcast domain
  Right side of router (PC7+PC8): 1 broadcast domain
  Total: 2 broadcast domains

Key rule:
  Switches separate COLLISION domains
  Routers separate BROADCAST domains
  VLANs also create separate BROADCAST domains
```

---

## 3. MAC Address Table (CAM Table)

### CAM Table Population Step-by-Step

```
t=0: Empty switch, PC1 sends ARP to find PC2

         CAM TABLE (empty)
         ┌──────────────┬──────┬──────┐
         │  MAC Address │ VLAN │ Port │
         └──────────────┴──────┴──────┘

t=1: ARP from PC1 (MAC=AAAA) arrives on Gi0/1
     Switch LEARNS: AAAA → VLAN10 → Gi0/1

         CAM TABLE
         ┌──────────────┬──────┬──────┐
         │     AAAA     │  10  │ Gi0/1│  ← learned
         └──────────────┴──────┴──────┘

     BBBB (PC2) not in table → FLOOD to Gi0/2, Gi0/3, Gi0/4

t=2: PC2 (MAC=BBBB, port Gi0/2) replies to PC1
     Switch LEARNS: BBBB → VLAN10 → Gi0/2

         CAM TABLE
         ┌──────────────┬──────┬──────┐
         │     AAAA     │  10  │ Gi0/1│
         │     BBBB     │  10  │ Gi0/2│  ← learned
         └──────────────┴──────┴──────┘

t=3: Future PC1→PC2 frames forwarded ONLY to Gi0/2 ✅
     No more flooding! Switch is now efficient.
```

### CAM Table Management

```cisco
! View entire MAC address table
Switch# show mac address-table
Switch# show mac address-table dynamic           ! only dynamic entries
Switch# show mac address-table static            ! only static entries
Switch# show mac address-table count             ! total counts
Switch# show mac address-table vlan 10           ! entries for VLAN 10
Switch# show mac address-table address 0050.7966.6800  ! specific MAC
Switch# show mac address-table interface Gi0/1   ! entries on specific port

! Modify aging time (default = 300 seconds)
Switch(config)# mac address-table aging-time 600  ! 10 minutes

! Add permanent static entry (never ages out)
Switch(config)# mac address-table static 0050.7966.6800 vlan 10 interface Gi0/3

! Clear dynamic entries
Switch# clear mac address-table dynamic
Switch# clear mac address-table dynamic interface Gi0/1
Switch# clear mac address-table dynamic vlan 10
```

### MAC Flooding Attack — How It Works

```
ATTACK SCENARIO:
  Attacker runs tool (e.g., macof) that generates 1000s of random source MACs
  Switch CAM table fills up (typically 8000-16000 entries)
  New legitimate MACs can no longer be learned

RESULT:
  Switch can't find unknown destination MACs in full table
  Switch FLOODS ALL TRAFFIC to all ports
  Attacker receives all network traffic on their port
  Attacker captures sensitive data (passwords, emails, etc.)

MITIGATION: Port Security (Section 15)
  Limit MACs per port → attacker fills their one port's limit
  Switch err-disables the attacker's port
```

---

## 4. Switch Port Modes, Duplex & Speed

### Interface Status Reference

```
Switch# show interfaces GigabitEthernet0/1
GigabitEthernet0/1 is [line status], line protocol is [protocol status]

Status combinations:
┌──────────────────────┬──────────────────┬──────────────────────────────────┐
│ Line Status          │ Protocol Status  │ Meaning & Likely Cause           │
├──────────────────────┼──────────────────┼──────────────────────────────────┤
│ up                   │ up               │ ✅ Fully operational              │
├──────────────────────┼──────────────────┼──────────────────────────────────┤
│ up                   │ down             │ L2 issue: keepalive failure,     │
│                      │                  │ encapsulation mismatch            │
├──────────────────────┼──────────────────┼──────────────────────────────────┤
│ down                 │ down             │ Physical problem: no cable,      │
│                      │                  │ bad cable, wrong cable type,     │
│                      │                  │ speed mismatch, far-end down     │
├──────────────────────┼──────────────────┼──────────────────────────────────┤
│ administratively     │ down             │ Manual shutdown: `shutdown`      │
│ down                 │                  │ command was issued               │
├──────────────────────┼──────────────────┼──────────────────────────────────┤
│ err-disabled         │ err-disabled     │ Disabled by switch feature:      │
│                      │                  │ port security, BPDU Guard,       │
│                      │                  │ UDLD, EtherChannel mismatch      │
└──────────────────────┴──────────────────┴──────────────────────────────────┘
```

### Duplex Mismatch — Most Common Misconfiguration

```
Symptoms of duplex mismatch:
  ✗ Late collisions in interface counters (definitive sign)
  ✗ FCS errors, CRC errors
  ✗ Extremely poor throughput (may only achieve 10-30% of expected)
  ✗ Runts (frames < 64 bytes — collision fragments)
  ✗ Inconsistent/intermittent connectivity

Why it happens:
  One side configured full-duplex, other side half-duplex
  Often from manually setting one side and auto on other
  Auto-negotiation failure on old equipment

Fix: Explicitly set BOTH sides to the same duplex and speed
```

```cisco
! Configure interface duplex and speed manually (recommended for servers/uplinks)
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)# duplex full
Switch(config-if)# speed 1000
Switch(config-if)# description "Server-01 - Database"
Switch(config-if)# no shutdown

! Check interface statistics (look for error counters)
Switch# show interfaces GigabitEthernet0/1
  5 minute input rate 0 bits/sec, 0 packets/sec
  5 minute output rate 0 bits/sec, 0 packets/sec
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
     0 input packets with dribble condition detected
     0 output errors, 0 collisions, 0 interface resets
     0 late collision, 0 deferred ← late collision = duplex mismatch!

Switch# show interfaces GigabitEthernet0/1 counters errors
Switch# show interfaces status
```

### Interface Error Counter Reference

| Counter | What It Means | Common Cause |
|---------|--------------|-------------|
| **CRC** | Bad checksum | Bad cable, duplex mismatch, interference |
| **Frame** | Illegal framing | Duplex mismatch |
| **Runts** | Frame < 64 bytes | Collision fragments, duplex mismatch |
| **Giants** | Frame > max size | MTU mismatch, jumbo frames misconfigured |
| **Late collisions** | Collision after 512 bits | ⚠️ Definitive duplex mismatch sign |
| **Collisions** | CSMA/CD collisions | Normal in half-duplex only |
| **Input errors** | Total L1/L2 receive errors | Various |
| **Output drops** | Queue full, frames dropped | Congestion, QoS issues |

---

## 5. VLANs — Virtual Local Area Networks

### The Problem VLANs Solve

```
WITHOUT VLANs — One large broadcast domain:

[Sales-PC] [Sales-PC] [HR-PC] [Finance-PC] [IoT-Camera] [Guest]
     │           │        │         │             │          │
     └─────────────────── Switch ────────────────────────────┘
                    ONE Broadcast Domain

Problems:
  • Every ARP, DHCP, and other broadcast hits ALL 6 devices
  • Finance can potentially see HR traffic (no isolation)
  • Guest user shares same segment as internal servers
  • IoT camera on same network as sensitive finance data
  • Broadcast storm from one device takes down everyone

WITH VLANs — Multiple logical broadcast domains:

     VLAN 10 (Sales):    [Sales-PC] [Sales-PC]
     VLAN 20 (HR):       [HR-PC]
     VLAN 30 (Finance):  [Finance-PC]
     VLAN 40 (IoT):      [IoT-Camera]
     VLAN 99 (Guest):    [Guest]
                   ↑─── ALL on ONE physical switch ───┘

Benefits:
  • 5 separate broadcast domains — broadcasts stay in their VLAN
  • Finance/HR completely isolated from each other
  • Guest can only reach internet (via firewall policy)
  • IoT on separate VLAN — breach contained
  • Logical grouping regardless of physical location
```

### VLAN ID Ranges

```
┌─────────────────┬─────────────────────────────────────────────────┐
│ Range           │ Description                                      │
├─────────────────┼─────────────────────────────────────────────────┤
│ VLAN 1          │ Default VLAN — ALL ports start here              │
│                 │ Cannot be deleted or renamed                     │
│                 │ AVOID using for user traffic (security)          │
├─────────────────┼─────────────────────────────────────────────────┤
│ VLAN 2–1001     │ Normal range — user-created VLANs                │
│                 │ Stored in flash:/vlan.dat                        │
│                 │ Propagated by VTP                                │
├─────────────────┼─────────────────────────────────────────────────┤
│ VLAN 1002–1005  │ Reserved for legacy (FDDI/Token Ring)            │
│                 │ Cannot be used for Ethernet or deleted           │
├─────────────────┼─────────────────────────────────────────────────┤
│ VLAN 1006–4094  │ Extended range                                   │
│                 │ Requires VTP transparent/off mode                │
│                 │ Stored in running-config (not vlan.dat)          │
└─────────────────┴─────────────────────────────────────────────────┘
```

### Special VLAN Roles

| VLAN Role | Recommended ID | Purpose |
|-----------|---------------|---------|
| **Default VLAN** | 1 (fixed) | Unassigned ports default here — don't use for traffic |
| **Data VLAN** | 10, 20, 30... | User device traffic (PCs, servers) |
| **Management VLAN** | 99 (example) | Switch SSH/Telnet access — assign SVI IP here |
| **Native VLAN** | 999 (recommended) | Untagged trunk traffic — change from VLAN 1 |
| **Voice VLAN** | 150 (example) | IP phone traffic — tagged for QoS |
| **Black Hole VLAN** | 999 (example) | Unused ports — shut them down AND move to this VLAN |

---

## 6. VLAN Configuration (Cisco IOS)

### Creating VLANs

```cisco
! Create VLANs with names (best practice — always name VLANs)
Switch(config)# vlan 10
Switch(config-vlan)# name Sales_Department
Switch(config)# vlan 20
Switch(config-vlan)# name Human_Resources
Switch(config)# vlan 30
Switch(config-vlan)# name Finance
Switch(config)# vlan 40
Switch(config-vlan)# name IT_Infrastructure
Switch(config)# vlan 99
Switch(config-vlan)# name Management_VLAN
Switch(config)# vlan 150
Switch(config-vlan)# name Voice_VoIP
Switch(config)# vlan 999
Switch(config-vlan)# name BlackHole_DISABLED

! Verify VLAN database
Switch# show vlan brief
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------
1    default                          active    Gi0/9, Gi0/10
10   Sales_Department                 active    Gi0/1, Gi0/2
20   Human_Resources                  active    Gi0/3
30   Finance                          active    Gi0/4
40   IT_Infrastructure                active
99   Management_VLAN                  active
150  Voice_VoIP                       active
999  BlackHole_DISABLED               active    Gi0/5-Gi0/8

! Delete a VLAN
Switch(config)# no vlan 30
! WARNING: Ports still in VLAN 30 become inactive until reassigned!
```

### Assigning Access Ports to VLANs

```cisco
! Single port — end device connection
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)# description "Sales-PC-01 - John Smith"
Switch(config-if)# switchport mode access       ! must set access mode explicitly
Switch(config-if)# switchport access vlan 10    ! assign to VLAN 10
Switch(config-if)# spanning-tree portfast       ! instant forwarding for end device
Switch(config-if)# no shutdown

! Range of ports — multiple devices same VLAN
Switch(config)# interface range GigabitEthernet0/1 - 4
Switch(config-if-range)# description "Sales Department PCs"
Switch(config-if-range)# switchport mode access
Switch(config-if-range)# switchport access vlan 10
Switch(config-if-range)# spanning-tree portfast

! IP phone port — data VLAN for PC + voice VLAN for phone
Switch(config)# interface GigabitEthernet0/5
Switch(config-if)# description "IP Phone + PC - Reception"
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10      ! PC traffic (untagged)
Switch(config-if)# switchport voice vlan 150      ! phone traffic (tagged CoS=5)
Switch(config-if)# spanning-tree portfast
Switch(config-if)# no shutdown

! Unused ports — disable and isolate
Switch(config)# interface range GigabitEthernet0/9 - 24
Switch(config-if-range)# description "UNUSED PORT - DISABLED"
Switch(config-if-range)# switchport mode access
Switch(config-if-range)# switchport access vlan 999  ! black hole VLAN
Switch(config-if-range)# shutdown                    ! physically disable

! Management SVI — IP address for switch management
Switch(config)# interface vlan 99
Switch(config-if)# description "Management Interface"
Switch(config-if)# ip address 192.168.99.10 255.255.255.0
Switch(config-if)# no shutdown
Switch(config)# ip default-gateway 192.168.99.1    ! for remote SSH access
```

### Verification Commands

```cisco
Switch# show vlan brief                         ! all VLANs + assigned ports
Switch# show vlan id 10                         ! detailed info for VLAN 10
Switch# show interfaces GigabitEthernet0/1 switchport   ! switchport config detail
Switch# show interfaces status                  ! all ports — status, VLAN, speed
Switch# show interfaces vlan 99                 ! management SVI status
Switch# show ip interface brief                 ! IP interfaces summary
```

---

## 7. VLAN Trunking — IEEE 802.1Q

### Trunk vs Access Port

```
ACCESS PORT:                         TRUNK PORT:
  One VLAN only                        Multiple VLANs
  Frames are UNTAGGED                  Frames are TAGGED (except native VLAN)
  Connects to end devices              Connects to switches, routers, APs
  (PCs, servers, printers)

  [PC] ──access── [Switch]            [Switch] ══trunk══ [Switch]
  VLAN 10 only                        VLAN 10+20+30+40
```

### 802.1Q Frame Tag Structure

```
Normal Ethernet Frame (untagged — access port):
┌──────────┬──────────┬───────────┬────────────────────┬─────┐
│ Dst MAC  │ Src MAC  │EtherType  │       Data         │ FCS │
└──────────┴──────────┴───────────┴────────────────────┴─────┘

802.1Q Tagged Frame (trunk port):
┌──────────┬──────────┬─────────────────────┬───────────┬────────────────────┬─────┐
│ Dst MAC  │ Src MAC  │     802.1Q TAG      │EtherType  │       Data         │ FCS │
│          │          │      (4 bytes)      │           │                    │     │
└──────────┴──────────┴─────────────────────┴───────────┴────────────────────┴─────┘
                                │
                    ┌───────────┴───────────────┐
                    │ TPID:  0x8100 (16 bits)   │ Identifies 802.1Q frame
                    │ PCP:   3 bits (0-7)        │ Priority Code Point (QoS/CoS)
                    │ DEI:   1 bit               │ Drop Eligible Indicator
                    │ VID:   12 bits (0-4094)    │ ← VLAN Identifier
                    └───────────────────────────┘

Note: FCS is recalculated after tag insertion
      Min frame size becomes 68 bytes (4 bytes added for tag)
```

### Native VLAN — Critical Concept

```
The native VLAN is the ONLY VLAN that travels UNTAGGED on a trunk.
Default native VLAN = VLAN 1

Switch-A (native = VLAN 1) ════trunk════ Switch-B (native = VLAN 1)
  Tagged VLAN 10 → [10][frame] ──────────────► VLAN 10 ✅
  Untagged frame → [frame] ──────────────────► VLAN 1 (native) ✅

NATIVE VLAN MISMATCH SCENARIO:
Switch-A (native = VLAN 1) ════trunk════ Switch-B (native = VLAN 10)

  Switch-A sends untagged frame (VLAN 1 traffic)
  Switch-B receives untagged frame → assigns it to VLAN 10
  VLAN 1 traffic ends up in VLAN 10 !!

  This is called VLAN HOPPING — a security vulnerability
  CDP reports: %CDP-4-NATIVE_VLAN_MISMATCH

SECURITY BEST PRACTICE:
  Change native VLAN to an UNUSED VLAN (e.g., 999) on ALL trunk ports
  No user traffic should be in the native VLAN
```

### Trunk Configuration

```cisco
! ═══ SWITCH PORT TRUNK CONFIGURATION ═══

Switch(config)# interface GigabitEthernet0/24
Switch(config-if)# description "Uplink to Core-Switch-01"

! On older switches needing explicit encapsulation:
Switch(config-if)# switchport trunk encapsulation dot1q    ! 802.1Q standard

! Set as trunk
Switch(config-if)# switchport mode trunk

! Security: change native VLAN from default (VLAN 1) to unused VLAN
Switch(config-if)# switchport trunk native vlan 999

! Security: restrict which VLANs are allowed (don't carry unnecessary VLANs)
Switch(config-if)# switchport trunk allowed vlan 10,20,30,40,99

! Add a VLAN to existing trunk (doesn't overwrite current list)
Switch(config-if)# switchport trunk allowed vlan add 150

! Remove a VLAN from trunk (clients on that VLAN lose connectivity across link)
Switch(config-if)# switchport trunk allowed vlan remove 40

! Allow all VLANs on trunk (not recommended — carries all VLANs including VLAN 1)
Switch(config-if)# switchport trunk allowed vlan all

Switch(config-if)# no shutdown

! ═══ VERIFICATION ═══
Switch# show interfaces GigabitEthernet0/24 trunk

Port        Mode         Encapsulation  Status        Native vlan
Gi0/24      on           802.1q         trunking      999

Port        Vlans allowed on trunk
Gi0/24      10,20,30,40,99,150

Port        Vlans allowed and active in management domain
Gi0/24      10,20,30,40,99,150

Port        Vlans in spanning tree forwarding state and not pruned
Gi0/24      10,20,30,40,99,150
```

---

## 8. DTP — Dynamic Trunking Protocol

DTP is Cisco proprietary — automatically negotiates trunk formation between switches.

### DTP Mode Matrix

```
OUR MODE          │ trunk │ desirable │ auto  │ access
──────────────────┼───────┼───────────┼───────┼────────
trunk             │ TRUNK │   TRUNK   │ TRUNK │ ACCESS
desirable         │ TRUNK │   TRUNK   │ TRUNK │ ACCESS
auto              │ TRUNK │   TRUNK   │ACCESS │ ACCESS
access            │ACCESS │   ACCESS  │ACCESS │ ACCESS

KEY RULES:
  auto + auto = ACCESS (neither initiates, no trunk formed)
  access + trunk = ACCESS (access mode wins — no trunk)
```

### Disabling DTP (Security Best Practice)

```cisco
! On access ports — mode access implicitly disables DTP
Switch(config-if)# switchport mode access
Switch(config-if)# switchport nonegotiate   ! explicitly ensure no DTP

! On trunk ports — disable DTP negotiation
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport nonegotiate   ! static trunk, no DTP frames sent

! WHY: DTP is a VLAN hopping attack vector
! Attacker sends DTP desirable frames → negotiates trunk → accesses all VLANs
```

---

## 9. Inter-VLAN Routing

### Why It's Needed

```
PC-A (VLAN 10: 192.168.10.10) wants to talk to PC-B (VLAN 20: 192.168.20.10)

Without routing:
  PC-A: default gateway = 192.168.10.1
  PC-A sends packet to its default gateway
  Gateway must ROUTE between VLAN 10 and VLAN 20
  Without this routing = no communication between VLANs

Three ways to provide this routing:
  1. Legacy: Separate physical router interfaces (one per VLAN)
  2. ROAS: Router on a Stick (subinterfaces)
  3. Layer 3 Switch: SVI routing (best for enterprise)
```

### Method 1 — Legacy (Separate Physical Links)

```
[Switch]──Gi0/1──[Router Gi0/0] 192.168.10.1   (VLAN 10 gateway)
[Switch]──Gi0/2──[Router Gi0/1] 192.168.20.1   (VLAN 20 gateway)

Pros:  Simple, dedicated bandwidth per VLAN
Cons:  Needs N physical links for N VLANs — doesn't scale
```

### Method 2 — Router on a Stick (ROAS)

```cisco
! ═══ SWITCH CONFIGURATION ═══
Switch(config)# interface GigabitEthernet0/24       ! uplink to router
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk native vlan 999
Switch(config-if)# switchport trunk allowed vlan 10,20,30,999
Switch(config-if)# no shutdown

! ═══ ROUTER CONFIGURATION ═══

! Physical interface — just enable it, no IP address
Router(config)# interface GigabitEthernet0/0
Router(config-if)# description "Trunk to Access-Switch"
Router(config-if)# no shutdown

! Subinterface for VLAN 10
Router(config)# interface GigabitEthernet0/0.10
Router(config-subif)# description "VLAN 10 - Sales Gateway"
Router(config-subif)# encapsulation dot1Q 10          ! process VLAN 10 tagged frames
Router(config-subif)# ip address 192.168.10.1 255.255.255.0

! Subinterface for VLAN 20
Router(config)# interface GigabitEthernet0/0.20
Router(config-subif)# description "VLAN 20 - HR Gateway"
Router(config-subif)# encapsulation dot1Q 20
Router(config-subif)# ip address 192.168.20.1 255.255.255.0

! Subinterface for VLAN 30
Router(config)# interface GigabitEthernet0/0.30
Router(config-subif)# description "VLAN 30 - Finance Gateway"
Router(config-subif)# encapsulation dot1Q 30
Router(config-subif)# ip address 192.168.30.1 255.255.255.0

! Native VLAN subinterface
Router(config)# interface GigabitEthernet0/0.999
Router(config-subif)# encapsulation dot1Q 999 native   ! native keyword = untagged
Router(config-subif)# ip address 192.168.99.1 255.255.255.0

! Verify
Router# show ip route                    ! should see 192.168.10.0/24, 20.0/24, etc.
Router# show interfaces GigabitEthernet0/0.10

! ROAS Limitation: All inter-VLAN traffic goes through ONE physical uplink
! Heavy inter-VLAN traffic = potential bottleneck at router
```

### Method 3 — Layer 3 Switch / SVI (Enterprise Best Practice)

```cisco
! ═══ LAYER 3 SWITCH CONFIGURATION ═══

! CRITICAL FIRST STEP: Enable IP routing
L3Switch(config)# ip routing           ! disabled by default — must enable!

! Create VLANs
L3Switch(config)# vlan 10
L3Switch(config-vlan)# name Sales
L3Switch(config)# vlan 20
L3Switch(config-vlan)# name HR
L3Switch(config)# vlan 30
L3Switch(config-vlan)# name Finance

! Create SVIs — virtual Layer 3 interfaces (one per VLAN)
L3Switch(config)# interface vlan 10
L3Switch(config-if)# description "VLAN10 Sales Gateway"
L3Switch(config-if)# ip address 192.168.10.1 255.255.255.0
L3Switch(config-if)# no shutdown
! SVI comes up only when VLAN exists AND at least one access port in that VLAN is up

L3Switch(config)# interface vlan 20
L3Switch(config-if)# description "VLAN20 HR Gateway"
L3Switch(config-if)# ip address 192.168.20.1 255.255.255.0
L3Switch(config-if)# no shutdown

L3Switch(config)# interface vlan 30
L3Switch(config-if)# description "VLAN30 Finance Gateway"
L3Switch(config-if)# ip address 192.168.30.1 255.255.255.0
L3Switch(config-if)# no shutdown

! Create routed port for uplink to router/firewall (optional)
L3Switch(config)# interface GigabitEthernet0/24
L3Switch(config-if)# no switchport                ! convert from switchport to L3 port
L3Switch(config-if)# description "Uplink to Firewall"
L3Switch(config-if)# ip address 10.0.0.2 255.255.255.252
L3Switch(config-if)# no shutdown

! Default route toward firewall/internet
L3Switch(config)# ip route 0.0.0.0 0.0.0.0 10.0.0.1

! Verify SVI status
L3Switch# show interfaces vlan 10
L3Switch# show ip interface brief   ! check all SVIs are up/up
L3Switch# show ip route             ! verify routing table has all VLAN networks
```

### ROAS vs L3 Switch Comparison

| Feature | ROAS | L3 Switch (SVI) |
|---------|------|----------------|
| Cost | Cheaper (uses existing router) | Higher (L3 switch needed) |
| Performance | Limited by one uplink | Wire-speed internal routing |
| Scalability | Poor (bottleneck) | Excellent |
| Complexity | Moderate | Moderate |
| Best for | Small offices, lab | Enterprise, campus networks |

---

## 10. VTP — VLAN Trunking Protocol

### VTP Modes Comparison

```
┌──────────────┬──────────┬──────────┬─────────────┬─────────────┐
│ VTP Mode     │ Create   │ Sync     │ Forward VTP │ Stores in   │
│              │ VLANs?   │ from     │ Ads?        │             │
│              │          │ Server?  │             │             │
├──────────────┼──────────┼──────────┼─────────────┼─────────────┤
│ SERVER       │ YES      │ YES      │ YES         │ vlan.dat    │
│ (default)    │          │          │             │             │
├──────────────┼──────────┼──────────┼─────────────┼─────────────┤
│ CLIENT       │ NO       │ YES      │ YES         │ Memory only │
├──────────────┼──────────┼──────────┼─────────────┼─────────────┤
│ TRANSPARENT  │ YES      │ NO       │ YES (relay) │ running-cfg │
│              │ (local)  │          │             │             │
├──────────────┼──────────┼──────────┼─────────────┼─────────────┤
│ OFF (v3)     │ YES      │ NO       │ NO          │ running-cfg │
│              │ (local)  │          │             │             │
└──────────────┴──────────┴──────────┴─────────────┴─────────────┘
```

### VTP Revision Number — The Catastrophic Risk

```
⚠️ VTP REVISION NUMBER DANGER SCENARIO:

PRODUCTION ENVIRONMENT:
  Core-Switch:   VTP Server, domain=PROD, revision=45, VLANs 10,20,30,40,99
  Access-SW-A:   VTP Client, revision=45
  Access-SW-B:   VTP Client, revision=45
  Network is stable and working fine.

THE DANGER:
  Someone takes "Spare-Switch" from the storage room
  It was previously used in a lab
  It has: VTP domain=PROD, revision=78 (higher than 45!)
  It has: VLANs 10,20 only (most VLANs were deleted in the lab)

  Person plugs Spare-Switch into production trunk port.

WHAT HAPPENS:
  Spare-Switch sends VTP advertisement: domain=PROD, revision=78
  Core-Switch sees: 78 > 45 → "This is newer! I must sync from it!"
  Core-Switch OVERWRITES its VLAN database with Spare-Switch's
  VLANs 30, 40, 99 are DELETED from all switches
  All ports in those VLANs lose connectivity
  Potentially ENTIRE NETWORK IS DOWN

HOW TO PREVENT:
  Option A: Before connecting, reset revision number
    Switch(config)# vtp mode transparent
    Switch(config)# vtp mode client         ! revision resets to 0
  Option B: Change VTP domain before connecting
    Switch(config)# vtp domain TEMP_DUMMY
    Switch(config)# vtp domain PROD         ! revision resets to 0
  Option C: Use VTPv3 (primary server concept — explicit authorization)
  Option D: Use VTP transparent on all switches (no VTP sync at all)
```

### VTP Configuration

```cisco
! Configure VTP server
Switch(config)# vtp mode server
Switch(config)# vtp domain COMPANY_PROD      ! case-sensitive!
Switch(config)# vtp password Cisco@2024      ! must match all switches
Switch(config)# vtp version 2

! Configure VTP client
Switch(config)# vtp mode client
Switch(config)# vtp domain COMPANY_PROD
Switch(config)# vtp password Cisco@2024

! Transparent mode (recommended for most environments)
Switch(config)# vtp mode transparent
Switch(config)# vtp domain COMPANY_PROD      ! domain needed to relay VTP

! Verify
Switch# show vtp status
VTP Version capable             : 1 to 3
VTP version running             : 2
VTP Domain Name                 : COMPANY_PROD
VTP Operating Mode              : Server
Number of existing VLANs        : 8
Configuration Revision          : 45       ← Watch this number!
```

---

## 11. STP — Spanning Tree Protocol (802.1D)

### The Loop Problem

```
Without STP — Redundant switches create a loop:

         [Switch-A]
        /            \
   [Switch-B]────[Switch-C]

PC on SW-B sends broadcast:
  SW-B floods to SW-A and SW-C
  SW-A floods to SW-C
  SW-C floods to SW-B (already received this!)
  SW-B floods AGAIN to SW-A and SW-C
  INFINITE LOOP!

Effects:
  • Broadcast storm — 100% bandwidth in under a second
  • MAC table flapping (same MAC appears on multiple ports constantly)
  • All legitimate traffic completely blocked
  • Switch CPU at 100% processing loops
  • Possible physical hardware damage (overheating)

STP solution:
  Block ONE link in the loop → no loop possible
  Redundancy preserved (blocked port activates if active path fails)
```

### STP Election Process

#### Step 1: Root Bridge Election

```
Every switch starts assuming IT is the Root Bridge.
Switches exchange BPDUs. Switch with LOWEST Bridge ID wins.

Bridge ID (64 bits total):
  Bridge Priority: 0-65535 (default 32768, increments of 4096)
  System ID Extension: VLAN ID (for PVST+)
  MAC Address: 48 bits (tiebreaker)

Example election:
  Switch-A: priority 32769, MAC 0000.0000.000A → Bridge ID = 32769.000A
  Switch-B: priority 32769, MAC 0000.0000.000B → Bridge ID = 32769.000B
  Switch-C: priority 4096,  MAC 0000.0000.000C → Bridge ID = 04096.000C

  WINNER = Switch-C (lowest priority: 4096)
  Switch-C = ROOT BRIDGE

  All ports on Root Bridge = DESIGNATED (forwarding)
```

#### Step 2: Root Port Election (per non-root switch)

```
Each non-root switch selects ONE Root Port = port with lowest COST PATH to root.

Port cost values (IEEE 802.1D):
  10 Mbps    → cost 100
  100 Mbps   → cost 19
  1 Gbps     → cost 4
  10 Gbps    → cost 2

Example:
  Switch-A connected to Root (Switch-C) via 1 Gbps → cost = 4
  Switch-A also connected via Switch-B (100Mbps+100Mbps) → cost = 19+19 = 38
  Switch-A Root Port = direct link to Switch-C (cost 4 wins)

Tiebreakers (lowest wins):
  1. Lowest root path cost (cumulative cost to root)
  2. Lowest upstream Bridge ID (neighbor's Bridge ID)
  3. Lowest upstream Port ID (neighbor's port priority + port number)
  4. Lowest local Port ID
```

#### Step 3: Designated Port Election (per segment)

```
Each network segment needs ONE Designated Port = port closest to root
(the port that will FORWARD toward end devices)

Rule:
  Root Bridge: ALL ports = Designated
  Other switches: Port with best path to root = Designated for that segment
  Loser = Blocked (Alternate port)
```

#### Step 4: Remaining Ports → Blocked

```
Final STP topology:
  Root Port → forwarding (non-root switches, path to root)
  Designated Port → forwarding (best port on each segment)
  Alternate Port → BLOCKED (prevents loop, standby for failover)
```

### STP Port States

```
State          Receives BPDUs  Learns MACs  Forwards Frames  Duration
─────────────  ──────────────  ───────────  ───────────────  ────────────
DISABLED       NO              NO           NO               Until enabled
BLOCKING       YES             NO           NO               Up to 20 sec
LISTENING      YES             NO           NO               15 sec (Forward Delay)
LEARNING       YES             YES          NO               15 sec (Forward Delay)
FORWARDING     YES             YES          YES              Stable (indefinite)

Total time from link up to forwarding (worst case):
  20s (Max Age) + 15s (Listening) + 15s (Learning) = 50 seconds!

During this 50 seconds:
  Workstation gets link → NIC shows connected → tries DHCP
  DHCP times out (switch port still in Listening/Learning)
  User sees no network! They call helpdesk!
  Solution: PortFast (Section 13)
```

### STP Timers

| Timer | Default | Description |
|-------|---------|-------------|
| **Hello Time** | 2 seconds | Interval between Root Bridge BPDU transmissions |
| **Max Age** | 20 seconds | How long to store BPDU before discarding (if no hello received) |
| **Forward Delay** | 15 seconds | Time in Listening state AND time in Learning state |

```cisco
! Modify timers (change only on Root Bridge — propagated via BPDUs)
Switch(config)# spanning-tree vlan 10 hello-time 1
Switch(config)# spanning-tree vlan 10 max-age 14
Switch(config)# spanning-tree vlan 10 forward-time 10

! Better alternative: use RSTP (802.1w) instead of changing timers
```

### STP Configuration Commands

```cisco
! Set bridge priority (make this switch Root Bridge for VLAN 10)
Switch(config)# spanning-tree vlan 10 priority 4096
! Must be multiple of 4096: 0, 4096, 8192, 12288, 16384, 20480, 24576...
! Lower = more likely to be root. Priority 0 = guaranteed root

! Easier macro commands
Switch(config)# spanning-tree vlan 10 root primary     ! sets to 24576 (or lower)
Switch(config)# spanning-tree vlan 20 root secondary   ! sets to 28672

! Manually set port cost (influences Root Port selection)
Switch(config-if)# spanning-tree vlan 10 cost 10

! Manually set port priority (tiebreaker — lower = preferred)
Switch(config-if)# spanning-tree vlan 10 port-priority 64   ! default 128

! Verification
Switch# show spanning-tree vlan 10

VLAN0010
  Spanning tree enabled protocol ieee
  Root ID    Priority    4096
             Address     0000.0000.000C
             Cost        4               ← cost to reach root
             Port        1 (GigabitEthernet0/1)  ← root port
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    32778  (priority 32768 sys-id-ext 10)
             Address     0000.0000.000A
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- ----------------
Gi0/1               Root FWD 4         128.1    P2p
Gi0/2               Desg FWD 4         128.2    P2p
Gi0/3               Altn BLK 4         128.3    P2p   ← BLOCKED!
```

---

## 12. RSTP — Rapid Spanning Tree (802.1w)

### Why RSTP Was Needed

```
802.1D STP problems:
  • 50 seconds convergence — unacceptable for modern networks
  • Based on passive timers — must wait for timers to expire
  • No concept of quick backup path
  • Separate proposal/agreement required for every change

RSTP (802.1w) improvements:
  • 1-6 second convergence — uses active negotiation
  • Pre-identifies alternate paths (Alternate Port = ready backup)
  • Proposal/Agreement mechanism — handshake-based state changes
  • Backwards compatible with 802.1D
```

### RSTP Port Roles

| Role | Function | State |
|------|---------|-------|
| **Root Port** | Best path to Root Bridge | Forwarding |
| **Designated Port** | Best port on segment toward root | Forwarding |
| **Alternate Port** | Backup to Root Port (best alternate path to root) | Discarding |
| **Backup Port** | Backup to Designated Port (same segment) | Discarding |

### RSTP Port States

```
802.1D states:          RSTP states:
  Disabled         →      Discarding
  Blocking         →      Discarding
  Listening        →      (eliminated — replaced by P/A negotiation)
  Learning         →      Learning
  Forwarding       →      Forwarding
```

### RSTP Proposal/Agreement Mechanism

```
RSTP achieves rapid convergence via handshake:

New link comes up between Switch-A and Switch-B:

Switch-A sends BPDU with Proposal bit set:
  "I propose to become Designated (forwarding) on this link"

Switch-B evaluates:
  Is Switch-A superior? If YES:
  Switch-B immediately syncs (blocks all non-edge ports)
  Switch-B sends BPDU with Agreement bit set

Switch-A receives Agreement:
  Immediately transitions port to Forwarding!
  No waiting for timers!

Result: Convergence in milliseconds to seconds
```

### RSTP Link Types

| Type | Detected By | Behavior |
|------|------------|---------|
| **Point-to-Point** | Full-duplex | Fast convergence (P/A mechanism) |
| **Shared** | Half-duplex | Falls back to 802.1D behavior (timers) |
| **Edge** | PortFast configured | Instant Forwarding — connected to end device |

### PVST+ and Rapid PVST+

```
Standard 802.1D/802.1w:
  Single STP instance for ALL VLANs
  All VLANs use same blocked ports
  Can't load balance traffic

Cisco PVST+ (Per-VLAN Spanning Tree Plus):
  Separate STP instance PER VLAN
  Different Root Bridge for different VLANs
  Load balancing between links!

Example:
  VLAN 10 Root = Switch-A → uses Gi0/1 uplink
  VLAN 20 Root = Switch-B → uses Gi0/2 uplink
  VLAN 30 Root = Switch-A → uses Gi0/1 uplink
  All uplinks carrying traffic — efficient!

Rapid PVST+ = RSTP + per-VLAN instances (Cisco default)
```

```cisco
! Set STP mode
Switch(config)# spanning-tree mode rapid-pvst    ! Rapid PVST+ (Cisco default, recommended)
Switch(config)# spanning-tree mode pvst          ! Classic PVST+
Switch(config)# spanning-tree mode mst           ! Multiple Spanning Tree (802.1s)

! Verify mode
Switch# show spanning-tree summary
Switch is in rapid-pvst mode
Root bridge for: VLAN0010, VLAN0099
```

---

## 13. STP Enhancements — PortFast, BPDU Guard, Root Guard

### PortFast

Skips Listening and Learning states → immediate Forwarding. For access ports connected to end devices only.

```
Normal STP (30s before traffic):
Link Up → Blocking → Listening(15s) → Learning(15s) → Forwarding

With PortFast (immediate):
Link Up ────────────────────────────────────────────► Forwarding
PC gets IP immediately — no DHCP timeout!
```

```cisco
! Enable PortFast on individual port
Switch(config-if)# spanning-tree portfast

! Enable PortFast globally (applies to ALL non-trunking ports)
Switch(config)# spanning-tree portfast default

! Disable PortFast on specific port (override global setting)
Switch(config-if)# spanning-tree portfast disable

! Verify
Switch# show spanning-tree interface GigabitEthernet0/1 portfast
VLAN0010           enabled
```

> ⚠️ **NEVER configure PortFast on a switch-to-switch link.** If a switch is plugged into a PortFast port, BPDU Guard should catch it. PortFast only for PC/server/printer connections.

### BPDU Guard

If a PortFast-enabled port receives ANY BPDU → **immediately err-disables the port**.

Protects against:
- Unauthorized switches connected to user ports
- BPDU-based STP manipulation attacks
- Accidental loops from plugging switch into access port

```cisco
! Enable BPDU Guard on specific port
Switch(config-if)# spanning-tree bpduguard enable

! Enable BPDU Guard globally (automatically applied to all PortFast ports)
Switch(config)# spanning-tree portfast bpduguard default

! When BPDU Guard triggers:
! Port goes err-disabled — appears as:
Switch# show interfaces status err-disabled
Port      Name               Status       Reason               Err-disabled
Gi0/3                        err-disabled bpduguard

! Recovery options:
! Option 1: Manual
Switch(config-if)# shutdown
Switch(config-if)# no shutdown

! Option 2: Automatic (after timeout period)
Switch(config)# errdisable recovery cause bpduguard
Switch(config)# errdisable recovery interval 300    ! 5 minutes
```

### Root Guard

Prevents a port from becoming a Root Port. Protects the Root Bridge position.

```cisco
! Apply on ports toward untrusted devices/switches
Switch(config-if)# spanning-tree guard root

! If superior BPDU received → port goes root-inconsistent (blocked)
! Automatically recovers when superior BPDUs stop

Switch# show spanning-tree inconsistentports
Name                 Interface            Inconsistency
-------------------- -------------------- ------------------
VLAN0010             GigabitEthernet0/24  Root Inconsistent
```

### Loop Guard

Prevents Alternate (Blocked) ports from transitioning to Forwarding when BPDUs stop arriving (unidirectional link failure).

```cisco
Switch(config-if)# spanning-tree guard loop
! OR globally:
Switch(config)# spanning-tree loopguard default
```

### STP Enhancements Summary

```
Feature       │ Applied To           │ Protects Against            │ Action
──────────────┼──────────────────────┼─────────────────────────────┼───────────────────
PortFast      │ Access ports only    │ Slow convergence for PCs     │ Skip L/L states
BPDU Guard    │ PortFast ports       │ Unauthorized switches,       │ Err-disable port
              │                      │ STP manipulation attacks     │
Root Guard    │ Non-Root ports       │ Unauthorized Root Bridge     │ Root-inconsistent
              │ (toward untrusted)   │ takeover                     │ (blocked)
Loop Guard    │ Root/Alternate ports │ Unidirectional link failures │ Loop-inconsistent
              │                      │ causing loops                │ (blocked)
BPDU Filter   │ Access ports         │ Sending unnecessary BPDUs    │ Ignores BPDUs
              │                      │ (use carefully!)             │
```

---

## 14. EtherChannel / Link Aggregation

### The Problem EtherChannel Solves

```
Redundant links + STP = wasted bandwidth:

        [Core Switch]
       /              \
  Gi0/1            Gi0/2
(active)         (BLOCKED by STP)
      \              /
       [Access Switch]
       
  Without EtherChannel: 1 Gbps usable, 1 Gbps blocked (wasted)
  With EtherChannel: 2 Gbps usable (STP sees as ONE logical link)
```

### EtherChannel Protocols

```
LACP (IEEE 802.3ad) — Open Standard, vendor-neutral:
  active  → initiates LACP negotiation
  passive → responds to LACP (does NOT initiate)
  active + active  = channel ✅
  active + passive = channel ✅
  passive + passive = NO channel ✗ (neither initiates!)

PAgP (Cisco Proprietary) — Cisco-only:
  desirable → initiates PAgP negotiation
  auto      → responds to PAgP (does NOT initiate)
  desirable + desirable = channel ✅
  desirable + auto      = channel ✅
  auto + auto           = NO channel ✗ (neither initiates!)

Static (On) — No negotiation:
  on → forces channel formation, no protocol
  on + on = channel ✅ (but no protection against misconfiguration)
  on + LACP/PAgP = NO channel ✗ (protocol mismatch)
```

### EtherChannel Requirements (ALL Must Match)

```
For an EtherChannel to form, ALL member ports must have:
  ✓ Same speed
  ✓ Same duplex
  ✓ Same access VLAN (if access ports)
  ✓ Same trunk settings (if trunk ports):
      - Same native VLAN
      - Same allowed VLANs
      - Same encapsulation
  ✓ Same STP settings

Mismatch in ANY setting → EtherChannel fails to form
Switch will err-disable the channel!
```

### EtherChannel Configuration

```cisco
! ════════════ SWITCH-A ════════════
Switch-A(config)# interface range GigabitEthernet0/1 - 4

! First: configure L2 settings on physical interfaces
Switch-A(config-if-range)# switchport mode trunk
Switch-A(config-if-range)# switchport trunk encapsulation dot1q
Switch-A(config-if-range)# switchport trunk native vlan 999
Switch-A(config-if-range)# switchport trunk allowed vlan 10,20,30,99
Switch-A(config-if-range)# shutdown    ! shutdown before adding to channel

! Add to EtherChannel with LACP active mode
Switch-A(config-if-range)# channel-group 1 mode active
Switch-A(config-if-range)# no shutdown

! Configure Port-Channel logical interface
Switch-A(config)# interface Port-channel1
Switch-A(config-if)# description "4x1G LACP EtherChannel to Switch-B"
Switch-A(config-if)# switchport mode trunk
Switch-A(config-if)# switchport trunk encapsulation dot1q
Switch-A(config-if)# switchport trunk native vlan 999
Switch-A(config-if)# switchport trunk allowed vlan 10,20,30,99

! ════════════ SWITCH-B (same config, active mode) ════════════
Switch-B(config)# interface range GigabitEthernet0/1 - 4
Switch-B(config-if-range)# switchport mode trunk
Switch-B(config-if-range)# switchport trunk encapsulation dot1q
Switch-B(config-if-range)# switchport trunk native vlan 999
Switch-B(config-if-range)# switchport trunk allowed vlan 10,20,30,99
Switch-B(config-if-range)# channel-group 1 mode active    ! also active — both can initiate
Switch-B(config-if-range)# no shutdown

Switch-B(config)# interface Port-channel1
Switch-B(config-if)# switchport mode trunk
Switch-B(config-if)# switchport trunk encapsulation dot1q
Switch-B(config-if)# switchport trunk native vlan 999
Switch-B(config-if)# switchport trunk allowed vlan 10,20,30,99
```

### EtherChannel Verification

```cisco
Switch# show etherchannel summary
Flags:  D - down   P - bundled in port-channel
        I - stand-alone   s - suspended
        H - Hot-standby (LACP only)
        R - Layer3   S - Layer2
        U - in use   f - failed to allocate aggregator

Group  Port-channel  Protocol    Ports
------+-------------+-----------+-------------------------------------------
1      Po1(SU)         LACP      Gi0/1(P) Gi0/2(P) Gi0/3(P) Gi0/4(P)
!                                         ↑ P = bundled and active ✅

Switch# show etherchannel 1 port-channel
Switch# show etherchannel load-balance
Switch# show lacp neighbor
Switch# show lacp counters

! Check STP — should see Port-channel as ONE link:
Switch# show spanning-tree vlan 10
Interface           Role Sts Cost   Type
Po1                 Root FWD 3      P2p   ← port-channel as one STP link
```

### Load Balancing Algorithms

```cisco
! Set load balancing method
Switch(config)# port-channel load-balance src-dst-ip    ! best for routed traffic
Switch(config)# port-channel load-balance src-dst-mac   ! best for L2 traffic
Switch(config)# port-channel load-balance src-port      ! port-based hashing

! Verify
Switch# show etherchannel load-balance
EtherChannel Load-Balancing Configuration:
        src-dst-ip

! Note: EtherChannel is NOT round-robin per frame
! Same src-dst IP pair always takes same link (ensures ordered delivery)
! Different flows hash to different links
```

---

## 15. Port Security

### Complete Port Security Configuration

```cisco
! Enable port security (must be on access port)
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10
Switch(config-if)# switchport port-security            ! enable port security

! Maximum MAC addresses allowed (default = 1)
Switch(config-if)# switchport port-security maximum 2  ! allow max 2 MACs

! How MACs are defined:
! Method 1 — Static (manually configure exact MAC — most secure)
Switch(config-if)# switchport port-security mac-address 0050.7966.6800

! Method 2 — Sticky (auto-learn and SAVE to running-config)
Switch(config-if)# switchport port-security mac-address sticky
! First MAC(s) that connect are saved permanently in running-config

! Method 3 — Dynamic (auto-learn but LOST on reboot — default)
! (no extra command needed — this is default behavior)

! Violation action (what happens when policy violated):
Switch(config-if)# switchport port-security violation shutdown   ! default
! Other options: restrict | protect
```

### Violation Mode Comparison

```
VIOLATION MODE: SHUTDOWN (default)
  Trigger: Wrong MAC or max MACs exceeded
  Action:  Port immediately goes err-disabled
           No traffic at all can pass
  Logging: Syslog message generated
           SNMP trap sent
           Violation counter incremented
  Recovery: Manual (shutdown/no shutdown) or auto-recover timer
  Use when: Zero-tolerance security (corporate, financial)

VIOLATION MODE: RESTRICT
  Trigger: Wrong MAC or max MACs exceeded
  Action:  Violating frames DROPPED (other traffic still passes)
           Port stays UP for legitimate traffic
  Logging: Syslog message generated
           SNMP trap sent
           Violation counter incremented
  Recovery: Not needed (port stays up)
  Use when: Want to alert but not disrupt service

VIOLATION MODE: PROTECT
  Trigger: Wrong MAC or max MACs exceeded
  Action:  Violating frames DROPPED silently
           Port stays UP for legitimate traffic
  Logging: NO log message
           NO SNMP trap
           Violation counter NOT incremented
  Recovery: Not needed (port stays up)
  Use when: Want to silently filter (rarely recommended)
```

### Port Security Verification

```cisco
! Overall port security status for all ports
Switch# show port-security
Secure Port  MaxSecureAddr  CurrentAddr  SecurityViolation  Security Action
              (Count)        (Count)          (Count)
-----------  -------------  -----------  -----------------  ---------------
     Gi0/1          2              1                0            Shutdown
     Gi0/2          1              1                0            Restrict

! Detail for specific port
Switch# show port-security interface GigabitEthernet0/1
Port Security              : Enabled
Port Status                : Secure-up
Violation Mode             : Shutdown
Aging Time                 : 0 mins
Aging Type                 : Absolute
SecureStatic Address Aging : Disabled
Maximum MAC Addresses      : 2
Total MAC Addresses        : 1
Configured MAC Addresses   : 1
Sticky MAC Addresses       : 0
Last Source Address:Vlan   : 0050.7966.6800:10
Security Violation Count   : 0

! View all secured MAC addresses
Switch# show port-security address

! Check err-disabled ports
Switch# show interfaces status err-disabled
Switch# show errdisable reason
Switch# show errdisable recovery

! Configure automatic recovery from err-disabled
Switch(config)# errdisable recovery cause psecure-violation
Switch(config)# errdisable recovery interval 300      ! recover after 5 minutes
```

---

## 16. Switch Hardening Best Practices

```cisco
! ════════════ MANAGEMENT PLANE SECURITY ════════════

Switch(config)# hostname AccessSwitch-Floor2-Room101
Switch(config)# ip domain-name company.local
Switch(config)# crypto key generate rsa modulus 2048
Switch(config)# ip ssh version 2
Switch(config)# ip ssh time-out 60
Switch(config)# ip ssh authentication-retries 3

! Create admin user
Switch(config)# username admin privilege 15 algorithm-type scrypt secret Str0ngP@ss!

! Set enable secret
Switch(config)# enable algorithm-type scrypt secret Str0ngEn@ble!

! VTY lines — SSH only, require authentication
Switch(config)# line vty 0 15
Switch(config-line)# transport input ssh
Switch(config-line)# login local
Switch(config-line)# exec-timeout 5 0
Switch(config-line)# logging synchronous

! Console security
Switch(config)# line console 0
Switch(config-line)# login local
Switch(config-line)# exec-timeout 5 0

! Disable unused services
Switch(config)# no ip http server
Switch(config)# ip http secure-server          ! HTTPS management only
Switch(config)# no service tcp-small-servers
Switch(config)# no service udp-small-servers
Switch(config)# no cdp run                     ! disable CDP globally (optional)

! ════════════ VLAN SECURITY ════════════

! Move all unused ports to black hole VLAN and shut them down
Switch(config)# interface range GigabitEthernet0/9 - 24
Switch(config-if-range)# description "UNUSED - DISABLED"
Switch(config-if-range)# switchport mode access
Switch(config-if-range)# switchport access vlan 999
Switch(config-if-range)# shutdown

! Disable DTP everywhere
Switch(config)# interface range GigabitEthernet0/1 - 24
Switch(config-if-range)# switchport nonegotiate

! Change native VLAN on trunks
Switch(config)# interface GigabitEthernet0/24
Switch(config-if)# switchport trunk native vlan 999

! ════════════ STP SECURITY ════════════

Switch(config)# spanning-tree portfast default          ! PortFast on all access
Switch(config)# spanning-tree portfast bpduguard default ! BPDU Guard on all PortFast
Switch(config)# spanning-tree loopguard default         ! Loop Guard globally

Switch(config)# interface GigabitEthernet0/24           ! uplink
Switch(config-if)# spanning-tree guard root              ! Root Guard on uplink

! ════════════ PORT SECURITY (access ports) ════════════

Switch(config)# interface range GigabitEthernet0/1 - 8
Switch(config-if-range)# switchport port-security
Switch(config-if-range)# switchport port-security maximum 2
Switch(config-if-range)# switchport port-security mac-address sticky
Switch(config-if-range)# switchport port-security violation restrict

! ════════════ DHCP SNOOPING ════════════

Switch(config)# ip dhcp snooping
Switch(config)# ip dhcp snooping vlan 10,20,30,99
Switch(config)# interface GigabitEthernet0/24
Switch(config-if)# ip dhcp snooping trust               ! only uplink is trusted

! ════════════ DYNAMIC ARP INSPECTION ════════════

Switch(config)# ip arp inspection vlan 10,20,30,99
Switch(config)# interface GigabitEthernet0/24
Switch(config-if)# ip arp inspection trust

! ════════════ LOGGING ════════════

Switch(config)# logging host 192.168.99.50              ! syslog server
Switch(config)# logging trap informational
Switch(config)# service timestamps log datetime msec localtime show-timezone
Switch(config)# ntp server 192.168.99.1                 ! sync time for accurate logs
```

---

## 17. Key Takeaways & Exam Tips

### Key Takeaways

- Switches forward frames based on **MAC address table** — unknown = flood, known = unicast forward
- Each switch port = **separate collision domain**; without VLANs = **one broadcast domain**
- **VLANs** = logical broadcast domain separation on one physical switch
- **802.1Q trunk** carries multiple VLANs using 4-byte frame tags; **native VLAN** is untagged
- **DTP** = auto trunk negotiation — **disable it** (`nonegotiate`) for security
- **ROAS** = one router interface + subinterfaces; **L3 Switch SVI** = better enterprise choice
- **VTP** propagates VLAN config — higher revision number wins — **reset before connecting new switches**
- **STP (802.1D)** = ~50 second convergence; blocks redundant paths to prevent loops
- **RSTP (802.1w)** = 1–6 second convergence; uses proposal/agreement; Cisco default = **Rapid PVST+**
- **PortFast** = skip Listening/Learning on access ports; **BPDU Guard** = err-disable if BPDU received
- **EtherChannel** = bundle links (LACP/PAgP/static); STP sees as one link; all links active
- **Port Security** = limit MACs per port; modes: shutdown (err-disable), restrict (drop+log), protect (silent)

### Exam Tips

> 🎯 **Most Tested Topics:**

1. **"Default STP bridge priority?"** → **32768** (must be multiple of 4096)
2. **"STP state that learns MACs but doesn't forward?"** → **Learning**
3. **"Total STP convergence time (802.1D)?"** → **~50 seconds** (20 Max Age + 15 Listening + 15 Learning)
4. **"Default native VLAN?"** → **VLAN 1** — change it for security!
5. **"What does PortFast do?"** → Skips Listening and Learning states → immediate Forwarding
6. **"BPDU Guard action?"** → **Err-disables** the port when BPDU received
7. **"LACP passive + passive = ?"** → **No EtherChannel** — neither side initiates
8. **"LACP active + passive = ?"** → **EtherChannel forms** ✅
9. **"What STP variant does Cisco use by default?"** → **Rapid PVST+**
10. **"Port security mode that silently drops (no log)?"** → **protect**
11. **"Port security mode that keeps port up but logs violations?"** → **restrict**
12. **"`ip routing` command is needed on which device?"** → **Layer 3 switch** for inter-VLAN routing via SVIs
13. **"What ROAS encapsulation command is required on subinterface?"** → `encapsulation dot1Q [vlan-id]`
14. **"VTP transparent mode behavior?"** → Does NOT sync VLAN DB, DOES forward VTP advertisements (relay)
15. **"What is the VTP danger?"** → Higher revision number switch **overwrites** VLAN database on entire domain

---

*← Previous: [05 — TCP vs UDP](./05_TCP_vs_UDP.md)*  
*Next: [07 — Routing Fundamentals →](./07_Routing_Fundamentals.md)*
