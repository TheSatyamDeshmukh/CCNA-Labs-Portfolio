# 05 — TCP vs UDP: Transport Layer Protocols

> **CCNA Exam Domain:** 1.0 Network Fundamentals  
> **Exam Weight:** ~20% of total exam  
> **Difficulty:** ⭐⭐⭐☆☆

---

## Table of Contents
- [Transport Layer Overview](#transport-layer-overview)
- [TCP — Transmission Control Protocol](#tcp--transmission-control-protocol)
- [UDP — User Datagram Protocol](#udp--user-datagram-protocol)
- [TCP vs UDP — Full Comparison](#tcp-vs-udp--full-comparison)
- [Port Numbers Deep Dive](#port-numbers-deep-dive)
- [TCP Connection Management](#tcp-connection-management)
- [TCP Flow Control & Windowing](#tcp-flow-control--windowing)
- [TCP Congestion Control](#tcp-congestion-control)
- [Choosing TCP vs UDP](#choosing-tcp-vs-udp)
- [Key Takeaways](#key-takeaways)
- [Exam Tips](#exam-tips)

---

## Transport Layer Overview

The **Transport Layer (Layer 4)** is responsible for **end-to-end communication** between processes running on different hosts. It sits between the Application layer (which needs data delivered) and the Network layer (which routes packets between networks).

### Key Functions of the Transport Layer

```
1. MULTIPLEXING / DEMULTIPLEXING
   Multiple applications can use the network simultaneously.
   Port numbers identify which process gets which data.

   Host A:                          Host B:
   ┌─────────────────┐              ┌─────────────────┐
   │ Browser (port   │◄────HTTP────►│ Web Server      │
   │ 54231 → 80)     │              │ (port 80)       │
   ├─────────────────┤              ├─────────────────┤
   │ SSH Client      │◄─────SSH────►│ SSH Server      │
   │ (54232 → 22)    │              │ (port 22)       │
   ├─────────────────┤              ├─────────────────┤
   │ Email (54233    │◄────SMTP────►│ Email Server    │
   │ → 25)           │              │ (port 25)       │
   └─────────────────┘              └─────────────────┘

2. SEGMENTATION & REASSEMBLY
   Breaks large data into segments for transmission,
   reassembles them in order at the destination.

3. CONNECTION ESTABLISHMENT & TERMINATION (TCP only)
   Establishes a reliable connection before data transfer.

4. ERROR DETECTION & RECOVERY (TCP only)
   Detects lost/corrupted segments and retransmits them.

5. FLOW CONTROL (TCP only)
   Prevents a fast sender from overwhelming a slow receiver.
```

---

## TCP — Transmission Control Protocol

TCP is a **connection-oriented, reliable** protocol. It ensures all data arrives correctly and in order.

### TCP Segment Header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |           |U|A|P|R|S|F|                               |
| Offset| Reserved  |R|C|S|S|Y|I|            Window             |
|       |           |G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options (if any)                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             Data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### TCP Header Fields Explained

| Field | Size | Description |
|-------|------|-------------|
| Source Port | 16 bits | Sending application's port number |
| Destination Port | 16 bits | Receiving application's port number |
| Sequence Number | 32 bits | Byte number of first byte in this segment |
| Acknowledgment Number | 32 bits | Next byte the receiver expects |
| Data Offset | 4 bits | TCP header length in 32-bit words |
| Control Flags | 6 bits | URG, ACK, PSH, RST, SYN, FIN |
| Window Size | 16 bits | Receiver's buffer space (flow control) |
| Checksum | 16 bits | Error detection |
| Urgent Pointer | 16 bits | Used when URG flag is set |

### TCP Control Flags

| Flag | Name | Purpose |
|------|------|---------|
| **SYN** | Synchronize | Initiates connection; exchanges sequence numbers |
| **ACK** | Acknowledge | Confirms receipt of data |
| **FIN** | Finish | Gracefully terminates connection |
| **RST** | Reset | Abruptly terminates connection (error state) |
| **PSH** | Push | Instructs receiver to pass data to application immediately |
| **URG** | Urgent | Indicates urgent data (rarely used) |

---

## UDP — User Datagram Protocol

UDP is a **connectionless, unreliable** protocol that prioritizes speed over reliability.

### UDP Datagram Header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Length             |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             Data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### Why UDP Header Is So Small

UDP has only **4 fields** (8 bytes total) — compared to TCP's minimum **20-byte** header. This minimal overhead makes UDP ideal for:
- High-frequency, low-latency applications
- Applications that implement their own reliability (e.g., QUIC, used by HTTP/3)
- Short transactions where connection setup cost outweighs benefits (e.g., DNS)

---

## TCP vs UDP — Full Comparison

```
Characteristic        | TCP                        | UDP
----------------------|----------------------------|---------------------------
Connection type       | Connection-oriented        | Connectionless
Reliability           | Guaranteed delivery        | Best-effort (no guarantee)
Ordering              | Ordered (seq numbers)      | No ordering guarantee
Error recovery        | Yes (retransmission)       | No
Flow control          | Yes (windowing)            | No
Congestion control    | Yes                        | No
Header size           | Minimum 20 bytes           | 8 bytes (fixed)
Speed                 | Slower (more overhead)     | Faster (minimal overhead)
Handshake required    | Yes (3-way handshake)      | No
Acknowledgements      | Yes (ACK for every segment)| No
Suitable for          | Web, Email, FTP, SSH       | DNS, DHCP, VoIP, Streaming
Analogy               | Certified mail             | Regular mail
```

---

## Port Numbers Deep Dive

**Port numbers** identify specific processes or services on a host. Combined with an IP address, they form a **socket** that uniquely identifies an application-to-application communication.

```
Socket = IP Address + Port Number
Example: 192.168.1.10:54231 ↔ 93.184.216.34:80
```

### Well-Known Ports (0–1023) — MUST MEMORIZE

```
Port  | Protocol | Service
------|----------|--------------------------------------------
20    | TCP      | FTP Data (active mode)
21    | TCP      | FTP Control
22    | TCP      | SSH (Secure Shell)
23    | TCP      | Telnet (unencrypted — avoid in production!)
25    | TCP      | SMTP (Simple Mail Transfer Protocol)
53    | UDP/TCP  | DNS (UDP for queries; TCP for zone transfers)
67    | UDP      | DHCP Server (receives from clients)
68    | UDP      | DHCP Client (receives from server)
69    | UDP      | TFTP (Trivial File Transfer Protocol)
80    | TCP      | HTTP
110   | TCP      | POP3 (Post Office Protocol v3)
123   | UDP      | NTP (Network Time Protocol)
143   | TCP      | IMAP (Internet Message Access Protocol)
161   | UDP      | SNMP (Simple Network Management Protocol)
162   | UDP      | SNMP Trap
179   | TCP      | BGP (Border Gateway Protocol)
389   | TCP/UDP  | LDAP (Lightweight Directory Access Protocol)
443   | TCP      | HTTPS (HTTP over TLS/SSL)
445   | TCP      | SMB/CIFS (Windows File Sharing)
514   | UDP      | Syslog
587   | TCP      | SMTP Submission (email from clients)
636   | TCP      | LDAPS (LDAP over SSL)
993   | TCP      | IMAPS (IMAP over SSL)
995   | TCP      | POP3S (POP3 over SSL)
1433  | TCP      | Microsoft SQL Server
1521  | TCP      | Oracle Database
3306  | TCP      | MySQL
3389  | TCP      | RDP (Remote Desktop Protocol)
5060  | UDP/TCP  | SIP (VoIP Session Initiation Protocol)
8080  | TCP      | HTTP Alternate / Web Proxy
8443  | TCP      | HTTPS Alternate
```

---

## TCP Connection Management

### Three-Way Handshake (Connection Establishment)

```
Client                              Server
  |                                    |
  |──── SYN (seq=100, SYN=1) ────────►|  "I want to connect, my ISN is 100"
  |                                    |
  |◄─── SYN-ACK (seq=200, ack=101) ───|  "OK, my ISN is 200, I got your 100 (ack=101)"
  |                                    |
  |──── ACK (seq=101, ack=201) ──────►|  "Got it. Let's go (ack=201)"
  |                                    |
  |◄═══════ Data Exchange ════════════►|
```

**ISN (Initial Sequence Number):** A randomly chosen starting sequence number. Used to track byte ordering and detect stale segments from old connections.

### Four-Way Handshake (Connection Termination)

TCP uses a **four-step** process to close connections gracefully because TCP is **full-duplex** — each direction must be independently closed:

```
Client                              Server
  |                                    |
  |──── FIN (seq=500) ───────────────►|  "I'm done sending (client side)"
  |                                    |
  |◄─── ACK (ack=501) ────────────────|  "Got your FIN"
  |                                    |
  |              (Server finishes sending any remaining data)
  |                                    |
  |◄─── FIN (seq=300) ────────────────|  "I'm also done sending (server side)"
  |                                    |
  |──── ACK (ack=301) ───────────────►|  "Got your FIN. Goodbye!"
  |                                    |
  Connection closed

Note: Client enters TIME-WAIT state for 2x MSL (Maximum Segment Lifetime)
before fully closing — ensures all packets have cleared the network.
```

### TCP Connection Reset

When something goes wrong (port not open, error, firewall block), TCP sends a **RST** (Reset) to immediately terminate the connection without the 4-way handshake.

```
Client                              Server (port 9999 not open)
  |                                    |
  |──── SYN (port 9999) ─────────────►|
  |                                    |
  |◄─── RST-ACK ──────────────────────|  "That port isn't open. Connection refused."
```

---

## TCP Flow Control & Windowing

**Flow control** prevents the sender from overwhelming the receiver. TCP uses a **sliding window** mechanism.

### How the Window Works

The **window size** in the TCP header tells the sender how much data the receiver's buffer can currently accept.

```
Scenario: Receiver has 3000 bytes of buffer space
          Sender's window = 3000 bytes

Sender can transmit up to 3000 bytes before waiting for ACK:

Segment 1: bytes 1-1000   (1000 bytes)  ──────►
Segment 2: bytes 1001-2000 (1000 bytes) ──────►
Segment 3: bytes 2001-3000 (1000 bytes) ──────►
                                         Sender stops — window full

◄──── ACK 3001, Window=3000 ────────────────────  Receiver acknowledges all 3000
                                                   and says buffer is free again

Segment 4: bytes 3001-4000 ─────────────────────►
... continues
```

### Window Scaling

TCP window size field is 16 bits → maximum of **65,535 bytes** per window. For high-speed networks, this is insufficient. TCP **Window Scaling** (RFC 1323) extends this using a scale factor, allowing windows up to **1 GB**.

```
Effective Window = Window Size × 2^(Scale Factor)
Example: Window=65535, Scale=7 → 65535 × 128 = 8,388,480 bytes (~8 MB)
```

---

## TCP Congestion Control

Beyond flow control (receiver limitation), TCP also manages **network congestion** (when routers/links are overwhelmed).

### Congestion Control Phases

```
Phase 1: Slow Start
  - Begin with Congestion Window (cwnd) = 1 MSS (Maximum Segment Size)
  - Double cwnd every RTT until Slow Start Threshold (ssthresh) is reached
  - Growth is EXPONENTIAL

  cwnd: 1 → 2 → 4 → 8 → 16 → 32... (until ssthresh)

Phase 2: Congestion Avoidance
  - Once cwnd = ssthresh, grow by 1 MSS per RTT
  - Growth is LINEAR (additive increase)

  cwnd: 32 → 33 → 34 → 35...

Phase 3: Congestion Detected (packet loss)
  Option A - Timeout:
    ssthresh = cwnd / 2
    cwnd = 1 MSS
    Restart Slow Start (aggressive recovery)

  Option B - 3 Duplicate ACKs (Fast Retransmit):
    ssthresh = cwnd / 2
    cwnd = ssthresh
    Enter Congestion Avoidance directly (faster recovery)
```

---

## Choosing TCP vs UDP

Use this decision framework:

```
Does the application REQUIRE guaranteed delivery?
├── YES → Use TCP
│         Web browsing, Email, File transfers,
│         Remote access (SSH), Database queries
│
└── NO → Does the application prioritize SPEED / LOW LATENCY?
          ├── YES → Use UDP
          │         VoIP, Video streaming, Online gaming,
          │         DNS queries, DHCP, SNMP, TFTP, NTP
          │
          └── NO → Does the application implement its OWN reliability?
                   ├── YES → Use UDP
                   │         (QUIC/HTTP3, some proprietary protocols)
                   └── NO → Use TCP (default to reliable)
```

### Real-World Examples

**Why VoIP uses UDP, not TCP:**

```
VoIP (Voice over IP) call:
- Audio generates ~50 packets per second
- If one packet is lost:
  TCP: Stops everything, retransmits, causes DELAY → you hear a pause/stutter
  UDP: Skips the lost packet → you hear a tiny glitch (nearly imperceptible)

Conclusion: A tiny glitch in voice is better than a noticeable pause.
UDP is the correct choice.
```

**Why HTTP uses TCP:**

```
Loading a web page:
- Missing even 1 byte of HTML/CSS/JS breaks the page
- A small delay to retransmit is acceptable
- Correct, complete data is essential

Conclusion: Reliability matters more than speed. TCP is correct.
```

**Why DNS primarily uses UDP:**

```
DNS query:
- Request: "What is the IP for google.com?" (~50 bytes)
- Response: "142.250.80.46" (~100 bytes)

Total data is tiny. TCP handshake overhead (3 packets before data)
would be larger than the actual DNS exchange.

UDP gets the answer faster with 1 request + 1 reply.
(TCP is used for large responses or zone transfers > 512 bytes)
```

---

## Key Takeaways

- **TCP** = connection-oriented, reliable, ordered, slower — uses 3-way handshake
- **UDP** = connectionless, unreliable, fast, low overhead — no handshake
- **TCP flags:** SYN, ACK, FIN, RST, PSH, URG
- **3-way handshake:** SYN → SYN-ACK → ACK (connection setup)
- **4-way termination:** FIN → ACK → FIN → ACK (each side closes independently)
- **Flow control** uses the TCP window size to prevent receiver buffer overflow
- **Congestion control** uses Slow Start + Congestion Avoidance algorithms
- **Port ranges:** 0–1023 (well-known), 1024–49151 (registered), 49152–65535 (dynamic)
- **Key ports to memorize:** 20/21 (FTP), 22 (SSH), 23 (Telnet), 25 (SMTP), 53 (DNS), 67/68 (DHCP), 80 (HTTP), 110 (POP3), 143 (IMAP), 161 (SNMP), 443 (HTTPS), 3389 (RDP)

---

## Exam Tips

> 🎯 **High-Frequency Exam Questions:**

1. **"Which protocol provides reliable, ordered delivery?"** → **TCP**
2. **"What are the steps of the TCP 3-way handshake?"** → **SYN → SYN-ACK → ACK**
3. **"Which port does HTTPS use?"** → **443**
4. **"Which protocol would you use for VoIP?"** → **UDP** (low latency over reliability)
5. **"What port does SSH use?"** → **22**
6. **"What is the purpose of the TCP window size?"** → **Flow control** — limits how much data sender can transmit before receiving an ACK
7. **"Which port does DHCP server listen on?"** → **67 (UDP)**
8. **"What happens when a TCP host receives 3 duplicate ACKs?"** → **Fast retransmit** — retransmits missing segment without waiting for timeout

> 📝 **Trick Question Alert:** DHCP uses **UDP**, not TCP. Students often assume it uses TCP because "it needs reliability." But DHCP is a broadcast-based protocol, and broadcasts cannot use TCP. The DORA process includes retransmission if needed at the application level.

---

*← Previous: [04 — IPv6 Addressing](./04_IPv6.md)*  
*[← Back to Index](./README.md)*
