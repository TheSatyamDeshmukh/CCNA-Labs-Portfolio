# 01 — OSI Model: Layers, Functions & Data Flow

> **CCNA Exam Domain:** 1.0 Network Fundamentals  
> **Exam Weight:** ~20% of total exam  
> **Difficulty:** ⭐⭐☆☆☆

---

## Table of Contents
- [What is the OSI Model?](#what-is-the-osi-model)
- [The 7 Layers — Overview](#the-7-layers--overview)
- [Layer-by-Layer Deep Dive](#layer-by-layer-deep-dive)
- [Data Encapsulation & De-encapsulation](#data-encapsulation--de-encapsulation)
- [PDUs — Protocol Data Units](#pdus--protocol-data-units)
- [OSI vs Real-World Devices](#osi-vs-real-world-devices)
- [Key Takeaways](#key-takeaways)
- [Exam Tips](#exam-tips)

---

## What is the OSI Model?

The **OSI (Open Systems Interconnection)** model is a **conceptual framework** developed by the International Organization for Standardization (ISO) in 1984. It describes how data travels from one device to another across a network by breaking the communication process into **7 distinct layers**.

### Why does it matter?

- Provides a **common language** for vendors and engineers
- Helps **troubleshoot** network problems by isolating the faulty layer
- Allows different vendors' equipment to **interoperate**
- Separates concerns so each layer can be developed **independently**

> 💡 **Real-World Analogy:** Think of sending a physical letter. You write the letter (application), put it in an envelope (presentation/session), add the address (network), hand it to the postal service (transport/data link), and the postal truck delivers it (physical). Each step is independent yet works together.

---

## The 7 Layers — Overview

```
+---------------------------+-----------------------------+-------------------+
|  Layer #  |  Layer Name   |      Primary Function       |   Data Unit (PDU) |
+-----------+---------------+-----------------------------+-------------------+
|  Layer 7  |  Application  |  User interface & services  |      Data         |
|  Layer 6  |  Presentation |  Formatting, encryption     |      Data         |
|  Layer 5  |  Session      |  Session management         |      Data         |
|  Layer 4  |  Transport    |  End-to-end delivery        |      Segment      |
|  Layer 3  |  Network      |  Logical addressing/routing |      Packet       |
|  Layer 2  |  Data Link    |  Physical addressing (MAC)  |      Frame        |
|  Layer 1  |  Physical     |  Bits over the wire         |      Bit          |
+-----------+---------------+-----------------------------+-------------------+
```

### Memory Mnemonics

**Top-Down (Layer 7 → 1):**
> **A**ll **P**eople **S**eem **T**o **N**eed **D**ata **P**rocessing

**Bottom-Up (Layer 1 → 7):**
> **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way

---

## Layer-by-Layer Deep Dive

---

### Layer 7 — Application Layer

**Function:** The topmost layer. Provides **network services directly to end-user applications**. This is the layer humans interact with.

**Key Responsibilities:**
- Identifying communication partners
- Determining resource availability
- Synchronizing communication

**Common Protocols:**

| Protocol | Port | Purpose |
|----------|------|---------|
| HTTP | 80 | Web browsing (unencrypted) |
| HTTPS | 443 | Web browsing (encrypted) |
| FTP | 20/21 | File transfer |
| SMTP | 25 | Sending email |
| IMAP | 143 | Receiving email |
| DNS | 53 | Name-to-IP resolution |
| DHCP | 67/68 | Automatic IP assignment |
| SSH | 22 | Secure remote access |
| Telnet | 23 | Remote access (unencrypted) |
| SNMP | 161/162 | Network device management |

> ⚠️ **Common Mistake:** The Application Layer does NOT mean "applications like Chrome or Outlook." It refers to the **protocols** those apps use to communicate. Chrome uses HTTP/HTTPS; Outlook uses SMTP/IMAP.

---

### Layer 6 — Presentation Layer

**Function:** Acts as the **translator** between the application and the network. It ensures that data from the sender can be understood by the receiver regardless of format differences.

**Key Responsibilities:**
- **Translation:** Converts data formats (e.g., ASCII to EBCDIC)
- **Encryption/Decryption:** SSL/TLS operates here
- **Compression:** Reduces data size for efficient transmission (e.g., JPEG, MP3, MPEG)

**Examples:**
- Encrypting a password before sending it
- Compressing a video file for streaming
- Converting character encoding (UTF-8 ↔ ASCII)

> 💡 **Example:** When you access a website over HTTPS, the Presentation Layer handles the **TLS handshake** and encryption of your data before it is passed down to the Transport Layer.

---

### Layer 5 — Session Layer

**Function:** Manages **sessions** — persistent connections between two devices. A session is the logical link that allows a conversation to take place over time.

**Key Responsibilities:**
- **Establishing** sessions (login, authentication)
- **Maintaining** sessions (keeping connections alive)
- **Terminating** sessions (logging out, closing connections)
- **Synchronization** using checkpoints (so large file transfers can resume if interrupted)

**Protocols/Examples:**
- **NetBIOS** — Windows file/printer sharing
- **RPC (Remote Procedure Call)** — Used in distributed computing
- **SQL sessions** — Database query sessions
- **H.245** — Session control in video conferencing

> 💡 **Example:** When you log into a website and stay logged in as you browse different pages, the Session Layer is maintaining your active session. When you click "Logout," the Session Layer terminates it.

---

### Layer 4 — Transport Layer

**Function:** Provides **reliable (or unreliable) end-to-end data delivery** between applications on different hosts. This is the first layer that deals with host-to-host communication.

**Key Responsibilities:**
- **Segmentation:** Breaking large data into smaller segments
- **Reassembly:** Putting segments back together at destination
- **Flow Control:** Ensuring sender doesn't overwhelm receiver
- **Error Detection/Correction:** Acknowledging data and retransmitting if lost
- **Port Addressing:** Using port numbers to identify specific applications

**Two Main Protocols:**

```
+----------------+------------------------------------------+
| Protocol       | Characteristics                          |
+----------------+------------------------------------------+
| TCP            | - Connection-oriented (3-way handshake)  |
| (Transmission  | - Reliable (acknowledgements + retransmit)|
| Control        | - Ordered delivery                       |
| Protocol)      | - Flow control & congestion control      |
|                | - Slower due to overhead                 |
|                | - Use: HTTP, FTP, SSH, Email             |
+----------------+------------------------------------------+
| UDP            | - Connectionless (no handshake)          |
| (User Datagram | - Unreliable (no acknowledgements)       |
| Protocol)      | - No guaranteed ordering                 |
|                | - No flow control                        |
|                | - Fast, low overhead                     |
|                | - Use: DNS, DHCP, VoIP, Video streaming  |
+----------------+------------------------------------------+
```

**TCP Three-Way Handshake:**

```
Client                         Server
  |                              |
  |--- SYN (seq=100) ----------->|   Step 1: Client says "I want to connect"
  |                              |
  |<-- SYN-ACK (seq=200,ack=101)-|   Step 2: Server acknowledges and responds
  |                              |
  |--- ACK (ack=201) ----------->|   Step 3: Client acknowledges
  |                              |
  |<====== DATA FLOWS ===========>|  Connection Established!
```

**Port Number Ranges:**

| Range | Type | Description |
|-------|------|-------------|
| 0 – 1023 | Well-Known Ports | Reserved for common services (HTTP=80, SSH=22) |
| 1024 – 49151 | Registered Ports | Assigned by IANA for specific applications |
| 49152 – 65535 | Dynamic/Ephemeral Ports | Assigned temporarily by OS to client applications |

---

### Layer 3 — Network Layer

**Function:** Handles **logical addressing and routing** — determining the best path for data to travel from source to destination across multiple networks.

**Key Responsibilities:**
- **Logical Addressing:** IP addresses (IPv4 and IPv6)
- **Routing:** Forwarding packets between networks using routing tables
- **Path Determination:** Choosing the best route (via routing protocols)
- **Fragmentation:** Splitting packets if they exceed the MTU (Maximum Transmission Unit)

**Key Protocols:**

| Protocol | Purpose |
|----------|---------|
| IPv4 | Logical addressing (32-bit) |
| IPv6 | Logical addressing (128-bit) |
| ICMP | Error reporting (used by `ping` and `traceroute`) |
| OSPF | Link-state routing protocol |
| EIGRP | Cisco proprietary routing protocol |
| BGP | Routing between autonomous systems (internet routing) |

**Device at this layer:** **Router** (and Layer 3 switches)

```
        Network A               Network B
   192.168.1.0/24            10.0.0.0/24

   [PC1]----[Switch]----[Router]----[Switch]----[Server]
   192.168.1.10         ^                        10.0.0.5
                        |
              Routes between networks
              using IP addressing
```

---

### Layer 2 — Data Link Layer

**Function:** Provides **node-to-node delivery** on the same physical network segment. It uses **MAC addresses** to identify devices locally.

**Key Responsibilities:**
- **Physical Addressing:** MAC (Media Access Control) addresses
- **Framing:** Packaging data into frames with headers and trailers
- **Error Detection:** Using CRC (Cyclic Redundancy Check) in frame trailer
- **Media Access Control:** Managing access to the shared medium (CSMA/CD for Ethernet)

**Sub-layers:**

```
+-----------------------------+
|        Data Link Layer      |
+-----------------------------+
|  LLC  |  Logical Link       |   ← Communicates with Layer 3 (Network)
|       |  Control            |     Handles flow control, error notification
+-----------------------------+
|  MAC  |  Media Access       |   ← Communicates with Layer 1 (Physical)
|       |  Control            |     Handles MAC addressing, frame construction
+-----------------------------+
```

**Ethernet Frame Structure:**

```
+----------+----------+---------+----------+------+-----+
| Preamble | Dest MAC | Src MAC | EtherType | Data | FCS |
| (8 bytes)| (6 bytes)|(6 bytes)| (2 bytes) |46-1500| (4) |
+----------+----------+---------+----------+------+-----+
```

**Device at this layer:** **Switch** (and network bridges, NICs)

> 💡 **MAC Address Format:** `AA:BB:CC:DD:EE:FF`  
> - First 3 bytes (OUI): Identifies the manufacturer (e.g., `00:1A:2B` = Cisco)  
> - Last 3 bytes: Unique identifier assigned by manufacturer

---

### Layer 1 — Physical Layer

**Function:** Deals with the **actual transmission of raw bits** (0s and 1s) over a physical medium. It defines the electrical, optical, or radio signals used to carry data.

**Key Responsibilities:**
- **Bit transmission:** Encoding and decoding bits onto the medium
- **Media types:** Copper cable, fiber optic, wireless (radio)
- **Signal characteristics:** Voltage levels, timing, frequencies
- **Physical connectors:** RJ-45, fiber connectors, coax

**Common Standards & Media:**

| Type | Standard/Example | Speed | Distance |
|------|-----------------|-------|----------|
| Copper (UTP) | Cat5e Ethernet | Up to 1 Gbps | 100m |
| Copper (UTP) | Cat6 Ethernet | Up to 10 Gbps | 55m (10G) |
| Fiber (Single-mode) | OS2 | Up to 100 Gbps | 40+ km |
| Fiber (Multi-mode) | OM4 | Up to 100 Gbps | 400m |
| Wireless | 802.11ax (Wi-Fi 6) | Up to 9.6 Gbps | ~300m (open air) |
| Coax | 10BASE2 | 10 Mbps | 185m |

**Device at this layer:** **Hub**, cables, repeaters, modems, network interface cards (NIC)

---

## Data Encapsulation & De-encapsulation

As data travels **down** the OSI stack on the **sending device**, each layer **adds its own header (and sometimes trailer)** — this is called **encapsulation**.

As data travels **up** the OSI stack on the **receiving device**, each layer **removes its header** — this is called **de-encapsulation**.

```
SENDER (Encapsulation)                    RECEIVER (De-encapsulation)
===========================               ===========================

Layer 7: [      DATA      ]         →     Layer 7: [      DATA      ]
         ↓ Add L7/6/5 info                         ↑ Read & remove
Layer 4: [ TCP | DATA      ]        →     Layer 4: [ TCP | DATA      ]
         ↓ Add TCP header                          ↑ Read & remove
Layer 3: [ IP | TCP | DATA  ]       →     Layer 3: [ IP | TCP | DATA  ]
         ↓ Add IP header                           ↑ Read & remove
Layer 2: [ETH| IP | TCP |DATA|FCS]  →     Layer 2: [ETH| IP |TCP|DATA|FCS]
         ↓ Add Eth header+trailer                  ↑ Check FCS, remove
Layer 1: 010110100110101001101010...  →    Layer 1: 010110100110101001101010...
         (Raw bits on the wire)                    (Raw bits received)
```

---

## PDUs — Protocol Data Units

Each layer has a specific name for the unit of data it works with:

| Layer | PDU Name | Contains |
|-------|----------|---------|
| Layer 7–5 | **Data** | Application payload |
| Layer 4 | **Segment** (TCP) / **Datagram** (UDP) | L4 header + Data |
| Layer 3 | **Packet** | L3 header + Segment |
| Layer 2 | **Frame** | L2 header + Packet + L2 trailer |
| Layer 1 | **Bit** | Binary representation of Frame |

> ⚠️ **Exam Tip:** The CCNA exam will often ask "what PDU does the Network layer use?" → Answer: **Packet**. "What PDU does the Data Link layer use?" → Answer: **Frame**.

---

## OSI vs Real-World Devices

| Device | OSI Layers | Description |
|--------|-----------|-------------|
| **Hub** | Layer 1 | Repeats all incoming signals out all ports — no intelligence |
| **Switch** | Layer 2 (and L3 for multilayer) | Forwards frames based on MAC address table |
| **Router** | Layer 3 | Forwards packets based on IP routing table |
| **Firewall** | Layer 3–7 | Filters traffic based on rules at various layers |
| **NIC** | Layer 1–2 | Connects device to the network medium |
| **WAP (Wireless AP)** | Layer 1–2 | Provides wireless access using 802.11 standards |

---

## Key Takeaways

- The OSI model has **7 layers**: Physical, Data Link, Network, Transport, Session, Presentation, Application
- **Encapsulation** = adding headers as data moves DOWN the stack (sender side)
- **De-encapsulation** = removing headers as data moves UP the stack (receiver side)
- Each layer uses its own **PDU**: Bits → Frames → Packets → Segments → Data
- **Switches** operate at Layer 2; **Routers** operate at Layer 3
- The OSI model is **conceptual** — real-world protocols (like TCP/IP) don't map perfectly to it

---

## Exam Tips

> 🎯 **High-Frequency Exam Questions:**

1. **"Which layer is responsible for logical addressing?"** → **Layer 3 (Network)**
2. **"Which layer breaks data into segments?"** → **Layer 4 (Transport)**
3. **"What is the PDU at Layer 2?"** → **Frame**
4. **"Which layer does a switch primarily operate at?"** → **Layer 2**
5. **"Which layer handles encryption (SSL/TLS)?"** → **Layer 6 (Presentation)**
6. **"What device operates at Layer 1?"** → **Hub**
7. **"Which layer manages sessions and dialog control?"** → **Layer 5 (Session)**

> 📝 **Remember:** Layers 1–4 are often called the **lower layers** (concerned with transport), while Layers 5–7 are the **upper layers** (concerned with applications).

---

*Next: [02 — TCP/IP Model →](./02_TCPIP_Model.md)*
