# Project-02 – ACL-Based Security Policy

**Level:** Intermediate | **Time:** ~50 mins | **Tool:** Packet Tracer

---

## 🎯 Project Objective

Ek company ke liye complete ACL security policy banao:
- HR department (VLAN 10) — only HTTP/HTTPS allowed
- IT department (VLAN 20) — full access
- Guest network (VLAN 30) — only internet, no internal access
- All Telnet access blocked company-wide
- All SSH only from IT department allowed
- Reflexive ACL — return traffic automatically allowed

---

## 🖧 Network Topology

```
                    Internet
                       |
                   Gi0/1 (203.0.113.1)
                       |
                   [ R1 — Gateway ]
                       |
                   Gi0/0.10 → VLAN 10 → HR (192.168.10.0/24)
                   Gi0/0.20 → VLAN 20 → IT (192.168.20.0/24)
                   Gi0/0.30 → VLAN 30 → Guest (192.168.30.0/24)

HR PC:    192.168.10.10
IT PC:    192.168.20.10
Guest PC: 192.168.30.10
Server:   10.0.0.10 (internal)
```

---

## 🧰 Device & IP Table

| Device | IP | Department | Access Level |
|--------|----|------------|--------------|
| HR-PC | 192.168.10.10 | HR | HTTP/HTTPS only |
| IT-PC | 192.168.20.10 | IT | Full access |
| Guest-PC | 192.168.30.10 | Guest | Internet only |
| Internal Server | 10.0.0.10 | — | SSH from IT only |

---

## ⚙️ Step-by-Step Configuration

### Step 1 — Configure Subinterfaces (Router-on-a-Stick)

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# no shutdown
R1(config-if)# exit

R1(config)# interface GigabitEthernet0/0.10
R1(config-subif)# encapsulation dot1Q 10
R1(config-subif)# ip address 192.168.10.1 255.255.255.0
R1(config-subif)# exit

R1(config)# interface GigabitEthernet0/0.20
R1(config-subif)# encapsulation dot1Q 20
R1(config-subif)# ip address 192.168.20.1 255.255.255.0
R1(config-subif)# exit

R1(config)# interface GigabitEthernet0/0.30
R1(config-subif)# encapsulation dot1Q 30
R1(config-subif)# ip address 192.168.30.1 255.255.255.0
R1(config-subif)# exit

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address 203.0.113.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
```

---

### Step 2 — HR Department ACL (VLAN 10)

HR sirf HTTP, HTTPS, DNS allow — baaki sab block:

```
R1(config)# ip access-list extended HR-POLICY
R1(config-ext-nacl)# remark === HR Department — Limited Access ===
R1(config-ext-nacl)# permit tcp 192.168.10.0 0.0.0.255 any eq 80
R1(config-ext-nacl)# permit tcp 192.168.10.0 0.0.0.255 any eq 443
R1(config-ext-nacl)# permit udp 192.168.10.0 0.0.0.255 any eq 53
R1(config-ext-nacl)# permit icmp 192.168.10.0 0.0.0.255 any
R1(config-ext-nacl)# deny tcp 192.168.10.0 0.0.0.255 any eq 23
R1(config-ext-nacl)# deny tcp 192.168.10.0 0.0.0.255 any eq 22
R1(config-ext-nacl)# deny ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
R1(config-ext-nacl)# deny ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
R1(config-ext-nacl)# permit ip 192.168.10.0 0.0.0.255 any
R1(config-ext-nacl)# exit

R1(config)# interface GigabitEthernet0/0.10
R1(config-subif)# ip access-group HR-POLICY in
R1(config-subif)# exit
```

---

### Step 3 — IT Department ACL (VLAN 20)

IT ko full access — sirf Telnet block:

```
R1(config)# ip access-list extended IT-POLICY
R1(config-ext-nacl)# remark === IT Department — Full Access except Telnet ===
R1(config-ext-nacl)# deny tcp 192.168.20.0 0.0.0.255 any eq 23
R1(config-ext-nacl)# permit ip 192.168.20.0 0.0.0.255 any
R1(config-ext-nacl)# exit

R1(config)# interface GigabitEthernet0/0.20
R1(config-subif)# ip access-group IT-POLICY in
R1(config-subif)# exit
```

---

### Step 4 — Guest Network ACL (VLAN 30)

Guest ko only internet — internal network bilkul nahi:

```
R1(config)# ip access-list extended GUEST-POLICY
R1(config-ext-nacl)# remark === Guest — Internet Only ===
R1(config-ext-nacl)# deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
R1(config-ext-nacl)# deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
R1(config-ext-nacl)# deny ip 192.168.30.0 0.0.0.255 10.0.0.0 0.0.0.255
R1(config-ext-nacl)# permit tcp 192.168.30.0 0.0.0.255 any eq 80
R1(config-ext-nacl)# permit tcp 192.168.30.0 0.0.0.255 any eq 443
R1(config-ext-nacl)# permit udp 192.168.30.0 0.0.0.255 any eq 53
R1(config-ext-nacl)# deny ip any any log
R1(config-ext-nacl)# exit

R1(config)# interface GigabitEthernet0/0.30
R1(config-subif)# ip access-group GUEST-POLICY in
R1(config-subif)# exit
```

---

### Step 5 — Reflexive ACL (Stateful — return traffic auto-allow)

```
R1(config)# ip access-list extended OUTBOUND-INTERNET
R1(config-ext-nacl)# permit tcp 192.168.10.0 0.0.0.255 any reflect REFLECT-SESSIONS
R1(config-ext-nacl)# permit tcp 192.168.20.0 0.0.0.255 any reflect REFLECT-SESSIONS
R1(config-ext-nacl)# permit udp 192.168.10.0 0.0.0.255 any reflect REFLECT-SESSIONS
R1(config-ext-nacl)# exit

R1(config)# ip access-list extended INBOUND-INTERNET
R1(config-ext-nacl)# evaluate REFLECT-SESSIONS
R1(config-ext-nacl)# deny ip any any log
R1(config-ext-nacl)# exit

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip access-group OUTBOUND-INTERNET out
R1(config-if)# ip access-group INBOUND-INTERNET in
R1(config-if)# exit

R1# write memory
```

---

## ✅ Verification Commands

```
R1# show ip access-lists
R1# show ip access-lists HR-POLICY
R1# show ip access-lists GUEST-POLICY
R1# show ip interface GigabitEthernet0/0.10
R1# show ip interface GigabitEthernet0/1
```

---

## 🧪 Testing Matrix

| Source | Destination | Port | Expected |
|--------|-------------|------|---------|
| HR-PC | Internet | 80 | ✅ PASS |
| HR-PC | Internet | 443 | ✅ PASS |
| HR-PC | IT-PC | any | ❌ DENY |
| HR-PC | Any | 23 (Telnet) | ❌ DENY |
| IT-PC | Any | any | ✅ PASS |
| IT-PC | Any | 23 (Telnet) | ❌ DENY |
| Guest-PC | Internet | 80 | ✅ PASS |
| Guest-PC | HR-PC | any | ❌ DENY |
| Guest-PC | Internal Server | any | ❌ DENY |

---

## 💡 Key Concepts

| Concept | Explanation |
|---------|-------------|
| Extended ACL | Src IP + Dst IP + Protocol + Port filter karta hai |
| Named ACL | Number ki jagah naam use karna — edit karna easy hota hai |
| `remark` | ACL mein comment add karne ka command — documentation ke liye |
| Reflexive ACL | `reflect` keyword se stateful session tracking hoti hai |
| `evaluate` | Reflected sessions ko inbound direction mein allow karta hai |
| `log` keyword | Matching packets ko syslog mein record karta hai |
| ACL direction | `in` = entering interface, `out` = leaving interface |

---

## 🎤 Interview Questions & Answers

**Q1: Reflexive ACL kya hoti hai aur normal ACL se kaise different hai?**
> Reflexive ACL stateful hoti hai — jab outbound traffic jaati hai `reflect` keyword se ek temporary entry banti hai. Inbound direction mein `evaluate` us entry ko check karke return traffic allow karta hai. Normal ACL stateless hoti hai — dono directions manually configure karne padte hain.

**Q2: Named ACL aur numbered ACL mein kya practical difference hai?**
> Named ACL mein tum kisi bhi position pe entry add ya delete kar sakte ho sequence numbers se, bina puri ACL delete kiye. Numbered ACL mein editing ke liye puri ACL delete karni padti thi pehle. Production mein hamesha named ACLs use karo.

**Q3: `log` keyword ke side effects kya hain?**
> `log` keyword CPU usage badhata hai kyunki har matching packet ke liye syslog message generate hota hai. High-traffic environments mein sirf specific suspicious entries pe log lagao, `deny ip any any log` ke sath dhyan rakho.

**Q4: ACL sequence numbers kaise kaam karte hain?**
> Named ACL mein har entry ka ek sequence number hota hai (10, 20, 30...). Agar tumhe 10 aur 20 ke beech entry add karni ho, `15 permit...` likhte ho — woh automatically sahi position pe aa jaati hai.

**Q5: Guest network ko internal access se kyun block karte hain?**
> Guest network untrusted hai — visitors ke devices pe malware ho sakta hai. Agar guest network internal LAN se connected ho, attacker guest device se internal servers access kar sakta hai. ACL se explicit block karna defense-in-depth strategy ka hissa hai.

---

## ✅ Project Completion Checklist

- [ ] Subinterfaces configured for VLAN 10, 20, 30
- [ ] HR-POLICY ACL applied on Gi0/0.10 inbound
- [ ] IT-POLICY ACL applied on Gi0/0.20 inbound
- [ ] GUEST-POLICY ACL applied on Gi0/0.30 inbound
- [ ] Reflexive ACL on Gi0/1 (in + out)
- [ ] HR-PC can access internet HTTP/HTTPS
- [ ] HR-PC cannot reach IT-PC (denied)
- [ ] Guest-PC cannot reach any internal IP
- [ ] Telnet blocked for all users
- [ ] `show ip access-lists` shows match counts

---

*[← Project 01](./Project-01-ZBF.md) | [Back to Index](./README.md) | [Next Project →](./Project-03-IPsec-VPN.md)*
