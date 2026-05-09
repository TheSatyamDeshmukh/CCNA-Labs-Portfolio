# Project-01 – Zone-Based Firewall (ZBF)

**Level:** Intermediate | **Time:** ~60 mins | **Tool:** Packet Tracer / GNS3

---

## 🎯 Project Objective

Cisco router ko ek proper firewall ki tarah configure karna — Inside (LAN), Outside (Internet), aur DMZ (Servers) zones banao. Traffic policies define karo jo:
- Inside → Outside: HTTP, HTTPS, DNS allow karo
- Inside → DMZ: HTTP allow karo
- Outside → DMZ: HTTP, HTTPS allow karo
- Outside → Inside: Sab block karo
- DMZ → Inside: Sab block karo

---

## 🖧 Network Topology

```
Internet (Outside Zone)
        |
      Gi0/0 (203.0.113.1)
        |
     [ R1 — ZBF Router ]
        |              |
      Gi0/1          Gi0/2
  (192.168.1.1)   (10.0.0.1)
        |              |
  Inside Zone      DMZ Zone
  (LAN Users)    (Web Server)
  192.168.1.0/24  10.0.0.0/24

PC-A: 192.168.1.10  (Inside)
PC-B: 192.168.1.20  (Inside)
Web Server: 10.0.0.10  (DMZ)
Internet Router: 203.0.113.254  (Outside)
```

---

## 🧰 Device & IP Table

| Device | Interface | IP Address | Zone |
|--------|-----------|------------|------|
| R1 | Gi0/0 | 203.0.113.1/24 | Outside |
| R1 | Gi0/1 | 192.168.1.1/24 | Inside |
| R1 | Gi0/2 | 10.0.0.1/24 | DMZ |
| PC-A | NIC | 192.168.1.10/24 | Inside |
| PC-B | NIC | 192.168.1.20/24 | Inside |
| Web Server | NIC | 10.0.0.10/24 | DMZ |

---

## ⚙️ Step-by-Step Configuration

### Step 1 — Configure Interfaces on R1

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 203.0.113.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit

R1(config)# interface GigabitEthernet0/2
R1(config-if)# ip address 10.0.0.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
```

---

### Step 2 — Create Security Zones

```
R1(config)# zone security INSIDE
R1(config-sec-zone)# description LAN Users Zone
R1(config-sec-zone)# exit

R1(config)# zone security OUTSIDE
R1(config-sec-zone)# description Internet Zone
R1(config-sec-zone)# exit

R1(config)# zone security DMZ
R1(config-sec-zone)# description Server Zone
R1(config-sec-zone)# exit
```

---

### Step 3 — Create Class Maps (what traffic to match)

```
R1(config)# class-map type inspect match-any INSIDE-PROTOCOLS
R1(config-cmap)# match protocol http
R1(config-cmap)# match protocol https
R1(config-cmap)# match protocol dns
R1(config-cmap)# match protocol icmp
R1(config-cmap)# exit

R1(config)# class-map type inspect match-any DMZ-PROTOCOLS
R1(config-cmap)# match protocol http
R1(config-cmap)# match protocol https
R1(config-cmap)# exit

R1(config)# class-map type inspect match-any INSIDE-TO-DMZ
R1(config-cmap)# match protocol http
R1(config-cmap)# exit
```

---

### Step 4 — Create Policy Maps (what action to take)

```
R1(config)# policy-map type inspect INSIDE-TO-OUTSIDE-POLICY
R1(config-pmap)# class type inspect INSIDE-PROTOCOLS
R1(config-pmap-c)# inspect
R1(config-pmap-c)# exit
R1(config-pmap)# class class-default
R1(config-pmap-c)# drop
R1(config-pmap-c)# exit
R1(config-pmap)# exit

R1(config)# policy-map type inspect INSIDE-TO-DMZ-POLICY
R1(config-pmap)# class type inspect INSIDE-TO-DMZ
R1(config-pmap-c)# inspect
R1(config-pmap-c)# exit
R1(config-pmap)# class class-default
R1(config-pmap-c)# drop
R1(config-pmap-c)# exit
R1(config-pmap)# exit

R1(config)# policy-map type inspect OUTSIDE-TO-DMZ-POLICY
R1(config-pmap)# class type inspect DMZ-PROTOCOLS
R1(config-pmap-c)# inspect
R1(config-pmap-c)# exit
R1(config-pmap)# class class-default
R1(config-pmap-c)# drop
R1(config-pmap-c)# exit
R1(config-pmap)# exit
```

---

### Step 5 — Create Zone Pairs (direction matters!)

```
R1(config)# zone-pair security INSIDE-TO-OUTSIDE source INSIDE destination OUTSIDE
R1(config-sec-zone-pair)# service-policy type inspect INSIDE-TO-OUTSIDE-POLICY
R1(config-sec-zone-pair)# exit

R1(config)# zone-pair security INSIDE-TO-DMZ source INSIDE destination DMZ
R1(config-sec-zone-pair)# service-policy type inspect INSIDE-TO-DMZ-POLICY
R1(config-sec-zone-pair)# exit

R1(config)# zone-pair security OUTSIDE-TO-DMZ source OUTSIDE destination DMZ
R1(config-sec-zone-pair)# service-policy type inspect OUTSIDE-TO-DMZ-POLICY
R1(config-sec-zone-pair)# exit
```

> ⚠️ **Important:** OUTSIDE-TO-INSIDE zone-pair nahi banaya — matlab outside se inside koi bhi traffic block hai by default!

---

### Step 6 — Assign Interfaces to Zones

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# zone-member security OUTSIDE
R1(config-if)# exit

R1(config)# interface GigabitEthernet0/1
R1(config-if)# zone-member security INSIDE
R1(config-if)# exit

R1(config)# interface GigabitEthernet0/2
R1(config-if)# zone-member security DMZ
R1(config-if)# exit

R1(config)# end
R1# write memory
```

---

## ✅ Verification Commands

```
R1# show zone security
R1# show zone-pair security
R1# show policy-map type inspect zone-pair
R1# show class-map type inspect
R1# show policy-map type inspect zone-pair sessions
```

### Expected Output — `show zone security`

```
zone self
  Description: System defined zone

zone INSIDE
  Description: LAN Users Zone
  Member Interfaces:
    GigabitEthernet0/1

zone OUTSIDE
  Description: Internet Zone
  Member Interfaces:
    GigabitEthernet0/0

zone DMZ
  Description: Server Zone
  Member Interfaces:
    GigabitEthernet0/2
```

### Expected Output — `show zone-pair security`

```
Zone-pair name INSIDE-TO-OUTSIDE
    Source-Zone INSIDE  Destination-Zone OUTSIDE
    service-policy INSIDE-TO-OUTSIDE-POLICY

Zone-pair name INSIDE-TO-DMZ
    Source-Zone INSIDE  Destination-Zone DMZ
    service-policy INSIDE-TO-DMZ-POLICY

Zone-pair name OUTSIDE-TO-DMZ
    Source-Zone OUTSIDE  Destination-Zone DMZ
    service-policy OUTSIDE-TO-DMZ-POLICY
```

---

## 🧪 Testing

```
PC-A> ping 203.0.113.1        → WORK ✅  (Inside to Outside ICMP)
PC-A> ping 10.0.0.10          → WORK ✅  (Inside to DMZ)

From Outside router:
> ping 192.168.1.10           → FAIL ❌  (Outside to Inside — blocked!)
> ping 10.0.0.10              → WORK ✅  (Outside to DMZ — allowed)

From DMZ Web Server:
> ping 192.168.1.10           → FAIL ❌  (DMZ to Inside — blocked!)
```

---

## 💡 Key Concepts

| Concept | Explanation |
|---------|-------------|
| Security Zone | Logical grouping of interfaces with same security level |
| Zone-Pair | Directional relationship between two zones — INSIDE→OUTSIDE ≠ OUTSIDE→INSIDE |
| Class-Map | Defines WHAT traffic to match (protocol, ACL, etc.) |
| Policy-Map | Defines WHAT ACTION to take (inspect, drop, pass) |
| Service-Policy | Binds policy-map to a zone-pair |
| inspect | Stateful tracking — return traffic auto-allowed |
| drop | Silently discard traffic |
| pass | Allow without stateful tracking (not recommended for internet) |
| Self Zone | The router itself — SSH/OSPF/management traffic uses this |

### ZBF vs Classic Firewall ACL

| Feature | Classic ACL | Zone-Based Firewall |
|---------|-------------|---------------------|
| Stateful | ❌ No | ✅ Yes |
| Direction control | Per interface | Per zone-pair |
| Default behavior | Permit all | Deny between zones |
| Complexity | Simple | More structured |
| Enterprise use | Basic | Recommended |

---

## 🎤 Interview Questions & Answers

**Q1: ZBF mein zone-pair ka kya matlab hai?**
> Zone-pair ek directional relationship define karta hai — source zone se destination zone tak traffic ka behavior. Ek zone-pair sirf ek direction mein kaam karta hai. INSIDE-to-OUTSIDE aur OUTSIDE-to-INSIDE ke liye alag-alag zone-pairs banane padte hain.

**Q2: `inspect` aur `pass` keyword mein kya difference hai?**
> `inspect` stateful packet inspection karta hai — jab inside se connection jaati hai, return traffic automatically allow hoti hai without needing a reverse rule. `pass` bina stateful tracking ke allow karta hai — return traffic block ho sakti hai.

**Q3: Agar koi interface kisi zone mein assign nahi hai to kya hoga?**
> Agar interface kisi zone mein nahi hai, to us interface se traffic kisi bhi zone ke saath communicate nahi kar sakti — by default blocked. Isliye har interface ko explicitly zone-member assign karna zaroori hai.

**Q4: Self zone kab use karte hain?**
> Jab router khud ek destination ho — jaise SSH management, OSPF hello packets, SNMP, NTP. In cases mein INSIDE-to-SELF zone-pair banate hain specific protocols ke liye.

**Q5: ZBF aur classic ACL firewall mein kaun better hai aur kyun?**
> ZBF better hai kyunki: stateful tracking hai (return traffic auto-allow), default deny-all policy hai between zones, centralized policy management hai class-map/policy-map se, aur troubleshooting easy hai show commands se.

---

## ✅ Project Completion Checklist

- [ ] Teeno interfaces configured aur UP
- [ ] Teeno zones created (INSIDE, OUTSIDE, DMZ)
- [ ] Class-maps traffic ke liye defined
- [ ] Policy-maps inspect/drop actions ke saath
- [ ] Zone-pairs assigned (3 zone-pairs)
- [ ] Interfaces zones mein assign kiye
- [ ] PC-A → Outside ping kaam kar raha hai
- [ ] Outside → Inside ping band hai
- [ ] DMZ → Inside ping band hai
- [ ] `show zone-pair security` sab dikhata hai

---

*[Back to Security Projects Index](./README.md) | [Next Project →](./Project-02-ACL-Policy.md)*
