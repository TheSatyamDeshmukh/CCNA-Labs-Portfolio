# Lab-01 – Basic VLAN Configuration

**Topic:** VLANs | **Difficulty:** Intermediate | **Time:** ~30 mins

---

## 🎯 Objectives

By the end of this lab you will be able to:
- Create VLANs on a Cisco switch
- Assign switch ports to specific VLANs
- Verify VLAN configuration using `show` commands
- Understand why devices in different VLANs cannot communicate without a router

---

## 🖧 Topology

```
PC-A (192.168.10.10)            PC-B (192.168.20.10)
        |                               |
      Fa0/1                           Fa0/2
        |                               |
   +---+-------------------------------+---+
   |             SW1                       |
   |    Fa0/1=VLAN10   Fa0/2=VLAN20       |
   +---------------------------------------+
```

---

## 🧰 Device Table

| Device | Interface | IP Address     | Subnet Mask   | VLAN |
|--------|-----------|----------------|---------------|------|
| PC-A   | NIC       | 192.168.10.10  | 255.255.255.0 | 10   |
| PC-B   | NIC       | 192.168.20.10  | 255.255.255.0 | 20   |
| SW1    | VLAN 10   | 192.168.10.1   | 255.255.255.0 | 10   |
| SW1    | VLAN 20   | 192.168.20.1   | 255.255.255.0 | 20   |

---

## ⚙️ Step-by-Step Configuration

### Step 1 — Create VLANs on SW1

```
SW1> enable
SW1# configure terminal

SW1(config)# vlan 10
SW1(config-vlan)# name HR
SW1(config-vlan)# exit

SW1(config)# vlan 20
SW1(config-vlan)# name IT
SW1(config-vlan)# exit
```

### Step 2 — Assign ports to VLANs

```
SW1(config)# interface fastEthernet 0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# exit

SW1(config)# interface fastEthernet 0/2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 20
SW1(config-if)# exit
```

### Step 3 — Configure Switch Virtual Interfaces (SVI) for management

```
SW1(config)# interface vlan 10
SW1(config-if)# ip address 192.168.10.1 255.255.255.0
SW1(config-if)# no shutdown
SW1(config-if)# exit

SW1(config)# interface vlan 20
SW1(config-if)# ip address 192.168.20.1 255.255.255.0
SW1(config-if)# no shutdown
SW1(config-if)# exit

SW1(config)# ip routing
SW1(config)# end
```

### Step 4 — Save configuration

```
SW1# write memory
```

---

## ✅ Verification Commands

```
SW1# show vlan brief
SW1# show vlan id 10
SW1# show interfaces fastEthernet 0/1 switchport
SW1# show interfaces vlan 10
SW1# show running-config
```

### Expected Output — `show vlan brief`

```
VLAN Name                             Status    Ports
---- -------------------------------- --------- ------------------------
1    default                          active    Fa0/3, Fa0/4 ...
10   HR                               active    Fa0/1
20   IT                               active    Fa0/2
1002 fddi-default                     act/unsup
1003 token-ring-default               act/unsup
```

---

## 🧪 Testing

From PC-A, ping PC-B:
```
PC-A> ping 192.168.20.10
```
**Expected result:** Request timed out — because PCs are in different VLANs and no router is configured yet. (This is correct behavior for this lab!)

---

## 💡 Key Concepts

| Concept | Explanation |
|---------|-------------|
| VLAN | Virtual LAN — logically separates network traffic on same switch |
| Access port | Carries traffic for ONE VLAN only |
| SVI | Switch Virtual Interface — gives VLAN an IP address for management |
| Broadcast domain | VLANs create separate broadcast domains — VLAN 10 broadcasts don't reach VLAN 20 |

---

## 📝 Review Questions

1. What command verifies which ports are in VLAN 10?
2. Why can't PC-A ping PC-B in this lab?
3. What is the purpose of the `switchport mode access` command?
4. What happens to traffic if a port is not assigned to any VLAN?

> **Answers:** See Lab-03 (Inter-VLAN Routing) to understand how to enable communication between VLANs.

---

## ✅ Lab Completion Checklist

- [ ] VLANs 10 and 20 created successfully
- [ ] Ports assigned to correct VLANs
- [ ] `show vlan brief` shows correct output
- [ ] PC-A cannot ping PC-B (confirming VLAN isolation)
- [ ] Configuration saved with `write memory`

---

*Lab 01 of 12 | [Next Lab →](./Lab-02-VLAN-Trunking.md) | [Back to Labs Index](./README.md)*
