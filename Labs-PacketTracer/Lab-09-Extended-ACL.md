# Lab-09 – Extended ACLs

**Topic:** Access Control Lists | **Difficulty:** Intermediate | **Time:** ~40 mins

---

## 🎯 Objectives

- Create extended ACLs filtering on source IP, destination IP, protocol, and port
- Permit only HTTP (80) and HTTPS (443) traffic to web server
- Block all other traffic with logging
- Place extended ACL close to the source

---

## 🖧 Topology

```
LAN (192.168.1.0/24)              DMZ (10.0.0.0/24)
                      
PC-A ---+                         Web Server (10.0.0.10) — TCP 80, 443
        |--- SW1 ---Gi0/0--- R1 ---Gi0/1--- FTP Server (10.0.0.20) — TCP 21
PC-B ---+
                  ↑
          ACL applied INBOUND here
          (close to source — Extended ACL rule)
```

---

## 🧰 Device Table

| Device | IP Address | Role |
|--------|------------|------|
| PC-A | 192.168.1.10/24 | User |
| PC-B | 192.168.1.20/24 | User |
| R1 Gi0/0 | 192.168.1.1/24 | LAN Gateway |
| R1 Gi0/1 | 10.0.0.1/24 | DMZ Gateway |
| Web Server | 10.0.0.10/24 | Allows HTTP/HTTPS |
| FTP Server | 10.0.0.20/24 | Should be blocked |

---

## ⚙️ Step-by-Step Configuration

### Step 1 — Configure R1 interfaces

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit

R1(config)# interface GigabitEthernet0/1
R1(config-if)# ip address 10.0.0.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit
```

### Step 2 — Create Extended Named ACL

```
R1(config)# ip access-list extended LAN_TO_DMZ
R1(config-ext-nacl)# remark Permit HTTP to Web Server
R1(config-ext-nacl)# permit tcp 192.168.1.0 0.0.0.255 host 10.0.0.10 eq 80
R1(config-ext-nacl)# remark Permit HTTPS to Web Server
R1(config-ext-nacl)# permit tcp 192.168.1.0 0.0.0.255 host 10.0.0.10 eq 443
R1(config-ext-nacl)# remark Permit DNS queries
R1(config-ext-nacl)# permit udp 192.168.1.0 0.0.0.255 any eq 53
R1(config-ext-nacl)# remark Permit ICMP for ping/troubleshooting
R1(config-ext-nacl)# permit icmp 192.168.1.0 0.0.0.255 any
R1(config-ext-nacl)# remark Block everything else and log it
R1(config-ext-nacl)# deny ip any any log
R1(config-ext-nacl)# exit
```

### Step 3 — Apply ACL inbound on Gi0/0 (close to source)

```
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip access-group LAN_TO_DMZ in
R1(config-if)# exit

R1(config)# end
R1# write memory
```

---

## ✅ Verification Commands

```
R1# show ip access-lists
R1# show ip access-lists LAN_TO_DMZ
R1# show ip interface GigabitEthernet0/0
R1# show running-config | section access-list
```

### Expected Output — `show ip access-lists LAN_TO_DMZ`

```
Extended IP access list LAN_TO_DMZ
    10 permit tcp 192.168.1.0 0.0.0.255 host 10.0.0.10 eq www (12 matches)
    20 permit tcp 192.168.1.0 0.0.0.255 host 10.0.0.10 eq 443 (8 matches)
    30 permit udp 192.168.1.0 0.0.0.255 any eq domain (4 matches)
    40 permit icmp 192.168.1.0 0.0.0.255 any (6 matches)
    50 deny ip any any log (3 matches)
```

---

## 🧪 Testing

```
PC-A> ping 10.0.0.10          → Should WORK ✅  (ICMP permitted)
PC-A> ping 10.0.0.20          → Should WORK ✅  (ICMP permitted)

From browser on PC-A:
http://10.0.0.10              → Should WORK ✅  (TCP 80 permitted)
ftp://10.0.0.20               → Should FAIL ❌  (TCP 21 not permitted — logged)
```

---

## 💡 Key Concepts

| Concept | Explanation |
|---------|-------------|
| Extended ACL | Filters on src IP + dst IP + protocol + port — numbers 100-199 or 2000-2699 |
| `eq 80` | Equals port 80 (HTTP) |
| `eq 443` | Equals port 443 (HTTPS) |
| `eq 53` | Equals port 53 (DNS) |
| `log` keyword | Logs matched packets to syslog — useful for security monitoring |
| Inbound ACL | Best placement for Extended ACLs — close to SOURCE |

### Extended ACL Syntax

```
access-list [number] [permit|deny] [protocol] [src] [src-wildcard] [dst] [dst-wildcard] [eq port]
```

### Common Port Numbers to Know for CCNA

| Port | Protocol | Service |
|------|----------|---------|
| 20, 21 | TCP | FTP |
| 22 | TCP | SSH |
| 23 | TCP | Telnet |
| 25 | TCP | SMTP |
| 53 | UDP/TCP | DNS |
| 67, 68 | UDP | DHCP |
| 80 | TCP | HTTP |
| 110 | TCP | POP3 |
| 443 | TCP | HTTPS |

---

## 📝 Review Questions

1. What is the main difference between Standard and Extended ACLs?
2. Why do we place Extended ACLs close to the SOURCE?
3. What does the `log` keyword do at the end of an ACL entry?
4. How would you permit only SSH (port 22) from one specific host?
5. What is the wildcard mask for a /24 subnet?

---

## ✅ Lab Completion Checklist

- [ ] R1 interfaces up and routing works
- [ ] Extended ACL `LAN_TO_DMZ` created with 5 rules
- [ ] ACL applied INBOUND on Gi0/0
- [ ] HTTP to Web Server works (port 80 permitted)
- [ ] FTP to FTP Server blocked (port 21 denied + logged)
- [ ] ICMP/ping still works (troubleshooting allowed)

---

*[← Lab 08](./Lab-08-Standard-ACL.md) | Lab 09 of 12 | [Next Lab →](./Lab-10-NAT-PAT.md) | [Back to Index](./README.md)*
