# Lab-02 – VLAN Trunking (802.1Q)

**Topic:** VLANs / Trunking | **Difficulty:** Intermediate | **Time:** ~30 mins

---

## 🎯 Objectives

- Configure 802.1Q trunk links between two switches
- Understand the difference between access and trunk ports
- Control which VLANs are allowed on a trunk
- Configure and verify the native VLAN

---

## 🖧 Topology

```
PC-A (VLAN 10)    PC-C (VLAN 10)
      |                  |
    Fa0/1              Fa0/1
      |                  |
+-----+----+        +----+-----+
|   SW1    +--------+   SW2   |
|          | Trunk  |         |
+-----+----+ Gi0/1  +----+----+
      |      <=====>      |
    Fa0/2              Fa0/2
      |                  |
PC-B (VLAN 20)    PC-D (VLAN 20)
```

---

## 🧰 Device Table

| Device | Interface | IP Address     | VLAN |
|--------|-----------|----------------|------|
| PC-A   | NIC       | 192.168.10.10  | 10   |
| PC-B   | NIC       | 192.168.20.10  | 20   |
| PC-C   | NIC       | 192.168.10.11  | 10   |
| PC-D   | NIC       | 192.168.20.11  | 20   |

---

## ⚙️ Step-by-Step Configuration

### Step 1 — Create VLANs on both switches

**On SW1 and SW2 (same commands):**
```
SW1(config)# vlan 10
SW1(config-vlan)# name HR
SW1(config-vlan)# exit

SW1(config)# vlan 20
SW1(config-vlan)# name IT
SW1(config-vlan)# exit

SW1(config)# vlan 99
SW1(config-vlan)# name Native
SW1(config-vlan)# exit
```

### Step 2 — Assign access ports on SW1

```
SW1(config)# interface fa0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# exit

SW1(config)# interface fa0/2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 20
SW1(config-if)# exit
```

### Step 3 — Configure trunk port on SW1

```
SW1(config)# interface GigabitEthernet0/1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk native vlan 99
SW1(config-if)# switchport trunk allowed vlan 10,20,99
SW1(config-if)# exit
```

### Step 4 — Configure trunk port on SW2 (same commands)

```
SW2(config)# interface GigabitEthernet0/1
SW2(config-if)# switchport mode trunk
SW2(config-if)# switchport trunk native vlan 99
SW2(config-if)# switchport trunk allowed vlan 10,20,99
SW2(config-if)# exit
```

### Step 5 — Assign access ports on SW2

```
SW2(config)# interface fa0/1
SW2(config-if)# switchport mode access
SW2(config-if)# switchport access vlan 10
SW2(config-if)# exit

SW2(config)# interface fa0/2
SW2(config-if)# switchport mode access
SW2(config-if)# switchport access vlan 20
SW2(config-if)# exit
```

---

## ✅ Verification Commands

```
SW1# show interfaces trunk
SW1# show interfaces GigabitEthernet0/1 switchport
SW1# show vlan brief
SW1# show running-config interface GigabitEthernet0/1
```

### Expected Output — `show interfaces trunk`

```
Port        Mode         Encapsulation  Status        Native vlan
Gi0/1       on           802.1q         trunking      99

Port        Vlans allowed on trunk
Gi0/1       10,20,99

Port        Vlans allowed and active in management domain
Gi0/1       10,20,99
```

---

## 🧪 Testing

```
PC-A> ping 192.168.10.11   (PC-C — same VLAN, different switch — should WORK ✅)
PC-A> ping 192.168.20.10   (PC-B — different VLAN — should FAIL ❌)
```

---

## 💡 Key Concepts

| Concept | Explanation |
|---------|-------------|
| 802.1Q | IEEE standard for VLAN tagging on trunk links |
| Trunk port | Carries multiple VLANs between switches |
| VLAN tag | 4-byte header added to frames on trunk ports |
| Native VLAN | Traffic sent untagged on trunk — both ends must match! |
| Allowed VLANs | Controls which VLANs can pass through the trunk |

> ⚠️ **Security Note:** Never use VLAN 1 as your native VLAN in production. Use an unused VLAN (like 99) to prevent VLAN hopping attacks.

---

## 📝 Review Questions

1. What is the difference between an access port and a trunk port?
2. What happens if native VLAN is different on each end of a trunk?
3. How do you remove a VLAN from the allowed list on a trunk?
4. What encapsulation does `show interfaces trunk` show for 802.1Q?

---

## ✅ Lab Completion Checklist

- [ ] VLANs 10, 20, 99 created on both switches
- [ ] Trunk configured on Gi0/1 of SW1 and SW2
- [ ] Native VLAN set to 99 on both sides
- [ ] PC-A can ping PC-C (same VLAN across switches)
- [ ] PC-A cannot ping PC-B (different VLAN — correct!)

---

*[← Lab 01](./Lab-01-VLAN-Config.md) | Lab 02 of 12 | [Next Lab →](./Lab-03-InterVLAN-Routing.md) | [Back to Index](./README.md)*
