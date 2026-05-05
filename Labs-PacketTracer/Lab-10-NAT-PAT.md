# Lab-10 – NAT & PAT (Overload)

**Topic:** NAT / PAT | **Difficulty:** Intermediate | **Time:** ~40 mins

---

## 🎯 Objectives

- Configure Static NAT for a server with a fixed public IP
- Configure Dynamic NAT for a pool of public addresses
- Configure PAT (Overload) so many private IPs share one public IP
- Verify translations with `show ip nat translations`

---

## 🖧 Topology

```
Private Network (Inside)          Public Network (Outside)
192.168.1.0/24                    203.0.113.0/24

PC-A (192.168.1.10) ---+
PC-B (192.168.1.20) ---+--- R1 ---Gi0/1--- Internet
Web-Srv (192.168.1.100)+   [NAT]     203.0.113.1 (Public IP)

Inside: Gi0/0 (192.168.1.1)
Outside: Gi0/1 (203.0.113.1)
```

---

## 🧰 Device Table

| Device | Private IP | Public IP (after NAT) | Type |
|--------|------------|----------------------|------|
| PC-A | 192.168.1.10 | 203.0.113.1 (PAT) | Dynamic PAT |
| PC-B | 192.168.1.20 | 203.0.113.1 (PAT) | Dynamic PAT |
| Web-Srv | 192.168.1.100 | 203.0.113.100 | Static NAT |

---

## ⚙️ Step-by-Step Configuration

### Step 1 — Configure R1 interfaces

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# ip nat inside
R1(config-if)# no shutdown
R1(config-if)# exit

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address 203.0.113.1 255.255.255.0
R1(config-if)# ip nat outside
R1(config-if)# no shutdown
R1(config-if)# exit
```

> ⚠️ **Critical step:** `ip nat inside` and `ip nat outside` MUST be set on correct interfaces or NAT won't work!

### Step 2 — Configure Static NAT (for Web Server)

```
R1(config)# ip nat inside source static 192.168.1.100 203.0.113.100
```

This maps Web Server's private IP permanently to one public IP.

### Step 3 — Configure PAT Overload (for PC-A and PC-B)

```
R1(config)# access-list 1 permit 192.168.1.0 0.0.0.255

R1(config)# ip nat inside source list 1 interface GigabitEthernet0/1 overload
```

> `overload` = PAT — many private IPs share ONE public IP using different port numbers.

### Step 4 — Add default route to simulate internet

```
R1(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.254
```

### Step 5 — Save configuration

```
R1# write memory
```

---

## ✅ Verification Commands

```
R1# show ip nat translations
R1# show ip nat statistics
R1# show running-config | include nat
R1# debug ip nat
```

### Expected Output — `show ip nat translations` (after PC-A pings internet)

```
Pro Inside global      Inside local       Outside local      Outside global
--- 203.0.113.100      192.168.1.100      ---                ---
tcp 203.0.113.1:1025   192.168.1.10:1025  8.8.8.8:80         8.8.8.8:80
tcp 203.0.113.1:1026   192.168.1.20:1035  8.8.8.8:80         8.8.8.8:80
```

> Notice PC-A and PC-B both use **203.0.113.1** but with **different port numbers** — that's PAT!

### Expected Output — `show ip nat statistics`

```
Total active translations: 3 (1 static, 2 dynamic; 2 extended)
Peak translations: 3
Outside interfaces: GigabitEthernet0/1
Inside interfaces: GigabitEthernet0/0
Hits: 24   Misses: 0
```

---

## 💡 Key Concepts

| Term | Explanation |
|------|-------------|
| NAT | Network Address Translation — changes IP addresses in packets |
| Static NAT | One-to-one mapping — one private IP = one fixed public IP |
| Dynamic NAT | Many-to-many — private IPs mapped to a pool of public IPs |
| PAT / Overload | Many-to-one — many private IPs share ONE public IP using ports |
| Inside Local | Private IP address (before NAT) |
| Inside Global | Public IP address (after NAT) |
| `ip nat inside` | Applied to LAN-facing interface |
| `ip nat outside` | Applied to WAN/Internet-facing interface |

### NAT Types Summary

| Type | Command | Use Case |
|------|---------|----------|
| Static NAT | `ip nat inside source static` | Servers needing fixed public IP |
| Dynamic NAT | `ip nat inside source list [acl] pool [pool]` | Users needing temporary public IPs |
| PAT | `ip nat inside source list [acl] interface [int] overload` | Home/office — many users, one IP |

---

## 📝 Review Questions

1. What is the difference between NAT and PAT?
2. Why does PAT use port numbers to differentiate connections?
3. What happens when the NAT translation table is full?
4. What does `show ip nat translations` show?
5. Why is PAT the most common NAT type used in homes and offices?

---

## ✅ Lab Completion Checklist

- [✅] `ip nat inside` set on Gi0/0 (LAN side)
- [✅] `ip nat outside` set on Gi0/1 (WAN side)
- [✅] Static NAT configured for Web Server (192.168.1.100 → 203.0.113.100)
- [✅] PAT configured for LAN subnet (192.168.1.0/24 → Gi0/1 overload)
- [✅] `show ip nat translations` shows active entries
- [✅] PC-A can reach internet (ping 8.8.8.8)

---

*[← Lab 09](./Lab-09-Extended-ACL.md) | Lab 10 of 12 | [Next Lab →](./Lab-11-DHCP.md) | [Back to Index](./README.md)*
