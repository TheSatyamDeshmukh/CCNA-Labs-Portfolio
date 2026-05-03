# Lab-08 – Standard ACLs

**Topic:** Access Control Lists | **Difficulty:** Intermediate | **Time:** ~35 mins

---

## 🎯 Objectives

- Create numbered and named standard ACLs
- Permit or deny traffic based on source IP address
- Apply ACLs to the correct interface and direction
- Verify ACL matches with `show ip access-lists`

---

## 🖧 Topology

```
PC-A (192.168.1.10)  ← PERMITTED
PC-B (192.168.1.20)  ← DENIED

PC-A ---+
        |
PC-B ---+--- SW1 ---Gi0/0--- R1 ---Gi0/1--- Server (10.0.0.10)

                         ACL applied outbound on Gi0/1
```

---

## 🧰 Device Table

| Device | Interface | IP Address | Role |
|--------|-----------|------------|------|
| PC-A | NIC | 192.168.1.10/24 | Permitted host |
| PC-B | NIC | 192.168.1.20/24 | Denied host |
| R1 | Gi0/0 | 192.168.1.1/24 | Gateway for LAN |
| R1 | Gi0/1 | 10.0.0.1/24 | Gateway for Server |
| Server | NIC | 10.0.0.10/24 | Destination |

---

## ⚙️ Step-by-Step Configuration

### Step 1 — Configure interfaces on R1

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address 10.0.0.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
```

### Step 2 — Create a Named Standard ACL

```
R1(config)# ip access-list standard PERMIT_PC_A
R1(config-std-nacl)# remark Allow PC-A only
R1(config-std-nacl)# permit host 192.168.1.10
R1(config-std-nacl)# deny host 192.168.1.20
R1(config-std-nacl)# deny any
R1(config-std-nacl)# exit
```

> 💡 There is an **implicit `deny any`** at the end of every ACL. The last `deny any` above makes it visible in the config.

### Step 3 — Apply ACL to interface

```
R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip access-group PERMIT_PC_A out
R1(config-if)# exit

R1(config)# end
R1# write memory
```

> 💡 **Standard ACLs** = place them **close to the destination** (outbound on the interface toward the server)

### Alternative — Numbered Standard ACL (older style)

```
R1(config)# access-list 10 permit host 192.168.1.10
R1(config)# access-list 10 deny host 192.168.1.20
R1(config)# access-list 10 deny any

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip access-group 10 out
```

---

## ✅ Verification Commands

```
R1# show ip access-lists
R1# show ip access-lists PERMIT_PC_A
R1# show ip interface GigabitEthernet0/1
R1# show running-config | include access
```

### Expected Output — `show ip access-lists`

```
Standard IP access list PERMIT_PC_A
    10 permit host 192.168.1.10 (5 matches)
    20 deny host 192.168.1.20 (3 matches)
    30 deny any
```

> The **match count** shows how many packets each rule has processed — great for troubleshooting!

---

## 🧪 Testing

```
PC-A> ping 10.0.0.10   → Should WORK ✅  (192.168.1.10 is permitted)
PC-B> ping 10.0.0.10   → Should FAIL ❌  (192.168.1.20 is denied)
```

---

## 💡 Key Concepts

| Concept | Explanation |
|---------|-------------|
| Standard ACL | Filters only on **source IP** — numbered 1-99 or 1300-1999 |
| Named ACL | Uses a name instead of a number — easier to manage |
| `permit host` | Matches exactly ONE IP address (equivalent to /32) |
| `deny any` | Blocks everything else (also implicit at end of every ACL) |
| Outbound ACL | Checks packets LEAVING the interface |
| Inbound ACL | Checks packets ENTERING the interface |

### Standard ACL Placement Rule

> **Place Standard ACLs as close to the DESTINATION as possible.**
> This is because standard ACLs only filter on source IP — if placed near the source, they might accidentally block traffic to other destinations.

---

## 📝 Review Questions

1. What is the range of numbered Standard ACLs?
2. Why place Standard ACLs close to the destination?
3. What is the implicit rule at the end of every ACL?
4. How do you remove an ACL from an interface?
5. What does `show ip access-lists` tell you that `show running-config` doesn't?

---

## ✅ Lab Completion Checklist

- [ ] R1 interfaces configured and UP
- [ ] Named ACL `PERMIT_PC_A` created with 3 rules
- [ ] ACL applied outbound on Gi0/1
- [ ] PC-A can ping Server (10.0.0.10)
- [ ] PC-B cannot ping Server
- [ ] Match counters incrementing in `show ip access-lists`

---

*[← Lab 07](./Lab-07-OSPF-MultiArea.md) | Lab 08 of 12 | [Next Lab →](./Lab-09-Extended-ACL.md) | [Back to Index](./README.md)*
