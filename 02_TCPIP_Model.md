# 02 — TCP/IP Model: The Real-World Networking Stack

> **CCNA Exam Domain:** 1.0 Network Fundamentals  
> **Exam Weight:** ~20% of total exam  
> **Difficulty:** ⭐⭐☆☆☆

---

## Table of Contents
- [What is the TCP/IP Model?](#what-is-the-tcpip-model)
- [TCP/IP vs OSI — Side-by-Side](#tcpip-vs-osi--side-by-side)
- [The 4 Layers of TCP/IP](#the-4-layers-of-tcpip)
- [Key Protocols at Each Layer](#key-protocols-at-each-layer)
- [How Data Flows Through TCP/IP](#how-data-flows-through-tcpip)
- [Important Protocols Deep Dive](#important-protocols-deep-dive)
- [Key Takeaways](#key-takeaways)
- [Exam Tips](#exam-tips)

---

## What is the TCP/IP Model?

The **TCP/IP model** (also called the **Internet Model** or **DoD Model**) is the **practical, real-world framework** that the modern internet is built on. Unlike the OSI model, which is purely conceptual, **TCP/IP is actually implemented** in every device connected to a network.

It was developed by the **U.S. Department of Defense (DoD)** in the 1970s through ARPANET research and became the standard for all internet communications.

### OSI vs TCP/IP — Which Is More Important?

| Aspect | OSI | TCP/IP |
|--------|-----|--------|
| Purpose | Conceptual reference model | Practical implementation model |
| Layers | 7 | 4 |
| Used in real networking? | No (reference only) | Yes (actual implementation) |
| Exam relevance | Very high | Very high |

> 💡 **Key Insight:** The CCNA exam expects you to know **both** models and how they map to each other. When engineers troubleshoot "at Layer X," they're referring to the **OSI** numbering, but the protocols they use come from the **TCP/IP** stack.

---

## TCP/IP vs OSI — Side-by-Side

```
OSI Model                   TCP/IP Model             Common Protocols
=========================   ===================      ==========================
+---------------------+     +---------------+
|  7 - Application    |     |               |        HTTP, HTTPS, FTP, SMTP,
|  6 - Presentation   |     |  Application  |        DNS, DHCP, SSH, Telnet,
|  5 - Session        |     |               |        SNMP, NTP, POP3, IMAP
+---------------------+     +---------------+
|  4 - Transport      |     |  Transport    |        TCP, UDP
+---------------------+     +---------------+
|  3 - Network        |     |  Internet     |        IPv4, IPv6, ICMP, ARP
+---------------------+     +---------------+
|  2 - Data Link      |     |               |        Ethernet, Wi-Fi (802.11),
|  1 - Physical       |     | Network Access|        PPP, HDLC, DSL, Cable
+---------------------+     +---------------+
```

> ⚠️ **Note:** Some textbooks (including older Cisco materials) show the TCP/IP model as **5 layers**, splitting the bottom layer into "Data Link" and "Physical" separately. Both representations are acceptable — just know what each means.

---

## The 4 Layers of TCP/IP

### Layer 4 — Application Layer

**Maps to OSI Layers:** 5, 6, and 7

**Function:** Provides the interface between user applications and the network. This single layer in TCP/IP handles what OSI splits into three layers: session management, data formatting/encryption, and application-level protocols.

**What happens here:**
- Applications request network resources
- Data is formatted for transmission
- Sessions are established and managed

**Examples in action:**
- Your browser sends an **HTTP GET request** to a web server
- Your email client uses **SMTP** to send an email
- A device queries a **DNS server** to resolve a domain name

---

### Layer 3 — Transport Layer

**Maps to OSI Layer:** 4

**Function:** End-to-end communication between applications on different hosts. Uses **port numbers** to direct traffic to the correct application process.

**Two protocols:**

```
TCP (Transmission Control Protocol)
├── Connection-oriented (requires handshake)
├── Reliable (guarantees delivery)
├── Ordered delivery
├── Error checking + retransmission
├── Flow control (sliding window)
└── Use cases: Web, Email, File Transfer

UDP (User Datagram Protocol)
├── Connectionless (no handshake)
├── Unreliable (best-effort delivery)
├── No guaranteed ordering
├── Minimal overhead
├── Speed-optimized
└── Use cases: DNS, DHCP, VoIP, Streaming, Gaming
```

---

### Layer 2 — Internet Layer

**Maps to OSI Layer:** 3

**Function:** Logical addressing and routing across multiple networks (internetworking). This is where IP operates.

**Core protocol: IP (Internet Protocol)**
- Assigns **logical addresses** (IP addresses) to hosts
- **Routes packets** from source network to destination network
- **Connectionless** — each packet is independent

**Supporting protocols at this layer:**

| Protocol | Purpose | Details |
|----------|---------|---------|
| IPv4 | Logical addressing | 32-bit addresses |
| IPv6 | Logical addressing (next-gen) | 128-bit addresses |
| ICMP | Error/diagnostic messages | Used by `ping` and `traceroute` |
| ARP | Resolves IP → MAC address | Works at L2/L3 boundary |
| IGMP | Multicast group management | Used for streaming/multicast |

---

### Layer 1 — Network Access Layer

**Maps to OSI Layers:** 1 and 2

**Function:** Handles the physical transmission of data on a specific network medium. Combines both the Data Link and Physical layers of OSI into one.

**Responsibilities:**
- Physical media access (cables, radio waves)
- MAC addressing and framing (Ethernet)
- Error detection at the frame level
- Encoding bits onto the medium

**Protocols and standards:**

| Technology | Type | Use |
|-----------|------|-----|
| Ethernet (802.3) | Wired LAN | Most common wired standard |
| Wi-Fi (802.11) | Wireless LAN | Wireless access |
| PPP | WAN | Point-to-point serial links |
| HDLC | WAN | Cisco default serial encapsulation |
| DSL | WAN/Broadband | Digital subscriber line |
| Cable (DOCSIS) | WAN/Broadband | Coaxial broadband |
| Fiber (SONET/SDH) | WAN | High-speed backbone links |

---

## Key Protocols at Each Layer

Here is a comprehensive protocol reference organized by TCP/IP layer:

### Application Layer Protocols

```
Protocol   | Port(s)    | TCP/UDP | Purpose
-----------|------------|---------|----------------------------------------
HTTP       | 80         | TCP     | Unencrypted web browsing
HTTPS      | 443        | TCP     | Encrypted web browsing (TLS)
FTP        | 20 (data)  | TCP     | File transfer (active mode data)
           | 21 (ctrl)  | TCP     | File transfer (control channel)
SFTP       | 22         | TCP     | Secure file transfer (over SSH)
SSH        | 22         | TCP     | Encrypted remote CLI access
Telnet     | 23         | TCP     | Unencrypted remote CLI (avoid!)
SMTP       | 25         | TCP     | Sending email (server-to-server)
DNS        | 53         | UDP/TCP | Domain name resolution
           |            |         | (UDP for queries, TCP for zone transfers)
DHCP       | 67 (server)| UDP     | IP address assignment
           | 68 (client)| UDP     | (Client → broadcast on 68)
TFTP       | 69         | UDP     | Trivial file transfer (no auth)
HTTP alt   | 8080       | TCP     | Alternative web/proxy port
POP3       | 110        | TCP     | Retrieving email (deletes from server)
IMAP       | 143        | TCP     | Retrieving email (keeps on server)
SNMP       | 161        | UDP     | Network device monitoring
SNMP Trap  | 162        | UDP     | SNMP alerts from devices
LDAP       | 389        | TCP/UDP | Directory services (Active Directory)
HTTPS alt  | 8443       | TCP     | Alternative HTTPS port
SMB        | 445        | TCP     | Windows file/printer sharing
Syslog     | 514        | UDP     | System/device logging
SMTPS      | 587        | TCP     | Secure SMTP (email submission)
LDAPS      | 636        | TCP     | Encrypted LDAP
IMAPS      | 993        | TCP     | Encrypted IMAP
POP3S      | 995        | TCP     | Encrypted POP3
NTP        | 123        | UDP     | Network Time Protocol
RDP        | 3389       | TCP     | Windows Remote Desktop Protocol
```

> 🎯 **Exam Focus:** Memorize all well-known ports above. The CCNA exam frequently tests port numbers.

### Transport Layer Protocols

| Protocol | Port Range Used | Key Feature |
|----------|----------------|-------------|
| TCP | Any (1-65535) | Reliable, connection-oriented |
| UDP | Any (1-65535) | Fast, connectionless |

### Internet Layer Protocols

| Protocol | Function | Notes |
|----------|---------|-------|
| IPv4 | Addressing & routing | 32-bit, most common |
| IPv6 | Addressing & routing | 128-bit, growing adoption |
| ICMP v4 | Error messages & diagnostics | Used by ping/traceroute |
| ICMPv6 | IPv6 error + neighbor discovery | Replaces ARP for IPv6 |
| ARP | IP-to-MAC resolution | Layer 2/3 boundary |
| RARP | MAC-to-IP (legacy) | Replaced by DHCP |

### Network Access Layer Standards

| Standard | Speed | Medium |
|----------|-------|--------|
| 802.3 (Ethernet) | 10 Mbps – 400 Gbps | Copper / Fiber |
| 802.11a | 54 Mbps | 5 GHz wireless |
| 802.11b | 11 Mbps | 2.4 GHz wireless |
| 802.11g | 54 Mbps | 2.4 GHz wireless |
| 802.11n (Wi-Fi 4) | 600 Mbps | 2.4/5 GHz wireless |
| 802.11ac (Wi-Fi 5) | 3.5 Gbps | 5 GHz wireless |
| 802.11ax (Wi-Fi 6) | 9.6 Gbps | 2.4/5/6 GHz wireless |

---

## How Data Flows Through TCP/IP

Let's trace what happens when you type `www.example.com` into your browser:

```
Step 1: Application Layer
─────────────────────────
Browser wants to load www.example.com
→ DNS query sent to resolve IP (UDP port 53)
→ DNS returns: 93.184.216.34
→ Browser builds HTTP GET request

Step 2: Transport Layer
────────────────────────
TCP takes the HTTP request and:
→ Performs 3-way handshake with 93.184.216.34 on port 80
→ Breaks HTTP data into segments
→ Adds source port (e.g., 54231) and destination port (80)
→ Tracks sequence numbers for reassembly

Step 3: Internet Layer
───────────────────────
IP takes each TCP segment and:
→ Adds source IP (your IP, e.g., 192.168.1.10)
→ Adds destination IP (93.184.216.34)
→ Determines best route (via default gateway)
→ Creates IP packets

Step 4: Network Access Layer
─────────────────────────────
Ethernet takes each packet and:
→ Adds source MAC (your NIC's MAC)
→ Adds destination MAC (default gateway's MAC via ARP)
→ Creates frames with CRC error check
→ Converts to electrical signals on the wire
```

---

## Important Protocols Deep Dive

### ARP — Address Resolution Protocol

ARP bridges the gap between Layer 3 (IP addresses) and Layer 2 (MAC addresses). Before a device can send a frame, it must know the MAC address of the destination.

**ARP Process:**

```
PC-A wants to send data to PC-B (192.168.1.20)
PC-A knows PC-B's IP but NOT its MAC address

Step 1: ARP Request (BROADCAST)
PC-A sends: "Who has 192.168.1.20? Tell 192.168.1.10"
Destination MAC: FF:FF:FF:FF:FF:FF (broadcast — everyone receives it)

Step 2: ARP Reply (UNICAST)
PC-B responds: "192.168.1.20 is at AA:BB:CC:DD:EE:FF"
Sent directly (unicast) back to PC-A

Step 3: Cache
PC-A stores this mapping in its ARP cache for future use
(typically kept for ~20 minutes)
```

**Check ARP cache on a device:**
```bash
# Windows
arp -a

# Linux/Mac
arp -n
ip neigh show

# Cisco IOS
show arp
```

---

### ICMP — Internet Control Message Protocol

ICMP is used for **error reporting** and **network diagnostics**. It operates at the Internet (Layer 3) layer and is embedded within IP packets.

**Common ICMP Message Types:**

| Type | Code | Meaning |
|------|------|---------|
| 0 | 0 | Echo Reply (ping response) |
| 3 | 0 | Destination Network Unreachable |
| 3 | 1 | Destination Host Unreachable |
| 3 | 3 | Destination Port Unreachable |
| 5 | 0 | Redirect — use different route |
| 8 | 0 | Echo Request (ping request) |
| 11 | 0 | TTL Exceeded (used by traceroute) |

**Ping Operation:**

```
PC-A                           PC-B
 |--ICMP Echo Request (Type 8)-->|
 |<-ICMP Echo Reply   (Type 0)---|

If PC-B is unreachable:
 |--ICMP Echo Request----------->| → lost
 |<-ICMP Type 3 (Unreachable)---| ← from router or no reply at all
```

**Traceroute Operation:**
Traceroute works by sending packets with incrementing **TTL (Time To Live)** values:
- TTL=1: First router returns "TTL Exceeded" (reveals router 1's IP)
- TTL=2: Second router returns "TTL Exceeded" (reveals router 2's IP)
- TTL=N: Destination reached

```
Traceroute to 8.8.8.8:
1  192.168.1.1    (your gateway)      1 ms
2  10.10.50.1     (ISP router)        8 ms
3  172.16.100.5   (ISP backbone)     12 ms
4  8.8.8.8        (destination)      14 ms
```

---

### DNS — Domain Name System

DNS translates human-readable **domain names** (www.google.com) into **IP addresses** (142.250.80.46).

**DNS Resolution Process:**

```
Browser → www.google.com
        ↓
[1] Check local cache — not found
        ↓
[2] Ask Recursive Resolver (your ISP or 8.8.8.8)
        ↓
[3] Resolver asks Root DNS Server → "try .com TLD servers"
        ↓
[4] Resolver asks .com TLD Server → "try google.com NS"
        ↓
[5] Resolver asks Google's Authoritative DNS → "142.250.80.46"
        ↓
[6] Resolver returns IP to browser + caches result
        ↓
Browser connects to 142.250.80.46
```

**Common DNS Record Types:**

| Record | Purpose | Example |
|--------|---------|---------|
| A | IPv4 address | www → 93.184.216.34 |
| AAAA | IPv6 address | www → 2606:2800:220:1:... |
| CNAME | Alias to another name | mail → mailhost.example.com |
| MX | Mail server for domain | example.com → mail.example.com |
| NS | Authoritative nameserver | example.com → ns1.example.com |
| PTR | Reverse DNS (IP → name) | 34.216.184.93 → www.example.com |
| TXT | Text information (SPF, DKIM) | "v=spf1 include:..." |
| SOA | Start of Authority (zone info) | Admin contact, serial number |

---

### DHCP — Dynamic Host Configuration Protocol

DHCP automatically assigns IP configuration to network devices.

**DHCP DORA Process:**

```
Client                          DHCP Server
  |                                  |
  |---DISCOVER (broadcast)---------->|  "Is there a DHCP server?"
  |                                  |
  |<--OFFER (unicast/broadcast)------|  "I offer you 192.168.1.50"
  |                                  |
  |---REQUEST (broadcast)----------->|  "I'll take 192.168.1.50"
  |                                  |
  |<--ACK (unicast/broadcast)--------|  "It's yours! Lease = 24 hours"
  |                                  |
Client configures itself:
  IP:      192.168.1.50
  Mask:    255.255.255.0
  Gateway: 192.168.1.1
  DNS:     8.8.8.8
```

**Information DHCP provides (DORA = Discover, Offer, Request, Acknowledge):**
- IP Address
- Subnet Mask
- Default Gateway
- DNS Server(s)
- Lease Duration
- (Optional) WINS, NTP servers, etc.

---

## Key Takeaways

- TCP/IP has **4 layers**: Application, Transport, Internet, Network Access
- It **maps to the OSI model** but consolidates layers 5–7 into Application and layers 1–2 into Network Access
- TCP/IP is the **actual implemented model** — OSI is the theoretical reference
- **ARP** resolves IP addresses to MAC addresses (works at L2/L3 boundary)
- **ICMP** is used for diagnostics — `ping` uses ICMP Echo Request/Reply
- **DNS** resolves hostnames to IPs — uses **UDP port 53** (TCP for zone transfers)
- **DHCP** assigns IP config automatically — uses **UDP ports 67/68** (DORA process)
- Memorize **all well-known port numbers** — they appear frequently on the exam

---

## Exam Tips

> 🎯 **High-Frequency Exam Questions:**

1. **"Which port does HTTPS use?"** → **443 (TCP)**
2. **"What protocol resolves IP addresses to MAC addresses?"** → **ARP**
3. **"What are the steps of the DHCP process?"** → **DORA (Discover, Offer, Request, ACK)**
4. **"Which protocol does traceroute use?"** → **ICMP (TTL Exceeded messages)**
5. **"What port does DNS use?"** → **53 (UDP for queries, TCP for zone transfers)**
6. **"Which layer of TCP/IP corresponds to OSI Layers 5, 6, and 7?"** → **Application**
7. **"What is the destination MAC in an ARP request?"** → **FF:FF:FF:FF:FF:FF (broadcast)**

> 📝 **Trick Question Alert:** Some exam questions ask about the "TCP/IP model" with 5 layers (splitting Network Access into Data Link + Physical). If you see 5 layers listed, the mapping still holds — just recognize both versions.

---

*← Previous: [01 — OSI Model](./01_OSI_Model.md)*  
*Next: [03 — IP Addressing →](./03_IP_Addressing.md)*
