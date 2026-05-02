# Lab-06 – OSPFv2 Single Area

**Topic:** OSPF Routing | **Difficulty:** Intermediate | **Time:** ~40 mins

---

## 🎯 Objectives

- Configure OSPFv2 in a single area (Area 0)
- Set Router IDs manually
- Verify OSPF neighbor adjacency (Full state)
- Verify routes learned via OSPF in the routing table
- Understand DR/BDR election on multi-access networks

---

## 🖧 Topology

```
192.168.12.0/30          192.168.23.0/30
                                 
10.0.0.0/24    10.0.1.0/24    10.0.2.0/24
(Lo0 R1)       (Lo0 R2)       (Lo0 R3)

   R1 ----Gi0/0-------- R2 --------Gi0/1---- R3
   |                    |                    |
 Lo0                  Lo0                  Lo0
(1.1.1.1)           (2.2.2.2)           (3.3.3.3)

All in OSPF Area 0
```

---

## 🧰 Device Table

| Device | Interface | IP Address        | Notes |
|--------|-----------|-------------------|-------|
| R1 | Gi0/0 | 192.168.12.1/30 | Link to R2 |
| R1 | Lo0 | 10.0.0.1/24 | Loopback — simulate LAN |
| R2 | Gi0/0 | 192.168.12.2/30 | Link to R1 |
| R2 | Gi0/1 | 192.168.23.1/30 | Link to R3 |
| R2 | Lo0 | 10.0.1.1/24 | Loopback |
| R3 | Gi0/0 | 192.168.23.2/30 | Link to R2 |
| R3 | Lo0 | 10.0.2.1/24 | Loopback |

---

## ⚙️ Step-by-Step Configuration

### Step 1 — Configure interfaces on all routers

**R1:**
```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.12.1 255.255.255.252
R1(config-if)# no shutdown
R1(config-if)# exit

R1(config)# interface Loopback0
R1(config-if)# ip address 10.0.0.1 255.255.255.0
R1(config-if)# exit
```

**R2:**
```
R2(config)# interface GigabitEthernet0/0
R2(config-if)# ip address 192.168.12.2 255.255.255.252
R2(config-if)# no shutdown
R2(config-if)# exit

R2(config)# interface GigabitEthernet0/1
R2(config-if)# ip address 192.168.23.1 255.255.255.252
R2(config-if)# no shutdown
R2(config-if)# exit

R2(config)# interface Loopback0
R2(config-if)# ip address 10.0.1.1 255.255.255.0
R2(config-if)# exit
```

**R3:**
```
R3(config)# interface GigabitEthernet0/0
R3(config-if)# ip address 192.168.23.2 255.255.255.252
R3(config-if)# no shutdown
R3(config-if)# exit

R3(config)# interface Loopback0
R3(config-if)# ip address 10.0.2.1 255.255.255.0
R3(config-if)# exit
```

### Step 2 — Configure OSPF on all routers

**R1:**
```
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1
R1(config-router)# network 192.168.12.0 0.0.0.3 area 0
R1(config-router)# network 10.0.0.0 0.0.0.255 area 0
R1(config-router)# passive-interface Loopback0
R1(config-router)# exit
```

**R2:**
```
R2(config)# router ospf 1
R2(config-router)# router-id 2.2.2.2
R2(config-router)# network 192.168.12.0 0.0.0.3 area 0
R2(config-router)# network 192.168.23.0 0.0.0.3 area 0
R2(config-router)# network 10.0.1.0 0.0.0.255 area 0
R2(config-router)# passive-interface Loopback0
R2(config-router)# exit
```

**R3:**
```
R3(config)# router ospf 1
R3(config-router)# router-id 3.3.3.3
R3(config-router)# network 192.168.23.0 0.0.0.3 area 0
R3(config-router)# network 10.0.2.0 0.0.0.255 area 0
R3(config-router)# passive-interface Loopback0
R3(config-router)# exit
```

---

## ✅ Verification Commands

```
R1# show ip ospf neighbor
R1# show ip route ospf
R1# show ip route
R1# show ip ospf
R1# show ip ospf interface GigabitEthernet0/0
R2# show ip ospf neighbor
```

### Expected Output — `show ip ospf neighbor` on R2

```
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           1   FULL/DR         00:00:37    192.168.12.1    GigabitEthernet0/0
3.3.3.3           1   FULL/DR         00:00:32    192.168.23.2    GigabitEthernet0/1
```

> ✅ **FULL** state = OSPF adjacency fully formed = routing working!

### Expected Output — `show ip route` on R1

```
     10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
C       10.0.0.0/24 is directly connected, Loopback0
O       10.0.1.0/24 [110/2] via 192.168.12.2, GigabitEthernet0/0
O       10.0.2.0/24 [110/3] via 192.168.12.2, GigabitEthernet0/0
```

> `O` = OSPF route | `[110/2]` = AD 110 / Cost 2

---

## 💡 Key Concepts

| Concept | Explanation |
|---------|-------------|
| Router ID | Unique identifier for each OSPF router (highest loopback IP or manually set) |
| Area 0 | Backbone area — all OSPF areas must connect to Area 0 |
| `network` command | Tells OSPF which interfaces to include (uses wildcard mask) |
| `passive-interface` | Advertises the network but doesn't send OSPF hellos on that interface |
| FULL state | Both routers have identical OSPF database — fully converged |
| AD 110 | Administrative Distance for OSPF (lower = preferred) |

### OSPF Neighbor States

| State | Meaning |
|-------|---------|
| INIT | Hello received but not yet two-way |
| 2-WAY | Both routers see each other's Hello |
| EXSTART | Beginning database exchange |
| EXCHANGE | Sharing DBD (Database Description) packets |
| LOADING | Requesting missing LSAs |
| **FULL** | Fully adjacent — database synchronized ✅ |

---

## 📝 Review Questions

1. What is the default Administrative Distance for OSPF?
2. Why do we set `passive-interface` on the Loopback interface?
3. What does the wildcard mask `0.0.0.3` mean for network `192.168.12.0`?
4. What does OSPF cost mean and how is it calculated?
5. What state must OSPF neighbors reach for routing to work?

---

## ✅ Lab Completion Checklist

- [ ] All router interfaces configured and UP/UP
- [ ] OSPF process 1 running on all routers
- [ ] Router IDs set manually (1.1.1.1, 2.2.2.2, 3.3.3.3)
- [ ] `show ip ospf neighbor` shows FULL state on all
- [ ] R1 routing table shows O routes to 10.0.1.0 and 10.0.2.0
- [ ] R1 can ping 10.0.2.1 (R3 loopback) — end-to-end test

---

*[← Lab 05](./Lab-05-EtherChannel.md) | Lab 06 of 12 | [Next Lab →](./Lab-07-OSPF-MultiArea.md) | [Back to Index](./README.md)*
