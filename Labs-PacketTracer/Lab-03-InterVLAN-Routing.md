# Lab-03 – Inter-VLAN Routing (Router-on-a-Stick)

**Topic:** Routing / VLANs | **Difficulty:** Intermediate | **Time:** ~40 mins

---

## 🎯 Objectives

- Configure router subinterfaces for Inter-VLAN routing
- Enable communication between VLAN 10 and VLAN 20
- Use `encapsulation dot1Q` on router subinterfaces
- Verify routing between VLANs using ping

---

## 🖧 Topology

```
PC-A (192.168.10.10)      PC-B (192.168.20.10)
        |                          |
      Fa0/1                      Fa0/2
        |                          |
   +----+---------------------------+----+
   |                SW1                  |
   |         Gi0/1 (Trunk)               |
   +--------------------+----------------+
                        |
                      Fa0/0
                        |
              +---------+----------+
              |         R1         |
              |  Fa0/0.10 Fa0/0.20 |
              +--------------------+
```

---

## 🧰 Device Table

| Device | Interface   | IP Address    | Subnet Mask   | Gateway        |
|--------|-------------|---------------|---------------|----------------|
| PC-A   | NIC         | 192.168.10.10 | 255.255.255.0 | 192.168.10.1   |
| PC-B   | NIC         | 192.168.20.10 | 255.255.255.0 | 192.168.20.1   |
| R1     | Fa0/0.10    | 192.168.10.1  | 255.255.255.0 | —              |
| R1     | Fa0/0.20    | 192.168.20.1  | 255.255.255.0 | —              |

---

## ⚙️ Step-by-Step Configuration

### Step 1 — Configure VLANs and trunk on SW1

```
SW1(config)# vlan 10
SW1(config-vlan)# name HR
SW1(config-vlan)# exit

SW1(config)# vlan 20
SW1(config-vlan)# name IT
SW1(config-vlan)# exit

SW1(config)# interface fa0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# exit

SW1(config)# interface fa0/2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 20
SW1(config-if)# exit

SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20
SW1(config-if)# exit
```

### Step 2 — Configure R1 subinterfaces

```
R1(config)# interface fa0/0
R1(config-if)# no shutdown
R1(config-if)# exit

R1(config)# interface fa0/0.10
R1(config-subif)# encapsulation dot1Q 10
R1(config-subif)# ip address 192.168.10.1 255.255.255.0
R1(config-subif)# exit

R1(config)# interface fa0/0.20
R1(config-subif)# encapsulation dot1Q 20
R1(config-subif)# ip address 192.168.20.1 255.255.255.0
R1(config-subif)# exit

R1(config)# end
R1# write memory
```

### Step 3 — Set default gateways on PCs

```
PC-A gateway: 192.168.10.1
PC-B gateway: 192.168.20.1
```

---

## ✅ Verification Commands

```
R1# show ip interface brief
R1# show ip route
R1# show running-config interface fa0/0.10
R1# show interfaces fa0/0.10
```

### Expected Output — `show ip route`

```
C    192.168.10.0/24 is directly connected, FastEthernet0/0.10
C    192.168.20.0/24 is directly connected, FastEthernet0/0.20
```

---

## 🧪 Testing

```
PC-A> ping 192.168.10.1    (R1 subinterface — should WORK ✅)
PC-A> ping 192.168.20.1    (R1 other subinterface — should WORK ✅)
PC-A> ping 192.168.20.10   (PC-B — should WORK ✅ Inter-VLAN routing!)
```

---

## 💡 Key Concepts

| Concept | Explanation |
|---------|-------------|
| Router-on-a-Stick | One physical router port handles multiple VLANs via subinterfaces |
| Subinterface | Logical division of one physical interface (Fa0/0.10, Fa0/0.20) |
| `encapsulation dot1Q` | Tells the subinterface which VLAN tag to process |
| Default gateway | Must point to the router subinterface IP for that VLAN |

> 💡 **Why "Router-on-a-Stick"?** Because there is only ONE cable from the switch to the router — like a stick — but it carries ALL VLAN traffic via 802.1Q tagging.

---

## 📝 Review Questions

1. What does `encapsulation dot1Q 10` do on a subinterface?
2. Why must the physical interface `fa0/0` be set to `no shutdown`?
3. What is the default gateway for PC-A? Why?
4. What would happen if you forgot to set `encapsulation dot1Q`?

---

## ✅ Lab Completion Checklist

- [ ] VLANs and trunk configured on SW1
- [ ] R1 Fa0/0 is up with `no shutdown`
- [ ] Subinterface Fa0/0.10 has IP 192.168.10.1
- [ ] Subinterface Fa0/0.20 has IP 192.168.20.1
- [ ] PC-A can ping PC-B (inter-VLAN routing works!)
- [ ] `show ip route` shows both subnets as connected

---

*[← Lab 02](./Lab-02-VLAN-Trunking.md) | Lab 03 of 12 | [Next Lab →](./Lab-04-STP.md) | [Back to Index](./README.md)*
