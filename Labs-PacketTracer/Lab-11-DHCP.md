# Lab-11 – DHCP Server on Router

**Topic:** DHCP | **Difficulty:** Intermediate | **Time:** ~30 mins

---

## 🎯 Objectives

- Configure a Cisco router as a DHCP server
- Exclude static IP addresses from the DHCP pool
- Set DHCP options (gateway, DNS, lease time)
- Verify DHCP bindings and client assignment

---

## 🖧 Topology

```
PC-A (DHCP client)  ---+
                        |--- SW1 ---Gi0/0--- R1 (DHCP Server)
PC-B (DHCP client)  ---+

R1 assigns IPs from 192.168.1.11 to 192.168.1.254
R1 itself = 192.168.1.1 (excluded from pool)
```

---

## 🧰 Device Table

| Device | IP Address | Method |
|--------|------------|--------|
| R1 Gi0/0 | 192.168.1.1/24 | Static (manually set) |
| PC-A | 192.168.1.11+ | DHCP (auto-assigned) |
| PC-B | 192.168.1.12+ | DHCP (auto-assigned) |
| DNS Server | 8.8.8.8 | Google DNS |

---

## ⚙️ Step-by-Step Configuration

### Step 1 — Configure R1 interface

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
```

### Step 2 — Exclude static addresses from DHCP pool

```
R1(config)# ip dhcp excluded-address 192.168.1.1 192.168.1.10
```

> This excludes .1 to .10 — reserved for router, servers, printers with static IPs.

### Step 3 — Create DHCP Pool

```
R1(config)# ip dhcp pool LAN_POOL
R1(dhcp-config)# network 192.168.1.0 255.255.255.0
R1(dhcp-config)# default-router 192.168.1.1
R1(dhcp-config)# dns-server 8.8.8.8 8.8.4.4
R1(dhcp-config)# domain-name ccna-lab.local
R1(dhcp-config)# lease 7
R1(dhcp-config)# exit

R1(config)# end
R1# write memory
```

### Step 4 — Set PCs to DHCP

In Packet Tracer:
- PC-A → Desktop → IP Configuration → Select **DHCP**
- PC-B → Desktop → IP Configuration → Select **DHCP**

Both should receive IPs starting from 192.168.1.11

---

## ✅ Verification Commands

```
R1# show ip dhcp binding
R1# show ip dhcp pool
R1# show ip dhcp statistics
R1# show ip dhcp conflict
R1# show running-config | section dhcp
```

### Expected Output — `show ip dhcp binding`

```
Bindings from all pools not associated with VRF:
IP address       Client-ID/              Lease expiration        Type
                 Hardware address/
                 User name
192.168.1.11     0100.AAAA.BBBB.CC       Apr 07 2026 12:00 AM    Automatic
192.168.1.12     0100.CCCC.DDDD.EE       Apr 07 2026 12:00 AM    Automatic
```

### Expected Output — `show ip dhcp pool`

```
Pool LAN_POOL :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0
 Total addresses                : 254
 Leased addresses               : 2
 Pending event                  : none
 1 subnet is currently in the context :
  subnet 192.168.1.0 mask 255.255.255.0
   Range of addresses to hand out:
   192.168.1.11 - 192.168.1.254
```

---

## 🧪 Testing

On PC-A (after setting to DHCP):
```
PC-A> ipconfig          (should show 192.168.1.11 or higher)
PC-A> ping 192.168.1.1  (ping default gateway — should WORK ✅)
```

---

## 💡 Key Concepts

| Concept | Explanation |
|---------|-------------|
| DHCP | Dynamic Host Configuration Protocol — automatically assigns IP config |
| DORA Process | **D**iscover → **O**ffer → **R**equest → **A**cknowledge |
| `excluded-address` | IPs reserved for static devices — excluded from the pool |
| `default-router` | The gateway IP given to DHCP clients |
| `dns-server` | DNS server IP given to clients |
| `lease` | How many days the IP is reserved for the client |

### DORA Process Explained

```
PC (Client)                      R1 (Server)
    |                                |
    |--- DHCPDISCOVER (broadcast) -->|   "Anyone have an IP for me?"
    |<-- DHCPOFFER ------------------|   "Yes! Here's 192.168.1.11"
    |--- DHCPREQUEST (broadcast) --> |   "I want 192.168.1.11 please"
    |<-- DHCPACK --------------------|   "It's yours for 7 days!"
    |                                |
```

---

## 📝 Review Questions

1. What are the 4 steps in the DORA process?
2. Why do we use `ip dhcp excluded-address`?
3. What happens when the DHCP lease expires?
4. How would you configure a DHCP helper address for a remote subnet?
5. What does `show ip dhcp conflict` show?

---

## ✅ Lab Completion Checklist

- [ ] R1 Gi0/0 configured with 192.168.1.1/24
- [ ] Addresses 1-10 excluded from DHCP pool
- [ ] DHCP pool `LAN_POOL` created with all options
- [ ] PC-A receives IP via DHCP (192.168.1.11+)
- [ ] PC-B receives different IP via DHCP
- [ ] `show ip dhcp binding` shows both clients
- [ ] PC-A can ping R1 (192.168.1.1)

---

*[← Lab 10](./Lab-10-NAT-PAT.md) | Lab 11 of 12 | [Next Lab →](./Lab-12-Port-Security.md) | [Back to Index](./README.md)*
