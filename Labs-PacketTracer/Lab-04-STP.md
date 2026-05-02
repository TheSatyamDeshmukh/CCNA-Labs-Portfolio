# Lab-04 – Spanning Tree Protocol (STP)

**Topic:** STP | **Difficulty:** Intermediate | **Time:** ~35 mins

---

## 🎯 Objectives

- Observe the STP root bridge election process
- Manually configure the root bridge using priority
- Identify Root Port, Designated Port, and Blocked Port
- Verify STP convergence with `show spanning-tree`

---

## 🖧 Topology

```
            +----------+
            |   SW1    |  ← Root Bridge (Priority 4096)
            | BID low  |
            +---+--+---+
               /    \
         Gi0/1/      \Gi0/2
             /        \
    +-------+--+    +--+-------+
    |   SW2    |    |   SW3    |
    |          +----+          |
    +----------+ Gi0/1 (BLOCKED on SW3 side)
```

---

## 🧰 Device Table

| Device | Role | Bridge Priority | Notes |
|--------|------|-----------------|-------|
| SW1 | Root Bridge | 4096 | Manually configured |
| SW2 | Non-root | 32768 | Default priority |
| SW3 | Non-root | 32768 | Default priority — has Blocked port |

---

## ⚙️ Step-by-Step Configuration

### Step 1 — Connect the switches (in Packet Tracer)

Connect the three switches in a triangle:
- SW1 Gi0/1 → SW2 Gi0/1
- SW1 Gi0/2 → SW3 Gi0/1
- SW2 Gi0/2 → SW3 Gi0/2 *(this link will be blocked by STP)*

### Step 2 — Check default STP before any config

```
SW1# show spanning-tree vlan 1
SW2# show spanning-tree vlan 1
SW3# show spanning-tree vlan 1
```
Note: By default, the switch with the lowest MAC address becomes root bridge.

### Step 3 — Manually set SW1 as Root Bridge

```
SW1(config)# spanning-tree vlan 1 priority 4096
```

> 💡 **Note:** STP priorities must be multiples of 4096. Lower = higher priority = becomes root.

### Step 4 — Verify SW2 and SW3 priority (leave as default)

```
SW2(config)# spanning-tree vlan 1 priority 32768
SW3(config)# spanning-tree vlan 1 priority 32768
```

### Step 5 — Wait 30–50 seconds for STP to converge

STP goes through these states: **Blocking → Listening → Learning → Forwarding**

```
SW1# show spanning-tree vlan 1
SW2# show spanning-tree vlan 1
SW3# show spanning-tree vlan 1
```

---

## ✅ Verification Commands

```
SW1# show spanning-tree
SW1# show spanning-tree vlan 1 detail
SW1# show spanning-tree vlan 1 brief
SW2# show spanning-tree
SW3# show spanning-tree
```

### Expected Output — SW1 `show spanning-tree vlan 1`

```
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    4097
             Address     0001.AAAA.AAAA
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    4097   (priority 4096 sys-id-ext 1)
             Address     0001.AAAA.AAAA
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec

Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- -----
Gi0/1               Desg FWD 4         128.1    P2p
Gi0/2               Desg FWD 4         128.2    P2p
```

### Expected Output — SW3 (has Blocked port)

```
Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- -----
Gi0/1               Root FWD 4         128.1    P2p
Gi0/2               Altn BLK 4         128.2    P2p   ← BLOCKED!
```

---

## 💡 Key Concepts

| Term | Explanation |
|------|-------------|
| Root Bridge | Switch with lowest Bridge ID (Priority + MAC). All traffic flows through it. |
| Root Port | Port on non-root switch facing the root bridge (best path to root) |
| Designated Port | Port that forwards traffic on a network segment |
| Blocked Port (Alternate) | Port that is shut down by STP to prevent loops |
| Bridge ID | Priority (4096) + VLAN ID (1) + MAC address |

### STP Port States

| State | Duration | Description |
|-------|----------|-------------|
| Blocking | 20 sec | Receives BPDUs, does not forward frames |
| Listening | 15 sec | Processes BPDUs, prepares to forward |
| Learning | 15 sec | Builds MAC address table |
| Forwarding | Permanent | Fully operational — forwards frames |

---

## 📝 Review Questions

1. What determines which switch becomes the root bridge?
2. Which switch has the Blocked port and which interface is it?
3. How long does STP convergence take by default?
4. What happens to the Blocked port if the Root Bridge fails?
5. What is `sys-id-ext` in the bridge priority output?

---

## ✅ Lab Completion Checklist

- [ ] SW1 is confirmed as Root Bridge in `show spanning-tree`
- [ ] SW2 has a Root Port facing SW1
- [ ] SW3 has a Root Port and one Blocked (Altn) port
- [ ] No loops exist in the network
- [ ] Understood all 4 STP port states

---

*[← Lab 03](./Lab-03-InterVLAN-Routing.md) | Lab 04 of 12 | [Next Lab →](./Lab-05-EtherChannel.md) | [Back to Index](./README.md)*
