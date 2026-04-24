# 03 — IP Addressing: IPv4, Subnetting & CIDR

> **CCNA Exam Domain:** 1.0 Network Fundamentals  
> **Exam Weight:** ~20% of total exam  
> **Difficulty:** ⭐⭐⭐⭐☆ (Most challenging fundamental topic)

---

## Table of Contents
- [IPv4 Address Structure](#ipv4-address-structure)
- [Address Classes](#address-classes)
- [Private vs Public IP Addresses](#private-vs-public-ip-addresses)
- [Special IP Addresses](#special-ip-addresses)
- [Subnet Mask Fundamentals](#subnet-mask-fundamentals)
- [CIDR Notation](#cidr-notation)
- [Subnetting — Step-by-Step](#subnetting--step-by-step)
- [VLSM — Variable Length Subnet Masking](#vlsm--variable-length-subnet-masking)
- [Subnetting Cheat Sheet](#subnetting-cheat-sheet)
- [Key Takeaways](#key-takeaways)
- [Exam Tips](#exam-tips)

---

## IPv4 Address Structure

An **IPv4 address** is a **32-bit binary number** divided into four 8-bit groups called **octets**, written in **dotted-decimal notation**.

```
Binary:   11000000 . 10101000 . 00000001 . 00001010
Decimal:     192   .   168   .    1     .    10
              ↑         ↑          ↑          ↑
           Octet 1   Octet 2    Octet 3    Octet 4
           (8 bits)  (8 bits)   (8 bits)   (8 bits)
                    ←————————— 32 bits total ————————→
```

### Binary-Decimal Conversion

Each octet has 8 bit positions with values:

```
Bit Position:  128  64  32  16   8   4   2   1
Binary:          1   1   0   0   0   0   0   0
                 ↓   ↓
              128 + 64 = 192
```

**Quick Reference — All 256 values (0–255):**

| Decimal | Binary | | Decimal | Binary |
|---------|--------|-|---------|--------|
| 0 | 00000000 | | 128 | 10000000 |
| 1 | 00000001 | | 192 | 11000000 |
| 127 | 01111111 | | 224 | 11100000 |
| 255 | 11111111 | | 240 | 11110000 |

> 💡 **Tip:** Memorize powers of 2: 1, 2, 4, 8, 16, 32, 64, 128. These are the backbone of binary-to-decimal conversion.

---

## Address Classes

IPv4 originally used **classful addressing** — where the first few bits determine the class (and therefore the default subnet mask).

```
Class  | First Octet Range | Default Mask     | Network/Host bits | # Networks  | Hosts/Network
-------|-------------------|------------------|-------------------|-------------|---------------
  A    |    1  – 126       | 255.0.0.0   /8   |   8 net / 24 host | 126         | 16,777,214
  B    |  128  – 191       | 255.255.0.0 /16  |  16 net / 16 host | 16,384      | 65,534
  C    |  192  – 223       | 255.255.255.0/24 |  24 net /  8 host | 2,097,152   | 254
  D    |  224  – 239       | N/A              | Multicast         | N/A         | N/A
  E    |  240  – 255       | N/A              | Experimental      | N/A         | N/A
```

### Class A Example

```
Address:   10.50.100.200
Class:     A (starts with 10)
Default:   255.0.0.0 (/8)
Network:   10.0.0.0
Host part: 50.100.200
Host range: 10.0.0.1 – 10.255.255.254
Broadcast: 10.255.255.255
```

### Class B Example

```
Address:   172.16.50.25
Class:     B (starts with 172)
Default:   255.255.0.0 (/16)
Network:   172.16.0.0
Host part: 50.25
Host range: 172.16.0.1 – 172.16.255.254
Broadcast: 172.16.255.255
```

### Class C Example

```
Address:   192.168.1.100
Class:     C (starts with 192)
Default:   255.255.255.0 (/24)
Network:   192.168.1.0
Host part: 100
Host range: 192.168.1.1 – 192.168.1.254
Broadcast: 192.168.1.255
```

> ⚠️ **Note:** 127.x.x.x is reserved for **loopback** (127.0.0.1 = "myself"). It falls between Class A and B but is not assigned to either.

---

## Private vs Public IP Addresses

**Private IP addresses** are reserved for internal use and are **NOT routable on the internet**. Devices using private IPs must use **NAT (Network Address Translation)** to communicate with the internet.

**RFC 1918 — Private Address Ranges:**

| Class | Private Range | CIDR | # of Addresses |
|-------|--------------|------|----------------|
| A | 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | 16,777,216 |
| B | 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | 1,048,576 |
| C | 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | 65,536 |

> 💡 **Real-World Example:** Your home router gets one **public IP** from your ISP. All devices in your home use **private IPs** (e.g., 192.168.1.x). The router NATs your traffic to the single public IP when you access the internet.

---

## Special IP Addresses

| Address / Range | Purpose |
|----------------|---------|
| 0.0.0.0 | "This host" — used by DHCP clients before obtaining IP |
| 127.0.0.1 | Loopback address — refers to the local device itself |
| 127.0.0.0/8 | Full loopback range |
| 169.254.0.0/16 | APIPA (Automatic Private IP Addressing) — when DHCP fails |
| 255.255.255.255 | Limited broadcast — sent to all hosts on local segment |
| x.x.x.255 | Directed broadcast — sent to all hosts on specific subnet |
| 224.0.0.0/4 | Multicast range (Class D) |
| 240.0.0.0/4 | Experimental (Class E) |

### APIPA — What Is It?

When a device **cannot reach a DHCP server**, Windows automatically assigns itself an IP from the **169.254.0.0/16** range. If you see a device with a 169.254.x.x address, it means **DHCP failed** — the device can only communicate with other APIPA devices on the same segment.

---

## Subnet Mask Fundamentals

A **subnet mask** tells a device which part of an IP address is the **network portion** and which is the **host portion**.

- **Binary 1s** in the mask = Network bits
- **Binary 0s** in the mask = Host bits

```
IP Address:  192.168.1.100
             11000000.10101000.00000001.01100100

Subnet Mask: 255.255.255.0
             11111111.11111111.11111111.00000000
             |←————— Network (24 bits) ————→|←Host→|

Network:     192.168.1.0    (host bits all = 0)
Broadcast:   192.168.1.255  (host bits all = 1)
Host Range:  192.168.1.1 – 192.168.1.254
```

**AND Operation — How routers determine network address:**

Routers perform a **bitwise AND** of the IP address and subnet mask to find the network address.

```
IP:   192.168.1.100 = 11000000.10101000.00000001.01100100
Mask: 255.255.255.0 = 11111111.11111111.11111111.00000000
AND:                = 11000000.10101000.00000001.00000000
Result:               192     . 168    . 1      . 0
                    = 192.168.1.0 (the network address)
```

---

## CIDR Notation

**CIDR (Classless Inter-Domain Routing)** notation represents the subnet mask as a **prefix length** — the number of consecutive 1s in the mask.

```
255.255.255.0  =  /24  (24 ones: 11111111.11111111.11111111.00000000)
255.255.0.0    =  /16  (16 ones: 11111111.11111111.00000000.00000000)
255.0.0.0      =  /8   ( 8 ones: 11111111.00000000.00000000.00000000)
255.255.255.128 = /25  (25 ones: 11111111.11111111.11111111.10000000)
255.255.255.192 = /26  (26 ones: 11111111.11111111.11111111.11000000)
255.255.255.224 = /27  (27 ones: 11111111.11111111.11111111.11100000)
255.255.255.240 = /28  (28 ones: 11111111.11111111.11111111.11110000)
255.255.255.248 = /29  (29 ones: 11111111.11111111.11111111.11111000)
255.255.255.252 = /30  (30 ones: 11111111.11111111.11111111.11111100)
```

**Formula for hosts per subnet:**

```
Usable Hosts = 2^(host bits) - 2

Subtract 2 because:
  - Network address (all host bits = 0) — cannot be assigned
  - Broadcast address (all host bits = 1) — cannot be assigned
```

---

## Subnetting — Step-by-Step

### The Magic Number Method

The **magic number** is the **block size** of the subnet. It determines where subnets start and end.

```
Magic Number = 256 - Interesting Octet Value of Subnet Mask

Example: Mask 255.255.255.192 (/26)
→ Interesting octet = 4th (192)
→ Magic number = 256 - 192 = 64

Subnets start at multiples of 64:
  Subnet 1: 192.168.1.0   → .63  (broadcast)
  Subnet 2: 192.168.1.64  → .127 (broadcast)
  Subnet 3: 192.168.1.128 → .191 (broadcast)
  Subnet 4: 192.168.1.192 → .255 (broadcast)
```

### Worked Example 1 — /26 Subnetting

**Problem:** Subnet 192.168.10.0/24 into /26 subnets. List all subnets, ranges, and broadcast addresses.

**Step 1:** Find host bits → /26 means 32-26 = **6 host bits**  
**Step 2:** Hosts per subnet → 2^6 - 2 = **62 usable hosts**  
**Step 3:** Number of subnets → 2^(26-24) = 2^2 = **4 subnets**  
**Step 4:** Block size → 256 - 192 = **64**

```
Subnet # | Network Address  | Host Range                | Broadcast
---------|------------------|---------------------------|------------------
    1    | 192.168.10.0/26  | 192.168.10.1 – .62        | 192.168.10.63
    2    | 192.168.10.64/26 | 192.168.10.65 – .126       | 192.168.10.127
    3    | 192.168.10.128/26| 192.168.10.129 – .190      | 192.168.10.191
    4    | 192.168.10.192/26| 192.168.10.193 – .254      | 192.168.10.255
```

---

### Worked Example 2 — /27 Subnetting

**Problem:** How many subnets and hosts does 172.16.0.0/27 provide from the base /16?

**Step 1:** Host bits → 32 - 27 = **5 host bits**  
**Step 2:** Hosts per subnet → 2^5 - 2 = **30 usable hosts**  
**Step 3:** Block size → 256 - 224 = **32**  

**Subnets in the first /24 block (172.16.0.x):**

```
172.16.0.0/27    → Hosts: 172.16.0.1 – .30    → Broadcast: 172.16.0.31
172.16.0.32/27   → Hosts: 172.16.0.33 – .62   → Broadcast: 172.16.0.63
172.16.0.64/27   → Hosts: 172.16.0.65 – .94   → Broadcast: 172.16.0.95
172.16.0.96/27   → Hosts: 172.16.0.97 – .126  → Broadcast: 172.16.0.127
172.16.0.128/27  → Hosts: 172.16.0.129 – .158 → Broadcast: 172.16.0.159
172.16.0.160/27  → Hosts: 172.16.0.161 – .190 → Broadcast: 172.16.0.191
172.16.0.192/27  → Hosts: 172.16.0.193 – .222 → Broadcast: 172.16.0.223
172.16.0.224/27  → Hosts: 172.16.0.225 – .254 → Broadcast: 172.16.0.255
(then continues 172.16.1.0/27, etc.)
```

---

### Finding Which Subnet an IP Belongs To

**Problem:** Which subnet does **192.168.5.77/26** belong to?

```
/26 → block size = 64
Subnets: .0, .64, .128, .192

77 falls between 64 and 128:
→ Network:   192.168.5.64
→ Broadcast: 192.168.5.127
→ Valid Host? Yes (77 is between 65 and 126)
```

---

## VLSM — Variable Length Subnet Masking

**VLSM** allows you to use **different subnet mask sizes** within the same network — allocating just the right number of addresses for each subnet rather than wasting them.

### VLSM Example

**Scenario:** You have 192.168.1.0/24 and need:
- Subnet A: 100 hosts (Sales)
- Subnet B: 50 hosts (Engineering)
- Subnet C: 25 hosts (Management)
- Subnet D: 2 hosts (Router-to-Router link)

**Approach — Largest to Smallest:**

```
Subnet A — 100 hosts needed:
  Need 2^n - 2 ≥ 100 → n=7 → 2^7-2 = 126 hosts ✓
  Use /25 (block 128)
  Assign: 192.168.1.0/25
  Hosts:  192.168.1.1 – .126
  Broadcast: 192.168.1.127
  Remaining space: 192.168.1.128/25

Subnet B — 50 hosts needed:
  Need 2^n - 2 ≥ 50 → n=6 → 2^6-2 = 62 hosts ✓
  Use /26 (block 64)
  Assign: 192.168.1.128/26
  Hosts:  192.168.1.129 – .190
  Broadcast: 192.168.1.191
  Remaining: 192.168.1.192/26

Subnet C — 25 hosts needed:
  Need 2^n - 2 ≥ 25 → n=5 → 2^5-2 = 30 hosts ✓
  Use /27 (block 32)
  Assign: 192.168.1.192/27
  Hosts:  192.168.1.193 – .222
  Broadcast: 192.168.1.223
  Remaining: 192.168.1.224/27

Subnet D — 2 hosts (point-to-point link):
  Need 2^n - 2 ≥ 2 → n=2 → 2^2-2 = 2 hosts ✓
  Use /30 (block 4) — most efficient for p2p links!
  Assign: 192.168.1.224/30
  Hosts:  192.168.1.225 – .226
  Broadcast: 192.168.1.227
```

> 💡 **Point-to-Point Links:** Always use **/30** for router-to-router links — provides exactly 2 usable IPs with minimal waste. In modern designs, **/31** is also used (RFC 3021) with no broadcast needed.

---

## Subnetting Cheat Sheet

### Quick Prefix Reference Table

| CIDR | Subnet Mask | Block Size | Hosts/Subnet | # Subnets from /24 |
|------|-------------|------------|--------------|---------------------|
| /24 | 255.255.255.0 | 256 | 254 | 1 |
| /25 | 255.255.255.128 | 128 | 126 | 2 |
| /26 | 255.255.255.192 | 64 | 62 | 4 |
| /27 | 255.255.255.224 | 32 | 30 | 8 |
| /28 | 255.255.255.240 | 16 | 14 | 16 |
| /29 | 255.255.255.248 | 8 | 6 | 32 |
| /30 | 255.255.255.252 | 4 | 2 | 64 |
| /31 | 255.255.255.254 | 2 | 0* | 128 |
| /32 | 255.255.255.255 | 1 | 1 (host route) | 256 |

> **/31** — Special case per RFC 3021. Used on point-to-point links where broadcast concept doesn't apply. No subtraction of 2 needed.  
> **/32** — Host route. Identifies a single specific host.

### Powers of 2 Reference

| 2^n | Value | Common Use |
|-----|-------|-----------|
| 2^1 | 2 | /30 usable hosts |
| 2^2 | 4 | /30 block size |
| 2^3 | 8 | /29 block size |
| 2^4 | 16 | /28 block size |
| 2^5 | 32 | /27 block size |
| 2^6 | 64 | /26 block size |
| 2^7 | 128 | /25 block size |
| 2^8 | 256 | /24 block size |

---

## Key Takeaways

- IPv4 addresses are **32-bit** numbers written in **dotted-decimal** notation
- **Classful** ranges: A (1-126), B (128-191), C (192-223), D (224-239 multicast), E (240-255 experimental)
- **Private ranges** (RFC 1918): 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
- **127.0.0.1** = loopback; **169.254.x.x** = APIPA (DHCP failure)
- Subnet mask divides IP into **network** and **host** portions
- **Usable hosts** = 2^(host bits) - 2
- **Block size** = 256 - interesting octet value
- **VLSM** allows different mask sizes — assign largest subnets first
- Use **/30** for point-to-point links (2 usable IPs)

---

## Exam Tips

> 🎯 **High-Frequency Exam Questions:**

1. **"How many usable hosts does a /28 provide?"** → **14** (2^4 - 2)
2. **"What is the subnet mask for /26?"** → **255.255.255.192**
3. **"Which subnet does 10.0.0.200/28 belong to?"** → Block=16; 192 ÷ 16 = 12 → **10.0.0.192/28** (range .193–.206, broadcast .207)
4. **"What is the broadcast address of 172.16.5.64/26?"** → Block=64; next subnet starts at .128; broadcast = **.127**
5. **"Which private range is 172.20.0.1 in?"** → **172.16.0.0/12** (172.16–172.31)
6. **"What does APIPA indicate?"** → **DHCP failure** — device assigned 169.254.x.x

> 📝 **Speed Tip:** Practice the "block size method" until it's automatic. For any prefix /25-/30, you should be able to instantly recall block size, hosts per subnet, and the subnet boundaries within 30 seconds.

---

*← Previous: [02 — TCP/IP Model](./02_TCPIP_Model.md)*  
*Next: [04 — IPv6 Addressing →](./04_IPv6.md)*
