# Lab-05 – EtherChannel (LACP)

**Topic:** EtherChannel / STP | **Difficulty:** Intermediate | **Time:** ~35 mins

---

## 🎯 Objectives

- Bundle two physical links into one logical EtherChannel using LACP
- Configure EtherChannel with `channel-group` and `mode active`
- Verify EtherChannel status using `show etherchannel summary`
- Understand how EtherChannel provides redundancy and increased bandwidth

---

## 🖧 Topology

```
+----------+                    +----------+
|          | Gi0/1 ==========> |          |
|   SW1    |                    |   SW2    |
|          | Gi0/2 ==========> |          |
+----------+   (Port-Channel1)  +----------+

Both physical links bundled into Po1 = 2 Gbps logical link
```

---

## 🧰 Device Table

| Device | Interfaces | Port-Channel | Mode |
|--------|-----------|--------------|------|
| SW1 | Gi0/1, Gi0/2 | Port-channel1 | LACP Active |
| SW2 | Gi0/1, Gi0/2 | Port-channel1 | LACP Passive |

---

## ⚙️ Step-by-Step Configuration

### Step 1 — Configure EtherChannel on SW1 (Active side)

```
SW1(config)# interface range GigabitEthernet0/1 - 2
SW1(config-if-range)# channel-group 1 mode active
SW1(config-if-range)# exit
```

### Step 2 — Configure EtherChannel on SW2 (Passive side)

```
SW2(config)# interface range GigabitEthernet0/1 - 2
SW2(config-if-range)# channel-group 1 mode passive
SW2(config-if-range)# exit
```

### Step 3 — Configure Port-Channel interface as trunk

**On SW1:**
```
SW1(config)# interface port-channel 1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk allowed vlan 10,20
SW1(config-if)# exit
```

**On SW2:**
```
SW2(config)# interface port-channel 1
SW2(config-if)# switchport mode trunk
SW2(config-if)# switchport trunk allowed vlan 10,20
SW2(config-if)# exit
```

### Step 4 — Save configuration

```
SW1# write memory
SW2# write memory
```

---

## ✅ Verification Commands

```
SW1# show etherchannel summary
SW1# show etherchannel port-channel
SW1# show interfaces port-channel 1
SW1# show interfaces port-channel 1 trunk
SW1# show running-config interface port-channel 1
```

### Expected Output — `show etherchannel summary`

```
Flags:  D - down        P - bundled in port-channel
        I - stand-alone s - suspended
        H - Hot-standby (LACP only)
        R - Layer3      S - Layer2
        U - in use      N - not in use, no aggregation
        f - failed to allocate aggregator

        M - not in use, minimum links not met
        u - unsuitable for bundling
        w - waiting to be aggregated
        d - default port

Number of channel-groups in use: 1
Number of aggregators:           1

Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SU)         LACP      Gi0/1(P)   Gi0/2(P)
```

> ✅ `SU` = Layer **2** (**S**) and in **U**se | `P` = **P**ort bundled = SUCCESS

---

## 💡 Key Concepts

| Concept | Explanation |
|---------|-------------|
| EtherChannel | Bundles multiple physical links into one logical link |
| LACP | Link Aggregation Control Protocol (IEEE 802.3ad) — negotiates automatically |
| Active | Actively sends LACP negotiation packets |
| Passive | Responds to LACP but doesn't initiate |
| Port-Channel | The logical interface created by EtherChannel |

### LACP Mode Combinations

| SW1 Mode | SW2 Mode | Result |
|----------|----------|--------|
| Active | Active | ✅ Forms EtherChannel |
| Active | Passive | ✅ Forms EtherChannel |
| Passive | Passive | ❌ Does NOT form (neither initiates) |

### EtherChannel Benefits

- **Increased bandwidth** — 2 × 1Gbps = 2Gbps logical link
- **Redundancy** — if one physical link fails, traffic continues on the other
- **STP sees ONE link** — EtherChannel hides the physical redundancy from STP

---

## 📝 Review Questions

1. What is the difference between LACP `active` and `passive` mode?
2. What does `SU` mean in `show etherchannel summary` output?
3. Can you bundle interfaces with different speeds in EtherChannel? Why or why not?
4. What happens to traffic if one of the physical links (Gi0/1) goes down?
5. How does STP treat a Port-Channel interface?

---

## ✅ Lab Completion Checklist

- [ ] EtherChannel formed (Po1 shows `SU` in summary)
- [ ] Both Gi0/1 and Gi0/2 show `P` (bundled)
- [ ] Port-Channel1 configured as trunk
- [ ] `show interfaces port-channel 1` shows UP/UP
- [ ] Tested by disabling one link — traffic still flows

---

*[← Lab 04](./Lab-04-STP.md) | Lab 05 of 12 | [Next Lab →](./Lab-06-OSPF-SingleArea.md) | [Back to Index](./README.md)*
