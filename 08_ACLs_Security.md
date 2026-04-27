# 08 — ACLs & Network Security: Complete Reference

> **CCNA Exam Domain:** 5.0 Security Fundamentals  
> **Exam Weight:** ~15% of total exam  
> **Difficulty:** ⭐⭐⭐☆☆

---

## Table of Contents

1. [Network Security Fundamentals](#1-network-security-fundamentals)
2. [Access Control Lists — Core Concepts](#2-access-control-lists--core-concepts)
3. [Standard ACLs](#3-standard-acls)
4. [Extended ACLs](#4-extended-acls)
5. [Named ACLs](#5-named-acls)
6. [ACL Placement Rules](#6-acl-placement-rules)
7. [ACL Best Practices & Troubleshooting](#7-acl-best-practices--troubleshooting)
8. [AAA — Authentication, Authorization, Accounting](#8-aaa--authentication-authorization-accounting)
9. [Port Security](#9-port-security)
10. [DHCP Snooping](#10-dhcp-snooping)
11. [Dynamic ARP Inspection (DAI)](#11-dynamic-arp-inspection-dai)
12. [Layer 2 Security Threats & Mitigations](#12-layer-2-security-threats--mitigations)
13. [VPN Fundamentals](#13-vpn-fundamentals)
14. [Wireless Security](#14-wireless-security)
15. [Key Takeaways & Exam Tips](#15-key-takeaways--exam-tips)

---

## 1. Network Security Fundamentals

### The CIA Triad — Foundation of Security

Every security decision in networking is based on protecting three fundamental properties. Think of these as the three legs of a stool — remove any one and the whole structure falls.

```
        CONFIDENTIALITY
        Only authorized users
        can read the data
              │
              │
  ┌───────────┴───────────┐
  │                       │
INTEGRITY             AVAILABILITY
Data is accurate       Systems are
and unmodified         up and accessible
in transit             when needed
```

**Confidentiality** means ensuring that sensitive data — passwords, financial records, personal information — can only be seen by those who have the right to see it. Encryption is the primary tool here (TLS for web traffic, IPSec for VPNs).

**Integrity** ensures that data has not been modified in transit, either accidentally (bit errors) or deliberately (man-in-the-middle tampering). Hashing algorithms like SHA-256 and HMAC protect integrity.

**Availability** means that services and systems are up and running when legitimate users need them. Denial-of-service attacks target availability. Redundancy, rate limiting, and firewalls protect it.

### Common Network Threats

Understanding what attackers do helps you understand why each security feature exists. Every tool we configure in this module is a direct response to one or more of these threats.

```
Threat               Layer   Description                     Mitigation
───────────────────  ──────  ──────────────────────────────  ─────────────────────
MAC Flooding         L2      Fill CAM table → switch floods  Port Security
VLAN Hopping         L2      Access other VLANs via DTP/tag  Disable DTP, change native VLAN
DHCP Spoofing        L2/L3   Rogue DHCP server → MITM setup DHCP Snooping
ARP Poisoning        L2/L3   Fake ARP replies → redirect     Dynamic ARP Inspection (DAI)
Man-in-the-Middle    L3-L7   Intercept communication         Encryption (TLS/IPSec)
DoS / DDoS           L3-L7   Flood target → unavailable      Rate limiting, ACLs, upstream
IP Spoofing          L3      Forge source IP address         ACLs, uRPF
Reconnaissance       L3-L7   Port scanning, network mapping  Firewall, IDS/IPS
Unauthorized Access  L4-L7   Access without permission       ACLs, 802.1X, AAA
Password Attacks     L7      Brute force, dictionary         Strong passwords, lockout
Phishing             L7      Trick users into giving creds   User training, email filtering
```

### Defense in Depth

No single security control is sufficient. The professional approach is **layered security** — multiple overlapping defenses so that if one fails, others still protect the network.

```
INTERNET
    │
[Firewall]          ← Layer 1: Perimeter — blocks known bad traffic
    │
[IPS/IDS]           ← Layer 2: Detection — spots and stops attacks
    │
[Router with ACLs]  ← Layer 3: Routing layer — restricts traffic flow
    │
[Switch with]       ← Layer 4: LAN security — DHCP snooping, DAI, port security
[Security Features]
    │
[802.1X on ports]   ← Layer 5: Identity — only auth devices connect
    │
[Endpoint Security] ← Layer 6: Host — antivirus, host firewall
    │
[User Training]     ← Layer 7: Human — phishing awareness
```

---

## 2. Access Control Lists — Core Concepts

### What Is an ACL?

An **Access Control List (ACL)** is an ordered sequence of **permit** or **deny** statements — called **ACEs (Access Control Entries)** — that a router evaluates against incoming or outgoing traffic to decide whether to allow or block each packet.

Think of an ACL like a bouncer at a nightclub with a list of rules: check rule 1, if it matches act on it; if not, check rule 2; and so on. The bouncer never skips ahead, and once a rule matches, the decision is final.

### The Implicit Deny — Most Critical Concept

Every ACL — without exception — ends with an **invisible implicit deny all** statement. You never see it in the configuration, but it is always there. This means if a packet does not match ANY ACE in the ACL, it is **dropped silently**.

```
ACL 10:
  10 permit 192.168.1.0 0.0.0.255
  20 permit 10.0.0.0 0.255.255.255
  [implicit deny any]   ← invisible, but ALWAYS present!

Packet from 172.16.1.5:
  Check rule 10 → no match
  Check rule 20 → no match
  End of ACL → implicit deny → PACKET DROPPED!

This is why you almost always need a "permit any" at the end
if you want non-matching traffic to pass through.
```

> ⚠️ **The most common ACL mistake** is forgetting the implicit deny and blocking legitimate traffic you intended to allow. Always think: "What happens to everything I did NOT explicitly permit?"

### How ACL Processing Works

```
Packet arrives at interface with ACL applied
                │
                ▼
        ┌── Check ACE #1 ──┐
        │                  │
      Match?             No match
        │                  │
        ▼                  ▼
  Apply action        Check ACE #2
  (permit/deny)            │
  STOP processing        Match?
                           │
                     Yes ──┴── No
                     │         │
                 Apply       Check ACE #3
                 action           │
                 STOP            ...
                                  │
                           End of ACL reached
                                  │
                           IMPLICIT DENY ALL
                           (packet dropped)
```

### ACL Types Summary

```
Type              Number Range      Filters Based On             Place Near
────────────────  ───────────────   ──────────────────────────   ─────────────
Standard          1–99              Source IP address ONLY       Destination
                  1300–1999
Extended          100–199           Src IP, Dst IP, Protocol,    Source
                  2000–2699         Src Port, Dst Port
Named Standard    Any name          Source IP address ONLY       Destination
Named Extended    Any name          Full L3/L4 criteria          Source
```

---

## 3. Standard ACLs

### What Standard ACLs Can and Cannot Do

A standard ACL is the simplest type. It looks at **only one thing**: the **source IP address** of the packet. It cannot distinguish based on destination, protocol, or port number. This simplicity makes it both easy to configure and limited in flexibility.

```
Standard ACL evaluates:        Standard ACL IGNORES:
  ✓ Source IP address            ✗ Destination IP address
                                 ✗ Protocol (TCP, UDP, ICMP)
                                 ✗ Port numbers
                                 ✗ Direction of traffic
```

### Standard ACL Syntax

```cisco
! Numbered standard ACL — add each ACE one at a time
access-list [1-99 or 1300-1999] {permit | deny} {source} [source-wildcard] [log]

! Special source keywords:
! host 192.168.1.10 = exactly 192.168.1.10 (same as 192.168.1.10 0.0.0.0)
! any              = any source IP (same as 0.0.0.0 255.255.255.255)

! Apply to interface:
interface [type/number]
  ip access-group [ACL-number] {in | out}
```

### Wildcard Masks in Standard ACLs

The wildcard mask tells the router which bits of the source IP **must match** and which bits can be **anything**. A `0` in the wildcard = "this bit must match exactly." A `1` in the wildcard = "this bit can be anything — don't care."

```
Source IP + Wildcard → What it matches
─────────────────────────────────────────────────────────────
192.168.1.0   0.0.0.255   → Matches 192.168.1.0 through 192.168.1.255
                             (any host in the /24 subnet)

192.168.1.10  0.0.0.0     → Matches only exactly 192.168.1.10
                             (same as: host 192.168.1.10)

10.0.0.0      0.255.255.255 → Matches 10.0.0.0 through 10.255.255.255
                               (entire Class A private range)

0.0.0.0       255.255.255.255 → Matches any IP address
                                 (same as: any)

192.168.0.0   0.0.255.255   → Matches 192.168.0.0 through 192.168.255.255
                               (the entire 192.168.x.x range)

Wildcard calculation trick:
  Wildcard = 255.255.255.255 − Subnet Mask
  For /26 (255.255.255.192):
    255.255.255.255 − 255.255.255.192 = 0.0.0.63
  So: network 192.168.1.64 wildcard 0.0.0.63 = 192.168.1.64/26
```

### Standard ACL — Worked Examples

**Example 1: Block one specific host from reaching any destination**

```cisco
! Block PC at 192.168.1.100 from any access
Router(config)# access-list 10 deny host 192.168.1.100
Router(config)# access-list 10 permit any           ! allow everyone else

! Apply outbound on the interface toward the destination network
Router(config)# interface GigabitEthernet0/1
Router(config-if)# ip access-group 10 out

! What happens:
! Packet from 192.168.1.100 → matches "deny host" → BLOCKED
! Packet from 192.168.1.50  → no match on deny, matches "permit any" → ALLOWED
! Packet from 10.0.0.5      → no match on deny, matches "permit any" → ALLOWED
```

**Example 2: Permit only the management subnet to Telnet to routers**

```cisco
! Only 192.168.99.0/24 (management VLAN) can access this router's VTY lines
Router(config)# access-list 15 permit 192.168.99.0 0.0.0.255
! (implicit deny all — everything else is blocked)

! Apply to VTY lines (restricts who can SSH/Telnet)
Router(config)# line vty 0 15
Router(config-line)# access-class 15 in
! Note: VTY uses "access-class", not "ip access-group"
```

**Example 3: Deny a whole subnet, permit everything else**

```cisco
! Block the 10.20.30.0/24 subnet, allow all other traffic
Router(config)# access-list 20 deny 10.20.30.0 0.0.0.255
Router(config)# access-list 20 permit any

! IMPORTANT: Order matters!
! If you wrote "permit any" first, it would match everything
! and the deny would never be reached.
! Always put more specific rules BEFORE general rules.
```

### Standard ACL Verification

```cisco
! View the ACL and its hit counts
Router# show access-lists 10
Standard IP access list 10
    10 deny   host 192.168.1.100 (25 matches)    ← hit counter
    20 permit any (1042 matches)

! View which interfaces have ACLs applied
Router# show ip interface GigabitEthernet0/1
GigabitEthernet0/1 is up, line protocol is up
  Inbound  access list is not set
  Outbound access list is 10                     ← ACL 10 applied outbound

! Clear hit counters (useful for testing)
Router# clear ip access-list counters 10
```

---

## 4. Extended ACLs

### Why Extended ACLs Are More Powerful

While a standard ACL can only ask "where is this packet coming from?", an extended ACL can ask a much richer set of questions: "Where is it coming from, where is it going, what protocol is it using, and what port number?" This granularity lets you write very precise traffic policies.

```
Extended ACL evaluates ALL of these:
  ✓ Source IP address (and wildcard)
  ✓ Destination IP address (and wildcard)
  ✓ IP Protocol (TCP, UDP, ICMP, OSPF, EIGRP, etc.)
  ✓ Source port number (for TCP/UDP)
  ✓ Destination port number (for TCP/UDP)
  ✓ TCP flags (established — for allowing return traffic)
  ✓ ICMP message types
```

### Extended ACL Syntax

```cisco
access-list [100-199] {permit|deny} protocol
            source src-wildcard [operator src-port]
            destination dst-wildcard [operator dst-port]
            [established] [log]

Protocol keywords:
  ip      → matches ANY IP protocol (use when you don't care about L4)
  tcp     → TCP traffic only
  udp     → UDP traffic only
  icmp    → ICMP only (ping, traceroute)
  ospf    → OSPF routing protocol
  eigrp   → EIGRP routing protocol
  47      → GRE tunnels (protocol number)

Operator keywords (for port matching):
  eq [port]          → equal to port (e.g., eq 80 = HTTP)
  neq [port]         → not equal to port
  lt [port]          → less than port number
  gt [port]          → greater than port number
  range [low] [high] → range of ports

Port name shortcuts (Cisco knows these):
  eq www = eq 80
  eq 443 = HTTPS
  eq ftp = eq 21
  eq ssh = eq 22
  eq telnet = eq 23
  eq smtp = eq 25
  eq dns = eq 53
  eq dhcp = eq 67
```

### Extended ACL — Worked Examples

**Example 1: Allow only HTTP and HTTPS from internal network to internet**

```cisco
Router(config)# access-list 110 remark --- Allow Web Traffic from LAN ---
Router(config)# access-list 110 permit tcp 192.168.1.0 0.0.0.255 any eq 80
Router(config)# access-list 110 permit tcp 192.168.1.0 0.0.0.255 any eq 443
Router(config)# access-list 110 remark --- Allow DNS queries ---
Router(config)# access-list 110 permit udp 192.168.1.0 0.0.0.255 any eq 53
Router(config)# access-list 110 remark --- Block everything else ---
Router(config)# access-list 110 deny ip any any log
! "log" keyword → logs denied packets to syslog (useful for monitoring)

! Apply inbound on the LAN interface (close to source — extended ACL rule)
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ip access-group 110 in
```

**Example 2: Block Telnet to all servers, allow everything else**

```cisco
! Block Telnet (TCP port 23) to server subnet 10.1.1.0/24
Router(config)# access-list 120 deny tcp any 10.1.1.0 0.0.0.255 eq 23
Router(config)# access-list 120 permit ip any any     ! allow everything else

Router(config)# interface GigabitEthernet0/0
Router(config-if)# ip access-group 120 in
```

**Example 3: Allow return traffic for established TCP sessions**

This is a very important concept. If you block inbound traffic from the internet, you also need to allow **return traffic** for connections your internal hosts initiated. The `established` keyword does exactly this — it permits TCP packets that have the ACK or RST flag set, which are characteristics of return/existing traffic.

```cisco
! Outbound ACL on internet-facing interface:
! Allow all outbound traffic from internal network
Router(config)# access-list 130 permit ip 192.168.0.0 0.0.255.255 any

! Inbound ACL on internet-facing interface:
! Block new connections from internet, but allow return traffic
Router(config)# access-list 140 permit tcp any 192.168.0.0 0.0.255.255 established
! "established" = ACK or RST bit set → this is return traffic, not new connection
Router(config)# access-list 140 deny ip any any log

Router(config)# interface GigabitEthernet0/1           ! internet-facing interface
Router(config-if)# ip access-group 130 out             ! outbound: allow internal traffic
Router(config-if)# ip access-group 140 in              ! inbound: only allow returns

! Note: "established" only works for TCP, not UDP or ICMP
! For UDP/ICMP return traffic, you need a stateful firewall (not just ACLs)
```

**Example 4: Comprehensive security policy**

```cisco
! Company policy:
! - Finance VLAN (192.168.30.0/24) can ONLY access the database server (10.1.1.5)
! - Finance CANNOT browse internet or access other servers
! - IT VLAN (192.168.40.0/24) can access everything
! - All others can access web (80/443) and DNS (53) only

Router(config)# access-list 150 remark === FINANCE VLAN RULES ===
Router(config)# access-list 150 permit tcp 192.168.30.0 0.0.0.255 host 10.1.1.5 eq 1433
! Allow Finance to SQL Server (port 1433) only
Router(config)# access-list 150 deny ip 192.168.30.0 0.0.0.255 any
! Block Finance from everything else

Router(config)# access-list 150 remark === IT VLAN RULES ===
Router(config)# access-list 150 permit ip 192.168.40.0 0.0.0.255 any
! IT can access everything

Router(config)# access-list 150 remark === GENERAL USERS ===
Router(config)# access-list 150 permit tcp any any eq 80
Router(config)# access-list 150 permit tcp any any eq 443
Router(config)# access-list 150 permit udp any any eq 53
Router(config)# access-list 150 deny ip any any log

Router(config)# interface GigabitEthernet0/0
Router(config-if)# ip access-group 150 in
```

### Protocol Numbers for Extended ACLs

When you need to match a specific IP protocol that doesn't have a keyword, use the protocol number directly:

```
Protocol Name    Number    Keyword Available?
───────────────  ────────  ──────────────────
ICMP             1         Yes (icmp)
TCP              6         Yes (tcp)
UDP              17        Yes (udp)
GRE              47        No  (use: access-list 100 permit 47 ...)
OSPF             89        Yes (ospf)
EIGRP            88        Yes (eigrp)
```

---

## 5. Named ACLs

### Why Named ACLs Are Better

Named ACLs were introduced to solve two major limitations of numbered ACLs. First, a number like "110" tells you nothing about what the ACL does, whereas "BLOCK_TELNET_TO_SERVERS" is self-documenting. Second, and more importantly, with numbered ACLs you **cannot delete individual entries** — you can only delete the entire ACL and rewrite it. Named ACLs let you add, remove, and reorder individual entries using sequence numbers.

```
Numbered ACL problem:
  access-list 110 permit tcp 192.168.1.0 0.0.0.255 any eq 80
  access-list 110 permit tcp 192.168.1.0 0.0.0.255 any eq 443
  access-list 110 deny ip any any

  Want to remove the HTTP rule?
  → Must delete ENTIRE ACL 110 and retype everything!

Named ACL solution:
  ip access-list extended ALLOW_WEB
    10 permit tcp 192.168.1.0 0.0.0.255 any eq 80
    20 permit tcp 192.168.1.0 0.0.0.255 any eq 443
    30 deny ip any any

  Want to remove rule 10?
  → ip access-list extended ALLOW_WEB
  → no 10                           ← just delete that one entry!
  → Done. Rules 20 and 30 unchanged.
```

### Named ACL Configuration

```cisco
! Named Standard ACL
Router(config)# ip access-list standard PERMIT_MGMT_ONLY
Router(config-std-nacl)# 10 permit 192.168.99.0 0.0.0.255    ! sequence 10
Router(config-std-nacl)# 20 deny any log                     ! sequence 20

! Named Extended ACL
Router(config)# ip access-list extended CORPORATE_POLICY
Router(config-ext-nacl)# 10 remark -- Allow Finance to DB Server only --
Router(config-ext-nacl)# 20 permit tcp 192.168.30.0 0.0.0.255 host 10.1.1.5 eq 1433
Router(config-ext-nacl)# 30 deny ip 192.168.30.0 0.0.0.255 any
Router(config-ext-nacl)# 40 remark -- Allow general web access --
Router(config-ext-nacl)# 50 permit tcp any any eq 80
Router(config-ext-nacl)# 60 permit tcp any any eq 443
Router(config-ext-nacl)# 70 permit udp any any eq 53
Router(config-ext-nacl)# 80 deny ip any any log
Router(config-ext-nacl)# exit

! Apply named ACL to interface (same as numbered)
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ip access-group CORPORATE_POLICY in

! Apply named ACL to VTY lines
Router(config)# line vty 0 15
Router(config-line)# access-class PERMIT_MGMT_ONLY in
```

### Editing Named ACLs — The Key Advantage

```cisco
! View current ACL with sequence numbers
Router# show access-lists CORPORATE_POLICY
Extended IP access list CORPORATE_POLICY
    10 remark -- Allow Finance to DB Server only --
    20 permit tcp 192.168.30.0 0.0.0.255 host 10.1.1.5 eq 1433 (42 matches)
    30 deny ip 192.168.30.0 0.0.0.255 any (15 matches)
    40 remark -- Allow general web access --
    50 permit tcp any any eq 80 (1205 matches)
    60 permit tcp any any eq 443 (3421 matches)
    70 permit udp any any eq 53 (89 matches)
    80 deny ip any any log (3 matches)

! Delete a specific entry
Router(config)# ip access-list extended CORPORATE_POLICY
Router(config-ext-nacl)# no 30           ! removes only the "deny Finance" rule

! Insert a new entry BETWEEN existing entries
Router(config-ext-nacl)# 25 permit icmp 192.168.30.0 0.0.0.255 host 10.1.1.5
! Sequence 25 → inserts between 20 and 30

! If you need to resequence (e.g., no room between entries)
Router# ip access-list resequence CORPORATE_POLICY 10 10
! Renumbers starting at 10, incrementing by 10
```

---

## 6. ACL Placement Rules

### The Golden Rule — Why Placement Matters

ACL placement is not just a configuration choice — it has a **major impact on what gets blocked and what doesn't**, and also on **network efficiency** (where traffic gets dropped affects how far unnecessary traffic travels).

The classic rule taught for the CCNA exam is:

> **Standard ACLs → place CLOSE TO THE DESTINATION**  
> **Extended ACLs → place CLOSE TO THE SOURCE**

Let's understand *why* through a detailed example, because understanding the reasoning is far more valuable than just memorizing the rule.

### Why Standard ACLs Go Near the Destination

Standard ACLs match only the **source IP**. If you place them near the source, you might block that source from reaching ALL destinations — even ones you wanted to allow.

```
Network Diagram:
PC-A (192.168.1.10) ──[R1]──[R2]──[Server-1] (10.1.1.1)
                                └──[Server-2] (10.2.2.2)
                                └──[Server-3] (10.3.3.3)

Goal: Block PC-A from reaching Server-2 only. Allow Server-1 and Server-3.

WRONG placement (near source — on R1):
  access-list 10 deny 192.168.1.10  ← deny PC-A
  access-list 10 permit any
  Applied on R1 Gi0/0 inbound

  Effect: PC-A is blocked from Server-1 ✗, Server-2 ✗, Server-3 ✗
  We wanted to block only Server-2, but we blocked EVERYTHING from PC-A!
  Standard ACL cannot specify destination, so it can't distinguish.

CORRECT placement (near destination — on R2 interface toward Server-2):
  access-list 10 deny 192.168.1.10
  access-list 10 permit any
  Applied on R2's interface toward Server-2, outbound

  Effect: PC-A blocked from Server-2 ✗ only
  PC-A still reaches Server-1 ✅ and Server-3 ✅
  Correct behavior!
```

### Why Extended ACLs Go Near the Source

Extended ACLs are specific enough to name both source AND destination, so placing them near the source is safe — you won't accidentally block unwanted traffic. And placing them near the source is more **efficient** because you drop the traffic early, before it wastes bandwidth traveling across the network.

```
Same network: PC-A wants to Telnet (port 23) to Server-2
Goal: Block Telnet from PC-A to Server-2

Extended ACL near source (R1 — CORRECT):
  access-list 110 deny tcp host 192.168.1.10 host 10.2.2.2 eq 23
  access-list 110 permit ip any any
  Applied on R1 Gi0/0 inbound

  Effect:
  - Telnet from PC-A to Server-2 → blocked immediately at R1 ✗
  - HTTP from PC-A to Server-2  → permitted ✅
  - PC-A to Server-1 → permitted ✅
  - Blocked traffic never wastes bandwidth on R1→R2 link!

Extended ACL near destination (R2 — less efficient):
  Same ACL, applied on R2 toward Server-2 outbound
  Effect is same, BUT: blocked packets travel R1→R2 first (wasting bandwidth)
  Then get dropped at R2. Functional but inefficient.
```

### In and Out — Interface Direction

```
Traffic flow relative to the router:

  [PC] ──────────────► [Router] ──────────────► [Server]
         INBOUND                   OUTBOUND
         (IN direction             (OUT direction
         on left interface)        on right interface)

"in"  = packets ENTERING the router on that interface (coming from outside)
"out" = packets LEAVING the router on that interface (going toward outside)

A single interface can have:
  ONE ACL applied inbound (ip access-group X in)
  ONE ACL applied outbound (ip access-group X out)
  Total = max 2 ACLs per interface (one each direction)
```

### ACL Applied to VTY Lines

When you want to restrict who can SSH or Telnet INTO the router itself (not traffic passing through it), apply the ACL to the VTY lines using `access-class` instead of `ip access-group`:

```cisco
! Only management subnet can manage this router
Router(config)# access-list 99 permit 192.168.99.0 0.0.0.255

Router(config)# line vty 0 15
Router(config-line)# access-class 99 in     ! "access-class" for VTY, not "access-group"
Router(config-line)# transport input ssh    ! also restrict to SSH only
Router(config-line)# login local
```

---

## 7. ACL Best Practices & Troubleshooting

### Best Practices

**Order your ACEs from most specific to least specific.** Since processing stops at the first match, more specific rules must come before general ones. If you put `permit any` before `deny host 192.168.1.100`, the deny will never be reached.

**Always include a comment (remark) for each section.** Future engineers (including yourself) will thank you. Use `remark` statements to document your intent.

**Use named ACLs instead of numbered.** They are more readable and allow individual entry deletion.

**Minimize the number of ACEs.** Each packet is checked against every ACE until a match is found. A very long ACL degrades performance on routers that process ACLs in software.

**Never forget the implicit deny.** If you want to log what's being dropped, add an explicit `deny ip any any log` as the last entry before the implicit deny takes over.

**Test before applying in production.** Use `ping` and `traceroute` to verify behavior, and check the hit counters with `show access-lists` after testing.

### Common ACL Mistakes

```
MISTAKE 1: Wrong order of rules
  access-list 10 permit any          ← matches EVERYTHING first!
  access-list 10 deny host 10.0.0.5  ← never reached!

  Fix: Most specific rules first
  access-list 10 deny host 10.0.0.5
  access-list 10 permit any

MISTAKE 2: Applied in wrong direction
  Wanting to filter inbound but applying "out"
  Or filtering outbound but applying "in"
  Always think: which direction does the packet travel on THIS interface?

MISTAKE 3: Standard ACL near source (blocks too much)
  Already covered — always place near destination

MISTAKE 4: Missing "permit any" at end
  Every packet gets hit by implicit deny
  Network goes dark!
  Fix: Always add "permit ip any any" or "permit any" at end

MISTAKE 5: Modifying numbered ACL entry (can't!)
  You CAN'T edit a single numbered ACL entry
  Must delete entire ACL and retype
  Fix: Use named ACLs from the start

MISTAKE 6: Forgetting "established" for return TCP traffic
  Block all inbound → block return traffic too → one-way communication
  Fix: permit tcp any [internal-net] established
```

### ACL Troubleshooting Steps

```cisco
! Step 1: View the ACL and check hit counters
Router# show access-lists
Router# show access-lists CORPORATE_POLICY

! Step 2: Check which interface has the ACL applied and in what direction
Router# show ip interface GigabitEthernet0/0
  Inbound  access list is CORPORATE_POLICY
  Outbound access list is not set

! Step 3: Clear counters, run test traffic, check which rule matched
Router# clear ip access-list counters CORPORATE_POLICY
! ... run ping or test traffic ...
Router# show access-lists CORPORATE_POLICY
! Check which entries show matches now

! Step 4: Temporarily remove ACL to confirm it is the cause
Router(config-if)# no ip access-group CORPORATE_POLICY in
! If traffic flows now → ACL is the problem → review rules

! Step 5: Debug (use with extreme caution in production)
Router# debug ip packet
Router# debug ip packet detail
Router# undebug all    ! ALWAYS turn off debug immediately after!
```

---

## 8. AAA — Authentication, Authorization, Accounting

### Understanding AAA

AAA is a security framework that controls **who can access the network**, **what they can do once inside**, and **what they did while connected**. Every enterprise network uses AAA for managing both network device access (router/switch CLI) and network access (connecting to Wi-Fi, VPN).

**Authentication** answers: *"Are you who you claim to be?"* It verifies identity using credentials — username/password, certificates, smart cards, or biometrics.

**Authorization** answers: *"What are you allowed to do?"* Once authenticated, authorization determines privileges — can this person run `show` commands only, or also `configure terminal`?

**Accounting** answers: *"What did you do and when?"* It logs all activity — login time, commands executed, session duration. This provides auditing and non-repudiation.

```
Without AAA:
  Line vty 0 15
    password cisco       ← everyone shares same password
    login                ← no individual accountability
    transport input ssh

With AAA:
  username admin privilege 15 secret Admin@Pass
  username auditor privilege 5 secret Audit@Pass
  aaa new-model
  aaa authentication login default group tacacs+ local
  aaa authorization exec default group tacacs+ local
  aaa accounting exec default start-stop group tacacs+
  
  Now: each user has own credentials, own privilege level,
  and all actions are logged to TACACS+ server
```

### RADIUS vs TACACS+

These are the two protocols used to communicate between a network device (the **NAS — Network Access Server**) and the AAA server. They serve similar purposes but have important differences that the CCNA exam tests.

```
Feature               RADIUS                    TACACS+
────────────────────  ────────────────────────  ────────────────────────────
Standard              Open (IETF RFC 2865)       Cisco proprietary
Transport             UDP                        TCP
Auth/Acct Ports       UDP 1812 (auth)            TCP 49
                      UDP 1813 (acct)
Encryption            Password only              Full packet encrypted
AAA separation        A+A combined in one resp.  A, A, A are separate
                      (Authorization included    (full flexibility)
                       in auth response)
Command authorization NOT supported              YES — per-command control
Best suited for       Network ACCESS             Device ADMINISTRATION
                      (Wi-Fi, VPN, 802.1X)       (router/switch CLI)
Vendor support        Universal                  Primarily Cisco
```

The key insight is this: **TACACS+ encrypts the entire packet**, whereas RADIUS only encrypts the password. This makes TACACS+ more secure for device administration where commands themselves are sensitive. Also, TACACS+ supports **per-command authorization** — you can define exactly which IOS commands each user is allowed to run.

### AAA Configuration on Cisco IOS

```cisco
!════════ STEP 1: Enable AAA ════════!
Router(config)# aaa new-model          ! enables AAA — disables old "login" commands

!════════ STEP 2: Define AAA Servers ════════!

! TACACS+ server (for device admin)
Router(config)# tacacs server TACACS_SERVER_1
Router(config-server-tacacs)# address ipv4 10.0.0.50
Router(config-server-tacacs)# key TacacsSecretKey@2024
Router(config-server-tacacs)# port 49                    ! default — can omit

! RADIUS server (for network access)
Router(config)# radius server RADIUS_SERVER_1
Router(config-radius-server)# address ipv4 10.0.0.51 auth-port 1812 acct-port 1813
Router(config-radius-server)# key RadiusSecretKey@2024

!════════ STEP 3: Create Server Groups ════════!
Router(config)# aaa group server tacacs+ TACACS_GROUP
Router(config-sg-tacacs+)# server name TACACS_SERVER_1

Router(config)# aaa group server radius RADIUS_GROUP
Router(config-sg-radius)# server name RADIUS_SERVER_1

!════════ STEP 4: Define Authentication Methods ════════!
! Method list: try TACACS+ first, fall back to local if server unreachable
Router(config)# aaa authentication login default group TACACS_GROUP local
!                                          ↑ "default" applies to all lines
!                              Try server first ↑           ↑ fallback to local users

! For console line — use local only (no server dependency at console)
Router(config)# aaa authentication login CONSOLE_AUTH local
Router(config)# line console 0
Router(config-line)# login authentication CONSOLE_AUTH

!════════ STEP 5: Define Authorization ════════!
Router(config)# aaa authorization exec default group TACACS_GROUP local
! Controls privilege level and exec shell access

Router(config)# aaa authorization commands 15 default group TACACS_GROUP local
! Controls which privilege-15 commands are allowed (per-command auth)

!════════ STEP 6: Define Accounting ════════!
Router(config)# aaa accounting exec default start-stop group TACACS_GROUP
! Logs when exec sessions start and stop

Router(config)# aaa accounting commands 15 default start-stop group TACACS_GROUP
! Logs every privilege-15 command entered

!════════ STEP 7: Apply to Lines ════════!
Router(config)# line vty 0 15
Router(config-line)# login authentication default   ! use "default" method list
Router(config-line)# transport input ssh
Router(config-line)# exec-timeout 10 0

!════════ VERIFY ════════!
Router# show aaa servers
Router# show tacacs
Router# show radius statistics
Router# test aaa group TACACS_GROUP username admin password Admin@Pass legacy
```

### Privilege Levels

Cisco IOS has 16 privilege levels (0–15). By default, level 1 = user EXEC (limited), level 15 = privileged EXEC (full access). AAA authorization can assign any level to any user.

```
Level 0  → Minimal commands: logout, enable, disable, help, exit
Level 1  → User EXEC: show commands (limited), ping, traceroute
Level 2-14 → Customizable intermediate levels
Level 15 → Privileged EXEC: ALL commands including configure terminal

Router(config)# username netops privilege 7 secret NetOps@Pass
! User "netops" gets level 7 — configure which commands are in level 7

Router(config)# privilege exec level 7 show running-config
! Allows level 7 users to run "show running-config"
Router(config)# privilege configure level 7 interface
! Allows level 7 to enter interface configuration mode
```

---

## 9. Port Security

### The Problem Port Security Solves

Without any restriction on a switch port, any device can be physically plugged in and immediately join the network. An attacker could walk up to an office, plug into an empty network port, and have full LAN access. Port security limits which MAC addresses (and how many) can communicate on each switch port.

### How Port Security Works

```
Port Security monitors the SOURCE MAC ADDRESS of every frame
that arrives on a secured port.

If the source MAC is:
  → In the allowed list AND count ≤ maximum:
    Frame forwarded normally ✅

  → NOT in allowed list OR count > maximum:
    VIOLATION! Action depends on configured mode.

The allowed MAC list is built in three ways:
  1. Static   → Administrator manually types the MACs
  2. Dynamic  → Switch learns MACs automatically (lost on reboot)
  3. Sticky   → Switch learns MACs AND saves them to running-config
                 (persists across reboots if you save config)
```

### Complete Port Security Configuration

```cisco
! PREREQUISITES: Port must be in access mode first
Switch(config)# interface GigabitEthernet0/5
Switch(config-if)# description "Sales-PC-Reception"
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10

! ENABLE port security
Switch(config-if)# switchport port-security

! SET maximum allowed MAC addresses (default = 1)
Switch(config-if)# switchport port-security maximum 2
! Only 2 devices can connect to this port simultaneously

! CONFIGURE how MACs are learned (choose one method):

! Method A — Sticky (auto-learn and save to config — most common)
Switch(config-if)# switchport port-security mac-address sticky
! First device(s) that connect get saved permanently

! Method B — Static (manually specify exact MAC address)
Switch(config-if)# switchport port-security mac-address 00AA.BBCC.DD01
Switch(config-if)# switchport port-security mac-address 00AA.BBCC.DD02
! Only these exact MACs are permitted — nobody else gets through

! Method C — Dynamic (auto-learned, cleared on reboot — least secure)
! No extra command needed — default behavior without sticky

! SET violation action (what happens on violation):
Switch(config-if)# switchport port-security violation shutdown
! Options: shutdown (default) | restrict | protect

! OPTIONAL: Configure aging so MACs eventually expire and can be relearned
Switch(config-if)# switchport port-security aging time 60    ! 60 minutes
Switch(config-if)# switchport port-security aging type inactivity
! Aging restarts every time the MAC is seen — expires after 60 min of silence

Switch(config-if)# no shutdown
```

### Violation Modes — Detailed Comparison

```
SHUTDOWN (default — strictest):
  What triggers it: Any unauthorized MAC OR max count exceeded
  What happens:
    → Port immediately goes ERR-DISABLED (completely shuts down)
    → ALL traffic stops (even legitimate traffic)
    → Syslog message generated
    → SNMP trap sent to NMS
    → Violation counter incremented
  Recovery: Manual (shutdown + no shutdown) or auto-recovery timer
  Best for: High-security environments where any violation = suspicious

RESTRICT (moderate):
  What triggers it: Unauthorized MAC OR max count exceeded
  What happens:
    → ONLY the violating frames are dropped
    → Legitimate traffic from allowed MACs continues ✅
    → Syslog message generated
    → SNMP trap sent
    → Violation counter incremented
  Recovery: Not needed — port stays up
  Best for: Environments where you want alerting without disrupting service

PROTECT (weakest):
  What triggers it: Unauthorized MAC OR max count exceeded
  What happens:
    → ONLY the violating frames are dropped silently
    → Legitimate traffic continues ✅
    → NO log message (completely silent)
    → NO SNMP trap
    → Violation counter NOT incremented
  Recovery: Not needed
  Best for: Rarely used — hard to know violations are happening
```

### Recovering from Err-Disabled

```cisco
! Check which ports are err-disabled and why
Switch# show interfaces status err-disabled
Switch# show errdisable reason
Switch# show port-security interface GigabitEthernet0/5

! Method 1: Manual recovery (most controlled)
Switch(config)# interface GigabitEthernet0/5
Switch(config-if)# shutdown
Switch(config-if)# no shutdown    ! clears err-disabled, port comes back up

! Method 2: Automatic recovery (convenient but less secure)
Switch(config)# errdisable recovery cause psecure-violation
Switch(config)# errdisable recovery interval 300   ! auto-recover after 5 minutes
! Port will re-enable itself after 300 seconds
! If violation happens again, port shuts again

! Verify port security status
Switch# show port-security
Switch# show port-security interface GigabitEthernet0/5
Switch# show port-security address    ! all sticky/static MACs learned
```

---

## 10. DHCP Snooping

### The Threat: Rogue DHCP Server

DHCP works by broadcast — a client sends a DHCP Discover to 255.255.255.255, and the first DHCP server to respond wins. If an attacker connects their own laptop running a DHCP server, **their server might respond before the legitimate one**. The attacker can then give clients a fake default gateway (pointing to the attacker's machine), effectively performing a man-in-the-middle attack on every device that gets an IP from the rogue server.

```
WITHOUT DHCP Snooping:

Legitimate DHCP Server ←─────────────────────────────┐
                                                       │
[PC] → DHCP Discover (broadcast)                      │
       ↓ goes everywhere                               │
       ↓                                               │
[Rogue Server] → DHCP Offer: IP=192.168.1.50, GW=192.168.1.99 (attacker's IP!)
[Legit Server] → DHCP Offer: IP=192.168.1.50, GW=192.168.1.1 (real GW)

PC accepts whichever arrives first!
If rogue wins → PC sends all traffic to attacker → MITM!

WITH DHCP Snooping:
Switch acts as firewall between clients and DHCP servers.
Only trusted ports can send DHCP server messages (Offer, ACK).
Untrusted ports (user ports) can only send client messages (Discover, Request).
Rogue server on untrusted port → Offer immediately dropped!
```

### DHCP Snooping Binding Table

When DHCP Snooping is enabled, the switch builds a **binding table** that maps each client's MAC address, IP address, VLAN, and port. This table is used by Dynamic ARP Inspection (Section 11) to validate ARP traffic.

```
Switch# show ip dhcp snooping binding

MacAddress         IpAddress       Lease(sec)  Type           VLAN  Interface
─────────────────  ──────────────  ──────────  ─────────────  ────  ──────────────
00:50:79:66:68:00  192.168.10.50   86400       dhcp-snooping  10    GigEthernet0/1
00:50:79:66:68:01  192.168.10.51   86400       dhcp-snooping  10    GigEthernet0/2
00:AA:BB:CC:DD:EE  192.168.20.100  86400       dhcp-snooping  20    GigEthernet0/5
```

### DHCP Snooping Configuration

```cisco
!════════ GLOBAL ENABLE ════════!
Switch(config)# ip dhcp snooping
Switch(config)# ip dhcp snooping vlan 10,20,30,99
! Must specify VLANs — not enabled globally on all VLANs by default

! Disable Option 82 insertion (optional — prevents issues with some servers)
Switch(config)# no ip dhcp snooping information option
! Option 82 adds switch/port info to DHCP packets — some servers reject it

!════════ TRUST CONFIGURATION ════════!
! Mark uplink to DHCP server or upstream switch as TRUSTED
! Only trusted ports can receive DHCP server messages (Offer, ACK, NAK)

Switch(config)# interface GigabitEthernet0/24   ! uplink to core switch / DHCP server
Switch(config-if)# ip dhcp snooping trust
Switch(config-if)# exit

! All OTHER ports are UNTRUSTED by default — no extra config needed
! Untrusted ports: client messages (Discover, Request) → allowed
!                  server messages (Offer, ACK) → DROPPED!

!════════ RATE LIMITING (optional but recommended) ════════!
! Limit DHCP packets per second on untrusted ports (prevents DHCP starvation)
Switch(config)# interface range GigabitEthernet0/1 - 20
Switch(config-if-range)# ip dhcp snooping limit rate 15
! Max 15 DHCP packets per second — excessive = port err-disabled (DHCP starvation attack)

!════════ VERIFICATION ════════!
Switch# show ip dhcp snooping
DHCP snooping is configured on following VLANs:
  10,20,30,99
Insertion of option 82 is disabled
Interface           Trusted    Allow option    Rate limit (pps)
──────────────────  ─────────  ──────────────  ────────────────
GigabitEthernet0/24 yes        yes             unlimited
GigabitEthernet0/1  no         no              15
GigabitEthernet0/2  no         no              15

Switch# show ip dhcp snooping binding
Switch# show ip dhcp snooping statistics
```

---

## 11. Dynamic ARP Inspection (DAI)

### The Threat: ARP Poisoning / Spoofing

ARP has a fundamental design flaw: it is completely unauthenticated. Any device can send an ARP reply claiming any IP address belongs to its MAC, without any verification. An attacker can send **gratuitous ARP replies** (unsolicited ARP messages) that say "192.168.1.1 is at MY MAC" to all devices on the subnet. Victims update their ARP tables and start sending traffic — including default gateway traffic — to the attacker's machine. The attacker forwards it onward (so the victim doesn't notice) while capturing everything in between.

```
ARP POISONING ATTACK:

Normal ARP table on PC-A:
  192.168.1.1 → 00:aa:bb:cc:dd:01  (real gateway MAC)

Attacker sends gratuitous ARP: "192.168.1.1 is at 00:EE:FF:00:11:22" (attacker's MAC)

Poisoned ARP table on PC-A:
  192.168.1.1 → 00:EE:FF:00:11:22  (ATTACKER'S MAC — wrong!)

Now PC-A sends ALL traffic (including login credentials!) to attacker.
Attacker forwards to real gateway → PC-A never notices anything is wrong.
This is a classic Man-in-the-Middle attack.
```

### How DAI Protects Against This

DAI intercepts all ARP packets on **untrusted ports** and validates them against the **DHCP Snooping binding table**. If the ARP reply says "IP X is at MAC Y" but the binding table shows "IP X should be at MAC Z on port A", DAI drops the fake ARP and logs it.

```
DAI Validation:
  ARP Reply arrives: "192.168.1.50 is at MAC 00:EE:FF:00:11:22"
                                           ↑ ATTACKER'S MAC
  DHCP Snooping binding table says:
    192.168.1.50 → MAC 00:50:79:66:68:00, Port Gi0/1

  DAI comparison:
    ARP says:    192.168.1.50 → 00:EE:FF:00:11:22
    Binding says: 192.168.1.50 → 00:50:79:66:68:00
    MISMATCH! → ARP packet DROPPED → attacker's poisoning fails!
```

### DAI Configuration

```cisco
! PREREQUISITE: DHCP Snooping must be configured first (DAI uses its binding table)
! (See Section 10 for DHCP Snooping config)

!════════ ENABLE DAI ════════!
Switch(config)# ip arp inspection vlan 10,20,30
! Enable DAI on specific VLANs

!════════ TRUST CONFIGURATION ════════!
! Mark same ports trusted as DHCP Snooping (uplinks to switches/routers)
Switch(config)# interface GigabitEthernet0/24
Switch(config-if)# ip arp inspection trust
! Trusted ports: ARP packets NOT validated (assumed already secured)
! Untrusted ports: ALL ARP packets validated against binding table

!════════ ADDITIONAL VALIDATION (optional) ════════!
! Check that ARP packet fields are consistent:
Switch(config)# ip arp inspection validate src-mac dst-mac ip
! src-mac: ARP sender MAC = Ethernet frame src MAC
! dst-mac: ARP target MAC = Ethernet frame dst MAC (for replies)
! ip:      ARP sender IP is not 0.0.0.0 or broadcast

!════════ RATE LIMITING (optional) ════════!
! Limit ARP packets on untrusted ports (prevent ARP flooding)
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)# ip arp inspection limit rate 100   ! max 100 ARP/sec

!════════ STATIC ARP ACL (for devices with static IPs — no DHCP binding) ════════!
! Devices with static IPs won't have DHCP Snooping bindings
! Create a static ARP ACL to manually define their mappings
Switch(config)# arp access-list STATIC_DEVICES
Switch(config-arp-nacl)# permit ip host 192.168.99.1 mac host 00aa.bbcc.dd01
! Router gateway has static IP → manually permit its ARP

Switch(config)# ip arp inspection filter STATIC_DEVICES vlan 99

!════════ VERIFICATION ════════!
Switch# show ip arp inspection
Switch# show ip arp inspection vlan 10
Switch# show ip arp inspection statistics
Switch# show ip arp inspection statistics vlan 10
```

### DAI + DHCP Snooping: Working Together

```
These two features are designed to complement each other:

DHCP Snooping:
  → Watches DHCP traffic
  → Builds binding table (MAC → IP → Port → VLAN)
  → Prevents rogue DHCP servers

DAI:
  → Uses DHCP Snooping binding table
  → Validates ARP traffic against that table
  → Prevents ARP poisoning/spoofing

Together they form a strong Layer 2 security foundation:
  Rogue DHCP server?  → DHCP Snooping blocks it
  ARP poisoning?      → DAI validates and drops it
  MAC flooding?       → Port Security limits MACs per port
```

---

## 12. Layer 2 Security Threats & Mitigations

### Complete L2 Threat Reference

```
THREAT: VLAN Hopping
  How: Attacker sends double-tagged 802.1Q frames to access VLANs
       they shouldn't reach. Uses DTP to negotiate a trunk.
  Defense:
    → Disable DTP: switchport nonegotiate
    → Change native VLAN to unused VLAN: switchport trunk native vlan 999
    → Explicitly set all ports as access or trunk: switchport mode access/trunk
    → Disable unused ports and put in black hole VLAN

THREAT: MAC Flooding (CAM Table Overflow)
  How: Attacker sends thousands of frames with random source MACs
       CAM table fills up → switch floods all traffic → attacker sniffs
  Defense:
    → Port Security: limit MACs per port
    → switchport port-security maximum 2 (or appropriate number)
    → violation shutdown: attacker's port gets err-disabled

THREAT: DHCP Starvation
  How: Attacker sends many DHCP Discovers with spoofed MACs
       Exhausts DHCP pool → legitimate clients can't get IPs
  Defense:
    → DHCP Snooping rate limiting: ip dhcp snooping limit rate 15
    → Port Security: limits unique MACs, limiting spoofed requests

THREAT: STP Manipulation
  How: Attacker connects switch with lower bridge priority
       Wins Root Bridge election → controls STP topology
       Now root of network → all traffic flows through attacker
  Defense:
    → BPDU Guard on access ports: spanning-tree bpduguard enable
    → Root Guard on uplinks: spanning-tree guard root
    → PortFast default: spanning-tree portfast default

THREAT: CDP/LLDP Information Disclosure
  How: CDP reveals router model, IOS version, IP addresses
       Attacker uses this for targeted exploits
  Defense:
    → Disable CDP on user-facing ports: no cdp enable (per interface)
    → Or globally: no cdp run (disables CDP completely)
    → Use LLDP selectively if needed for VoIP phone discovery
```

---

## 13. VPN Fundamentals

### What Is a VPN and Why Is It Needed?

A VPN creates a **secure, encrypted tunnel** over an untrusted network (typically the internet). Without a VPN, traffic sent over the internet is plaintext and can be read by anyone in the path — ISPs, governments, attackers. A VPN encrypts the data so that even if captured, it cannot be read.

```
WITHOUT VPN (traffic on internet = readable):
Branch ──[Router]── Internet ────────────── HQ
        Username: admin
        Password: cisco123         ← visible to anyone on the path!

WITH Site-to-Site IPSec VPN:
Branch ──[Router]══[ENCRYPTED TUNNEL]══[Router]── HQ
        €#@!$%^&*()_+{}|<>?   ← unreadable ciphertext to outsiders
        Only HQ router can decrypt it
```

### Site-to-Site VPN vs Remote Access VPN

```
SITE-TO-SITE VPN:
  Connects entire networks (offices) together
  Encryption/decryption done by ROUTERS (not end devices)
  Users are unaware of the VPN — works transparently
  Uses: Connecting branch office to HQ
  
  [Branch LAN] ──[Branch Router]══IPSec VPN══[HQ Router]── [HQ LAN]
                    encrypts                    decrypts

REMOTE ACCESS VPN:
  Connects INDIVIDUAL users to corporate network from anywhere
  VPN CLIENT software runs on user's laptop/phone
  User must deliberately connect the VPN
  Uses: Work-from-home, traveling employees
  
  [Home PC]──VPN Client──Internet──[VPN Gateway/ASA]──[Corp Network]
  Cisco AnyConnect is the common client software
```

### IPSec VPN — How It Works

IPSec is the industry-standard protocol suite for VPN encryption. It operates in two phases:

**Phase 1 — IKE (Internet Key Exchange):** The two VPN endpoints authenticate each other and negotiate encryption parameters. They establish a secure channel called an ISAKMP Security Association (SA). Authentication uses either a **pre-shared key (PSK)** — a password both sides know — or **digital certificates** (more secure, used in enterprise).

**Phase 2 — IPSec SA:** Using the Phase 1 secure channel, the routers negotiate the actual IPSec tunnel parameters (encryption algorithm, hashing, lifetime). This creates the IPSec SAs used to encrypt the actual data traffic.

```
VPN Components (what you negotiate):
  Encryption:     AES-256 (strong), AES-128, 3DES (legacy)
  Integrity:      SHA-256, SHA-384, MD5 (legacy — avoid)
  Authentication: Pre-shared key OR RSA certificates
  DH Group:       DH 14 (2048-bit), DH 19 (256-bit EC) — key exchange

IPSec Modes:
  Transport Mode: Encrypts PAYLOAD only, original IP header preserved
                  Used for host-to-host encryption
  Tunnel Mode:    Encrypts ENTIRE original packet (headers + data)
                  New IP header added (from VPN gateway to VPN gateway)
                  Used for site-to-site VPN — most common
```

```cisco
! Basic Site-to-Site IPSec VPN on Cisco IOS

! PHASE 1 — ISAKMP Policy
crypto isakmp policy 10
  encryption aes 256
  hash sha256
  authentication pre-share
  group 14              ! Diffie-Hellman group 14 (2048-bit)
  lifetime 86400        ! 24 hours

! Pre-shared key for the remote peer
crypto isakmp key VPN_Secret_Key@2024 address 203.0.113.2    ! remote router's public IP

! PHASE 2 — IPSec Transform Set
crypto ipsec transform-set TSET esp-aes 256 esp-sha256-hmac
  mode tunnel           ! encrypt entire packet (site-to-site)

! Define interesting traffic (what to encrypt)
ip access-list extended VPN_TRAFFIC
  permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255
  ! Branch LAN ↔ HQ LAN traffic gets encrypted

! Crypto Map (ties everything together)
crypto map VPN_MAP 10 ipsec-isakmp
  set peer 203.0.113.2                   ! remote VPN endpoint IP
  set transform-set TSET
  match address VPN_TRAFFIC              ! what traffic to encrypt

! Apply to WAN interface
interface GigabitEthernet0/1
  crypto map VPN_MAP

! Verify
Router# show crypto isakmp sa          ! Phase 1 status
Router# show crypto ipsec sa           ! Phase 2 status, packet counters
Router# show crypto map
```

---

## 14. Wireless Security

### Why Wireless Needs Extra Security

Wired networks require physical access — an attacker must plug a cable in. Wireless networks broadcast radio signals that pass through walls and extend beyond your building. Anyone within range can attempt to connect. This makes wireless fundamentally more exposed than wired, requiring stronger authentication and encryption.

### Wireless Encryption Evolution

```
WEP (Wired Equivalent Privacy) — 1997:
  Encryption: RC4 with 40-bit or 104-bit keys
  Authentication: Open or Shared Key
  STATUS: COMPLETELY BROKEN ⛔
  Cracked in minutes with freely available tools
  NEVER USE WEP for anything

WPA (Wi-Fi Protected Access) — 2003:
  Encryption: TKIP (Temporal Key Integrity Protocol) — RC4 based
  Authentication: PSK (personal) or 802.1X (enterprise)
  STATUS: DEPRECATED ⚠️
  Better than WEP but TKIP has weaknesses
  Avoid in new deployments

WPA2 (Wi-Fi Protected Access 2) — 2004:
  Encryption: AES-CCMP (AES with 128-bit keys)
  Authentication: PSK (personal) or 802.1X (enterprise)
  STATUS: CURRENT MINIMUM STANDARD ✅
  AES-CCMP is strong — still considered secure
  Use WPA2 at minimum in all deployments

WPA3 (Wi-Fi Protected Access 3) — 2018:
  Encryption: AES-GCMP (stronger than CCMP)
  Authentication: SAE (Simultaneous Authentication of Equals) or 802.1X
  STATUS: RECOMMENDED ✅✅
  SAE replaces PSK — resistant to offline dictionary attacks
  OWE (Opportunistic Wireless Encryption) for open networks
  PMF (Protected Management Frames) mandatory
  Use wherever hardware supports it
```

### Personal vs Enterprise Authentication

```
WPA2-PERSONAL (Pre-Shared Key / PSK):
  All users share the same Wi-Fi password
  Simple to set up — suitable for homes and small offices
  
  Problems:
  → If one employee leaves, password must change for everyone
  → No per-user identity — can't track who did what
  → Password shared widely → higher compromise risk
  → Vulnerable to offline dictionary attacks (attacker captures
     the 4-way handshake and brute-forces the password offline)

WPA2-ENTERPRISE (802.1X / EAP):
  Each user has their own unique credentials (Active Directory login)
  Authentication via RADIUS server
  
  How it works:
  [Laptop] ──────────────────────────────────────────────────────►
            Sends credentials (EAP)
          ◄──────────────────────────────────────────────────────
            AP forwards to RADIUS server, server says allow/deny
  
  Benefits:
  → Per-user credentials → full accountability
  → Remove one user → just disable their account → no password change
  → Can enforce certificate-based auth (no password = more secure)
  → Used by universities, enterprises, large organizations

EAP Methods (within Enterprise):
  PEAP (Protected EAP):
    Certificate on SERVER only, password from client
    Most common — works with Windows Active Directory natively
  
  EAP-TLS:
    Certificate on BOTH server AND client
    Most secure — but requires certificate distribution to every device
    Used in high-security environments
  
  EAP-FAST (Cisco):
    No certificates needed initially
    Uses Protected Access Credentials (PAC)
    Cisco-specific — simpler to deploy than EAP-TLS
```

---

## 15. Key Takeaways & Exam Tips

### Key Takeaways

Every ACL has an **implicit deny all** at the end — always account for traffic you want to permit. Standard ACLs match source IP only and belong near the destination; extended ACLs match src+dst+protocol+port and belong near the source. Named ACLs allow per-entry deletion using sequence numbers, making them far more manageable than numbered ACLs.

**TACACS+** uses TCP port 49 and encrypts the full packet — preferred for device administration because it supports per-command authorization. **RADIUS** uses UDP 1812/1813 and encrypts only the password — preferred for network access (Wi-Fi, VPN, 802.1X).

Port security limits MAC addresses per port. The **shutdown** violation mode err-disables the port; **restrict** drops and logs; **protect** drops silently. **DHCP Snooping** marks uplink ports trusted and drops DHCP server messages from untrusted ports, preventing rogue DHCP servers. **DAI** uses the DHCP Snooping binding table to validate ARP packets, preventing ARP poisoning — these two features are designed to work together.

**WEP is broken** and should never be used. **WPA2 is the minimum** acceptable standard. **WPA3 is recommended** wherever supported. Enterprise (802.1X) authentication provides per-user credentials and accountability, unlike PSK which everyone shares.

### Exam Tips

> 🎯 **Most Tested Topics:**

1. **"Where should a standard ACL be placed?"** → **Close to the destination** (matches source only — placing near source blocks too broadly)
2. **"Where should an extended ACL be placed?"** → **Close to the source** (specific enough to be safe, drops traffic early)
3. **"What is at the end of every ACL?"** → **Implicit deny all** — invisible but always present
4. **"Can you delete one entry from a numbered ACL?"** → **No** — must delete and retype entire ACL. Use named ACLs to delete individual entries.
5. **"What port does TACACS+ use?"** → **TCP 49**
6. **"What ports does RADIUS use?"** → **UDP 1812** (authentication) and **UDP 1813** (accounting)
7. **"Which is preferred for device administration — RADIUS or TACACS+?"** → **TACACS+** (full packet encryption, separates A/A/A, supports per-command authorization)
8. **"What does `access-class` do vs `ip access-group`?"** → `access-class` restricts who can access VTY lines (Telnet/SSH to the router itself); `ip access-group` filters transit traffic on interfaces
9. **"What port security violation mode err-disables the port?"** → **shutdown** (the default)
10. **"What port security violation mode drops silently with no log?"** → **protect**
11. **"What does DHCP Snooping trust mean?"** → Trusted port can receive DHCP **server** messages (Offer, ACK). Untrusted ports only allow client messages (Discover, Request).
12. **"What does DAI require to be configured first?"** → **DHCP Snooping** — DAI uses its binding table for validation
13. **"Which wireless encryption is broken and should never be used?"** → **WEP**
14. **"What is the minimum acceptable wireless security?"** → **WPA2**
15. **"What `established` keyword does in an extended ACL?"** → Permits TCP packets with ACK or RST flag set — used to allow return traffic for TCP sessions initiated from inside

---

*← Previous: [07 — Routing Fundamentals](./07_Routing_Fundamentals.md)*  
*Next: [09 — WAN & SD-WAN →](./09_WAN_SDWAN.md)*
