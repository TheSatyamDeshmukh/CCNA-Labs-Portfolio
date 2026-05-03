# Lab-07 – OSPFv2 Multi-Area

**Topic:** OSPF Routing | **Difficulty:** Advanced | **Time:** ~50 mins

---

## 🎯 Objectives

- Configure OSPF with Area 0 (backbone) and Area 1
- Understand the role of an ABR (Area Border Router)
- Configure inter-area route summarization
- Verify LSA types in different areas

---

## 🖧 Topology

```
Area 1                      Area 0 (Backbone)
                   
[R1]----Gi0/0----[R2 (ABR)]----Gi0/1----[R3]
 |                  |
Lo0:10.1.0.1/24   Lo0:2.2.2.2
(Area 1)          (Area 0 + Area 1)
```

---

## 🧰 Device Table

| Device | Interface | IP Address | Area |
|--------|-----------|------------|------|
| R1 | Gi0/0 | 10.12.0.1/24 | Area 1 |
| R1 | Lo0 | 10.1.1.1/24 | Area 1 |
| R2 (ABR) | Gi0/0 | 10.12.0.2/24 | Area 1 |
| R2 (ABR) | Gi0/1 | 10.23.0.1/24 | Area 0 |
| R2 (ABR) | Lo0 | 2.2.2.2/32 | — |
| R3 | Gi0/0 | 10.23.0.2/24 | Area 0 |
| R3 | Lo0 | 10.3.0.1/24 | Area 0 |

---

## ⚙️ Configuration

### R1 — Area 1 only

```
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# network 10.12.0.0 0.0.0.255 area 1
R1(config-router)# network 10.1.1.0 0.0.0.255 area 1
R1(config-router)# passive-interface Loopback0
R1(config-router)# exit
```

### R2 — ABR (Area 0 AND Area 1)

```
R2(config)# router ospf 1
R2(config-router)# router-id 2.2.2.2
R2(config-router)# network 10.12.0.0 0.0.0.255 area 1
R2(config-router)# network 10.23.0.0 0.0.0.255 area 0
R2(config-router)# area 1 range 10.1.0.0 255.255.0.0
R2(config-router)# exit
```

> `area 1 range` = summarizes all Area 1 routes into one prefix before advertising to Area 0

### R3 — Area 0 only

```
R3(config)# router ospf 1
R3(config-router)# router-id 3.3.3.3
R3(config-router)# network 10.23.0.0 0.0.0.255 area 0
R3(config-router)# network 10.3.0.0 0.0.0.255 area 0
R3(config-router)# passive-interface Loopback0
R3(config-router)# exit
```

---

## ✅ Verification

```
R2# show ip ospf
R2# show ip ospf border-routers
R3# show ip route ospf
R3# show ip ospf database
```

---

## 💡 Key Concepts

| Term | Explanation |
|------|-------------|
| ABR | Area Border Router — connects two OSPF areas |
| LSA Type 1 | Router LSA — within same area only |
| LSA Type 2 | Network LSA — within same area only |
| LSA Type 3 | Summary LSA — advertised by ABR between areas |
| `area range` | Summarizes routes at ABR to reduce routing table size |

---

## 📝 Review Questions

1. What is an ABR and why is it important?
2. What LSA type does the ABR use to advertise Area 1 routes into Area 0?
3. What is the benefit of using `area 1 range` for summarization?
4. Why must all OSPF areas connect to Area 0?

---

## ✅ Lab Completion Checklist

- [ ] R1 in Area 1, R3 in Area 0
- [ ] R2 configured as ABR (participates in both areas)
- [ ] R3 sees summarized route 10.1.0.0/16 (not individual /24s)
- [ ] Full reachability — R1 can ping R3 loopback

---

*[← Lab 06](./Lab-06-OSPF-SingleArea.md) | Lab 07 of 12 | [Next Lab →](./Lab-08-Standard-ACL.md) | [Back to Index](./README.md)*
