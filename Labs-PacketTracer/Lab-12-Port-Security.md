# Lab-12 – Port Security & Layer 2 Security

**Topic:** Security | **Difficulty:** Advanced | **Time:** ~45 mins

---

## 🎯 Objectives

- Enable port security on access switch ports
- Configure sticky MAC address learning
- Test all three violation modes: shutdown, restrict, protect
- Enable BPDU Guard to prevent rogue switches
- Verify security with `show port-security`

---

## 🖧 Topology

```
PC-A (Authorized) ---Fa0/1---+
PC-B (Authorized) ---Fa0/2---+--- SW1
Rogue PC (DENY!) ---Fa0/3---+     |
Rogue SW (DENY!) ---Fa0/4---+   Management
                                  VLAN 99
```

---

## 🧰 Device Table

| Device | Interface | MAC | Role |
|--------|-----------|-----|------|
| PC-A | Fa0/1 | Auto (sticky) | Authorized user |
| PC-B | Fa0/2 | Auto (sticky) | Authorized user |
| Rogue PC | Fa0/3 | Unknown | BLOCKED |
| Rogue Switch | Fa0/4 | — | BLOCKED by BPDU Guard |

---

## ⚙️ Step-by-Step Configuration

### Step 1 — Configure access ports with Port Security (Sticky MAC)

**For PC-A on Fa0/1:**
```
SW1(config)# interface fastEthernet 0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 1
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# switchport port-security violation shutdown
SW1(config-if)# exit
```

**For PC-B on Fa0/2:**
```
SW1(config)# interface fastEthernet 0/2
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 1
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# switchport port-security violation restrict
SW1(config-if)# exit
```

### Step 2 — Configure port with Protect violation mode (silent drop)

**For Fa0/3 — any device connecting gets drops (no log, no shutdown):**
```
SW1(config)# interface fastEthernet 0/3
SW1(config-if)# switchport mode access
SW1(config-if)# switchport port-security
SW1(config-if)# switchport port-security maximum 1
SW1(config-if)# switchport port-security mac-address sticky
SW1(config-if)# switchport port-security violation protect
SW1(config-if)# exit
```

### Step 3 — Enable PortFast and BPDU Guard (prevent rogue switches)

```
SW1(config)# interface fastEthernet 0/1
SW1(config-if)# spanning-tree portfast
SW1(config-if)# spanning-tree bpduguard enable
SW1(config-if)# exit

SW1(config)# interface fastEthernet 0/2
SW1(config-if)# spanning-tree portfast
SW1(config-if)# spanning-tree bpduguard enable
SW1(config-if)# exit

SW1(config)# interface fastEthernet 0/3
SW1(config-if)# spanning-tree portfast
SW1(config-if)# spanning-tree bpduguard enable
SW1(config-if)# exit
```

### Optional — Enable BPDU Guard globally for all PortFast ports

```
SW1(config)# spanning-tree portfast bpduguard default
```

### Step 4 — Save configuration

```
SW1# end
SW1# write memory
```

---

## ✅ Verification Commands

```
SW1# show port-security
SW1# show port-security interface fastEthernet 0/1
SW1# show port-security address
SW1# show interfaces fastEthernet 0/1 status
SW1# show spanning-tree interface fastEthernet 0/1 detail
```

### Expected Output — `show port-security interface fa0/1`

```
Port Security              : Enabled
Port Status                : Secure-up
Violation Mode             : Shutdown
Aging Time                 : 0 mins
Aging Type                 : Absolute
SecureStatic Address Aging : Disabled
Maximum MAC Addresses      : 1
Total MAC Addresses        : 1
Configured MAC Addresses   : 0
Sticky MAC Addresses       : 1
Last Source Address:Vlan   : 00AA.BBCC.DD11:10
Security Violation Count   : 0
```

### Expected Output — After a violation (shutdown mode)

```
Port Security              : Enabled
Port Status                : Secure-shutdown  ← PORT IS DOWN!
Violation Mode             : Shutdown
Security Violation Count   : 1
```

---

## 🧪 Testing Violations

### Test 1 — Connect Rogue PC to Fa0/1 (after PC-A's MAC is learned)

Disconnect PC-A, connect a different PC → Port should go to `err-disabled`!

```
SW1# show interfaces fastEthernet 0/1 status
Port      Name   Status       Vlan  Duplex Speed Type
Fa0/1            err-disabled  10    auto   auto  10/100BaseTX
```

### Re-enable a shutdown port after fixing the issue

```
SW1(config)# interface fastEthernet 0/1
SW1(config-if)# shutdown
SW1(config-if)# no shutdown
SW1(config-if)# exit
```

---

## 💡 Key Concepts

| Concept | Explanation |
|---------|-------------|
| Port Security | Restricts which MAC addresses can use a port |
| Maximum | Max number of MAC addresses allowed on the port |
| Sticky MAC | Automatically learns and saves MAC to running config |
| Violation Shutdown | **Best security** — shuts port down and sends SNMP trap |
| Violation Restrict | Drops frames, sends log/SNMP, keeps port up, increments counter |
| Violation Protect | Silently drops frames — no log, no SNMP, port stays up |
| BPDU Guard | Shuts port if a BPDU (switch packet) is received — stops rogue switches |
| PortFast | Skips STP listening/learning states — for end devices only |
| err-disabled | Port automatically shutdown by IOS due to a security/error event |

### Violation Mode Comparison

| Mode | Port Status | Drops Frames | Sends Log | Increments Counter |
|------|------------|--------------|-----------|---------------------|
| Shutdown | err-disabled | Yes | Yes | Yes |
| Restrict | Active | Yes | Yes | Yes |
| Protect | Active | Yes | No | No |

---

## 📝 Review Questions

1. What is the difference between `restrict` and `protect` violation modes?
2. How do you recover a port that went into `err-disabled` state?
3. What is a sticky MAC address and where is it saved?
4. Why should PortFast NEVER be enabled on a trunk port?
5. What attack does BPDU Guard prevent?

---

## ✅ Lab Completion Checklist

- [ ] Port security enabled on Fa0/1, Fa0/2, Fa0/3
- [ ] Sticky MAC learned automatically after connecting PCs
- [ ] Violation mode `shutdown` set on Fa0/1
- [ ] Violation mode `restrict` set on Fa0/2
- [ ] Violation mode `protect` set on Fa0/3
- [ ] BPDU Guard enabled on all access ports
- [ ] Tested violation — Fa0/1 goes to `err-disabled` when rogue PC connected
- [ ] Successfully recovered `err-disabled` port

---

*[← Lab 11](./Lab-11-DHCP.md) | Lab 12 of 12 — Final Lab! 🎉 | [Back to Labs Index](./README.md)*

---

> 🏆 **Congratulations!** You have completed all 12 CCNA Intermediate Labs.
> Next step: Complete [Practice Exams](../Practice-Exams/) to test your knowledge!
