# 11 — Network Automation & Programmability: Complete Reference

> **CCNA Exam Domain:** 6.0 Automation and Programmability  
> **Exam Weight:** ~10% of total exam  
> **Difficulty:** ⭐⭐⭐☆☆

---

## Table of Contents

1. [Why Network Automation?](#1-why-network-automation)
2. [Traditional vs Controller-Based Networking](#2-traditional-vs-controller-based-networking)
3. [The Three Planes of Networking](#3-the-three-planes-of-networking)
4. [SDN — Software Defined Networking](#4-sdn--software-defined-networking)
5. [Cisco DNA Center & Intent-Based Networking](#5-cisco-dna-center--intent-based-networking)
6. [REST APIs — How Automation Talks to Networks](#6-rest-apis--how-automation-talks-to-networks)
7. [Data Formats — JSON, XML, YAML](#7-data-formats--json-xml-yaml)
8. [NETCONF & RESTCONF](#8-netconf--restconf)
9. [YANG Data Models](#9-yang-data-models)
10. [Configuration Management Tools](#10-configuration-management-tools)
11. [Key Takeaways & Exam Tips](#11-key-takeaways--exam-tips)

---

## 1. Why Network Automation?

### The Problem with Manual Network Management

To understand why network automation has become one of the most important trends in networking, you first need to appreciate the scale problem that modern network engineers face. Picture a company with 300 branch offices, each with a router, two access switches, and a wireless controller. That is 900 physical devices. Now imagine your security team tells you: "We need to update the NTP configuration on every device and add a new ACL to all routers — by end of day."

Without automation, the only option is for engineers to SSH into each device individually, type the same commands hundreds of times, and hope they do not make a typo on device number 247 at 11 PM. This process is slow (weeks, not hours), error-prone (humans make mistakes especially when fatigued), and completely undocumented unless someone manually keeps a spreadsheet. When you are done, you have no reliable way to verify that all 900 devices received the correct change.

The consequences of this manual approach extend beyond inconvenience. Each device that sits on an older, unpatched IOS version because "we haven't had time to upgrade it" is a potential security vulnerability. Each inconsistency between device configurations — where router-A has the ACL slightly differently than router-B because two different engineers configured them at different times — creates unexpected behavior that is incredibly difficult to troubleshoot.

Network automation solves all of these problems by treating network configuration the same way software developers treat code: as something that is written once, tested, version-controlled, and deployed consistently and repeatably across any number of devices.

```
Manual Management — The Math Problem:
  300 routers × 15 CLI commands each = 4,500 commands to type
  Average typing speed with SSH latency: 2 minutes per device
  Total time: 300 × 2 = 600 minutes = 10 HOURS of work
  Probability of zero errors across 4,500 commands: very low

Automated Management — The Solution:
  Write automation script once: 30 minutes
  Test on 5 devices: 10 minutes
  Deploy to all 300 devices: 3 minutes
  Verify all devices received correct config: 2 minutes
  Total time: ~45 minutes, zero human typing errors
```

### The Four Pillars of Network Automation Value

**Speed** is the most immediately visible benefit. Changes that used to take a team of engineers a week to implement can be deployed in minutes. This matters especially for security — when a vulnerability is discovered, you want to patch it in hours, not days.

**Consistency** is perhaps more valuable than speed in the long run. Automation applies identical configuration to every device without variation. There are no "I thought I typed that" moments, no differences between how the morning shift configured something versus the evening shift. Consistent configuration means predictable, debuggable networks.

**Scalability** means that managing 3,000 devices takes almost the same effort as managing 300, because the automation does the work. Human effort scales with the number of devices when manual; automation means effort stays roughly constant as the network grows.

**Accuracy and auditability** come from treating configuration as code — storing it in version control (like Git), reviewing changes before deployment, and maintaining a complete history of every change ever made. If something breaks, you can see exactly what changed and roll it back.

---

## 2. Traditional vs Controller-Based Networking

### How Traditional Networks Are Managed

In traditional networking, intelligence is distributed. Every router and switch contains its own complete control plane — it makes its own routing decisions, runs its own spanning tree calculations, and stores its own configuration. When you make a change, you must go to each device individually.

This distributed approach has served networking well for decades, and it has genuine advantages. It is resilient — if the network management system goes offline, the devices keep functioning because they carry their own intelligence. It is also simple to understand conceptually: each device is a self-contained unit.

But at scale, the distributed model creates serious operational problems. There is no single source of truth for what the network looks like. Different devices may have subtly inconsistent configurations. Implementing consistent policy across hundreds of devices requires careful coordination. And getting a holistic view of network state requires querying devices individually and mentally assembling the picture.

```
Traditional Network — Distributed Intelligence:

[Router-A]         [Router-B]         [Router-C]
 ┌──────────┐       ┌──────────┐       ┌──────────┐
 │ Control  │       │ Control  │       │ Control  │
 │  Plane   │◄─────►│  Plane   │◄─────►│  Plane   │
 │(OSPF,STP)│       │(OSPF,STP)│       │(OSPF,STP)│
 ├──────────┤       ├──────────┤       ├──────────┤
 │  Data    │       │  Data    │       │  Data    │
 │  Plane   │       │  Plane   │       │  Plane   │
 │(forwarding)      │(forwarding)      │(forwarding)
 └──────────┘       └──────────┘       └──────────┘
 
 Management: SSH to each device separately
 Change propagation: Manual, device by device
 Consistency: Depends on human discipline
```

### How Controller-Based Networks Work

Controller-based networking centralizes the control plane into a dedicated **controller** — a software platform that has a complete, real-time view of the entire network and makes decisions on behalf of all the devices beneath it. The network devices themselves become simpler — they retain the data plane (the actual forwarding hardware) but offload the intelligence to the controller.

Think of it like the difference between a traditional army (where each soldier makes their own tactical decisions) and a modern army (where a command center has complete situational awareness and coordinates all forces). The command center model allows faster, more coordinated responses and ensures all elements of the network are working toward the same objective.

```
Controller-Based Network — Centralized Intelligence:

         ┌─────────────────────────────┐
         │       SDN CONTROLLER        │
         │  Complete network view       │
         │  Centralized control plane  │
         │  Policy engine              │
         │  REST API for management    │
         └──────────┬──────────────────┘
                    │ Southbound APIs
          ┌─────────┼──────────┐
          ▼         ▼          ▼
      [Device-A] [Device-B] [Device-C]
      Data Plane Data Plane Data Plane
      (forwarding)(forwarding)(forwarding)

Management: Configure controller once → pushed to all devices
Change propagation: Automatic, consistent, milliseconds
Consistency: Guaranteed by controller
```

---

## 3. The Three Planes of Networking

Understanding the three planes is fundamental to understanding SDN, because SDN is essentially the separation of these planes from each other. Every networking function falls into one of three categories.

### The Data Plane (Forwarding Plane)

The data plane is responsible for actually moving packets from one interface to another — it is the "doing" layer. When a packet arrives on an interface, the data plane looks up the destination in the forwarding table and sends the packet out the correct interface. This process happens billions of times per second on high-speed routers and must be done at wire speed.

Because performance is critical, the data plane is implemented in specialized hardware — ASICs (Application-Specific Integrated Circuits) in switches, and NPUs (Network Processing Units) in routers. These chips can make forwarding decisions in nanoseconds, which software running on a general-purpose CPU cannot match.

Examples of data plane functions include: IP packet forwarding based on the routing table, Ethernet frame switching based on the MAC address table, NAT address translation, QoS marking and queuing, ACL enforcement, and TTL decrement.

### The Control Plane

The control plane is responsible for building and maintaining the information that the data plane uses to make forwarding decisions. If the data plane is the factory floor that does the work, the control plane is the management office that decides what work needs to be done.

Control plane protocols include all routing protocols (OSPF, EIGRP, BGP — these build the routing table), Spanning Tree Protocol (which builds the MAC forwarding topology for Ethernet networks), ARP (which resolves IP addresses to MAC addresses), and LLDP/CDP (which discover neighbors). None of these protocols forward user data — they exist solely to populate tables that the data plane uses.

The control plane is computationally intensive but not time-critical in the same way as the data plane. It runs on the router's general-purpose CPU and can tolerate some delay in processing.

### The Management Plane

The management plane is how humans (and automation systems) interact with network devices to configure them, monitor them, and retrieve information. This includes SSH and Telnet sessions where engineers type CLI commands, SNMP polling by network management systems, NETCONF and RESTCONF APIs used by automation scripts, and syslog messages sent to log servers.

The management plane does not make forwarding decisions and does not build routing tables — it simply provides the interface through which the device is controlled and monitored.

```
┌────────────────────────────────────────────────────────────┐
│                   MANAGEMENT PLANE                          │
│  How humans/automation interact with the device             │
│  SSH, Telnet, SNMP, NETCONF, RESTCONF, Syslog              │
│  Runs on: CPU, low performance requirement                  │
├────────────────────────────────────────────────────────────┤
│                   CONTROL PLANE                             │
│  Builds the tables the data plane uses                      │
│  OSPF, EIGRP, BGP, STP, ARP, LLDP, CDP                     │
│  Runs on: CPU, moderate performance requirement             │
├────────────────────────────────────────────────────────────┤
│                   DATA PLANE                                │
│  Actually moves packets and frames                          │
│  IP forwarding, MAC switching, NAT, ACLs, QoS              │
│  Runs on: ASIC/NPU hardware, wire-speed requirement         │
└────────────────────────────────────────────────────────────┘
```

---

## 4. SDN — Software Defined Networking

### The Core SDN Concept

SDN is the architectural approach that **physically separates the control plane from the data plane**. In traditional networking these two planes run together on the same device. In SDN, the control plane is moved to a centralized software controller, while the physical network devices retain only the data plane.

This separation has profound implications. Because the controller has a complete view of the entire network topology, it can make globally optimal routing decisions rather than locally optimal ones. Because all policy is defined centrally, consistency is automatic. Because the devices become simpler (data plane only), they can be cheaper hardware.

The analogy that works well here is cloud computing. Traditional servers ran the operating system, application, database, and everything else on physical hardware that you owned. Cloud computing separated these concerns — compute, storage, and networking became virtualized, centralized, and managed through software. SDN applies the same concept to networking: the "intelligence" (control plane) moves to software running in a central location, while the physical network devices become like cloud compute instances — powerful but dumb, doing what the controller tells them.

### SDN Interfaces — North, South, East, West

The SDN controller sits between two worlds: the world of applications and business intent above it, and the world of physical network devices below it. Different interfaces handle communication in each direction.

**Northbound APIs** connect the controller to applications and automation systems above it. These APIs are how network automation scripts, orchestration platforms (like Cisco DNA Center), and third-party applications request network services from the controller. Northbound APIs are typically RESTful HTTP APIs that use JSON or XML to exchange data — they are designed to be easy to use from any programming language.

**Southbound APIs** connect the controller to the physical network devices below it. These APIs are how the controller programs the forwarding tables and configuration of switches and routers. Different southbound protocols have been developed for different purposes: OpenFlow is used to directly program flow tables in switches; NETCONF is used to configure devices using structured XML data; RESTCONF is a RESTful version of NETCONF; and traditional CLI/SNMP can also serve as southbound interfaces for legacy device support.

**Eastbound and Westbound APIs** (less commonly discussed on the CCNA) handle communication between multiple SDN controllers, allowing them to coordinate when the network spans multiple administrative domains.

```
                Applications / Automation Scripts
                           │
              ┌────────────▼────────────┐
              │  Northbound APIs        │
              │  (REST APIs, gRPC)      │
              ├─────────────────────────┤
              │                         │
              │    SDN CONTROLLER       │
              │  • Complete topology     │
              │  • Policy engine        │
              │  • Path computation     │
              │                         │
              ├─────────────────────────┤
              │  Southbound APIs        │
              │  (OpenFlow, NETCONF,    │
              │   RESTCONF, YANG)       │
              └────────────┬────────────┘
                           │
               ┌───────────┼───────────┐
               ▼           ▼           ▼
           [Switch-A]  [Switch-B]  [Router-C]
           Data Plane  Data Plane  Data Plane
```

### OpenFlow — The Original SDN Protocol

OpenFlow was the first and most discussed southbound protocol in early SDN. It gives the controller direct access to the flow tables inside switches, allowing the controller to install, modify, and delete individual flow entries.

An OpenFlow flow entry specifies a match condition (match on destination IP, or source MAC, or TCP port, or any combination) and an action to take when a packet matches (forward out port X, drop, send to controller for further processing). The controller programs these flow entries into the switches, effectively telling each switch exactly what to do with each type of traffic.

While OpenFlow is conceptually elegant and was enormously influential in advancing SDN thinking, it has seen limited deployment in production enterprise networks. Real-world SDN deployments more commonly use NETCONF/YANG and vendor-specific APIs, because these work better with existing networking hardware and protocols.

---

## 5. Cisco DNA Center & Intent-Based Networking

### What DNA Center Is

Cisco DNA Center (Cisco Digital Network Architecture Center) is Cisco's enterprise SDN controller and network management platform. It represents Cisco's implementation of a complete controller-based networking solution for enterprise campus networks.

DNA Center is important to understand because it embodies the concept of **Intent-Based Networking (IBN)** — a step beyond basic SDN. Where traditional SDN still requires the operator to specify HOW to implement a policy (configure this OSPF area, program this VLAN on these ports), IBN allows the operator to specify WHAT they want to achieve (all sales employees should have access to the CRM application with high-priority QoS, and no access to social media), and the system figures out how to implement it.

Think of the difference between telling a GPS "turn left on Main Street, then right on Oak Avenue, then left on Park Road" versus simply saying "take me to 123 Park Road." IBN is like the second approach — you express the business intent, and the system translates it into the precise technical configurations required.

### DNA Center Architecture

DNA Center runs as a software platform, typically on a dedicated appliance or on Cisco's cloud infrastructure. It communicates with network devices through the network's existing IP infrastructure — there is no need for a separate management network, though that is best practice.

```
┌────────────────────────────────────────────────────────────────┐
│                    Cisco DNA Center                             │
│                                                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐   │
│  │   Design    │  │   Policy     │  │     Assurance        │   │
│  │ • Topology  │  │ • QoS policy │  │ • AI/ML analytics   │   │
│  │ • IP Mgmt   │  │ • Segmentation│  │ • Path trace        │   │
│  │ • Templates │  │ • Security   │  │ • Client 360°       │   │
│  └─────────────┘  └──────────────┘  └─────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Northbound REST APIs                        │   │
│  │   (third-party integration, automation scripts)         │   │
│  └──────────────────────┬──────────────────────────────────┘   │
└─────────────────────────┼──────────────────────────────────────┘
                          │ NETCONF/YANG, SNMP, CLI (Southbound)
              ┌───────────┼───────────────────┐
              ▼           ▼                   ▼
       [Access SW]   [WAN Router]       [Wireless AP]
```

### DNA Center Key Capabilities

**Design** is where you define the network's physical and logical topology — floor maps, buildings, sites, IP address pools, and DNS/DHCP settings. You create device templates here that define the standard configuration for each device type in your network.

**Policy** is where IBN truly shines. You define business policies in plain terms — which users or devices can access which applications, with what priority, under what conditions — and DNA Center translates these into precise QoS configurations, ACLs, and VLANs pushed to the appropriate devices.

**Provision** handles deploying the configuration to devices — including zero-touch provisioning for new devices that automatically download their configuration when first connected.

**Assurance** uses AI and machine learning to continuously monitor network health, compare actual behavior to expected behavior, detect anomalies, and provide guided remediation when problems occur. The "path trace" tool is particularly valuable — you specify source and destination, and DNA Center shows you exactly which path the traffic takes, the health of each hop, and any policy that affects it.

### SD-Access — Cisco's Campus Fabric

Built on top of DNA Center, **SD-Access** is Cisco's complete campus network solution that replaces traditional VLAN-based network design with a more flexible, identity-based approach.

Traditional campus networks use VLANs to segment traffic — the Finance team is in VLAN 30, Sales is in VLAN 10, and ACLs between SVIs control who can talk to whom. The problem is that VLANs are tied to physical ports and subnets. If a Finance employee sits at a desk in the Sales area, they get Sales network access unless someone manually reconfigures their port.

SD-Access uses the user's or device's **identity** (from Active Directory, certificates, or 802.1X) to determine policy, regardless of physical location. The Finance employee gets Finance policy whether they plug in at their desk, a conference room, or a remote office — because the network recognizes their identity and applies the correct policy dynamically.

SD-Access uses VXLAN for the data plane overlay (to carry traffic between any two points in the campus fabric), LISP for the control plane (to track where each endpoint is located), and ISE (Identity Services Engine) for policy and authentication.

---

## 6. REST APIs — How Automation Talks to Networks

### What REST Is and Why It Matters

REST (Representational State Transfer) is an architectural style for designing networked applications. It is the dominant API design approach for web services and has become the standard way that network automation tools communicate with network controllers and devices.

Understanding REST is important not just as exam knowledge but as a practical skill. Every major network management platform — Cisco DNA Center, Meraki, NSO, ACI — exposes REST APIs. When you write an automation script that queries DNA Center for device inventory, configures an interface, or deploys a policy, you are making REST API calls.

The beauty of REST is its simplicity. It uses the same HTTP protocol your browser uses to load web pages, which means every programming language already has tools to make REST calls. You do not need specialized libraries or SDKs — any tool that can make HTTP requests can interact with a REST API.

### The Six REST Constraints

REST is defined by six architectural constraints that guide how APIs should be designed. You do not need to memorize all six for the CCNA exam, but understanding the key ones helps you understand why REST APIs work the way they do.

The most important constraint is **statelessness** — each REST API request must contain all the information needed to process it. The server does not store any session state between requests. This means every request must include authentication credentials (typically a token in the header), and the server treats each request independently. This is why you often need to obtain an authentication token first and then include it in every subsequent request.

**Uniform interface** means that all REST APIs follow the same conventions — you use the same HTTP methods, the same URL structure, the same status codes — regardless of which system you are interacting with. Once you learn REST, you can work with any REST API with minimal additional learning.

### HTTP Methods — The Verbs of REST

REST uses standard HTTP methods as its verbs. Each method has a specific, well-defined meaning that you must understand for the exam.

**GET** retrieves information without modifying anything. It is the read operation. When you want to know the current state of an interface, retrieve a list of devices, or check what VLANs exist, you use GET. GET requests are safe — they should never cause any changes on the server.

**POST** creates a new resource. When you want to add a new VLAN, create a new user, or add a network device, you use POST. The body of a POST request contains the data for the new resource in JSON or XML format.

**PUT** replaces an entire existing resource with new data. When you want to completely overwrite an interface's configuration, you use PUT. It is important to understand that PUT replaces the entire resource — if you PUT a new interface configuration that omits some fields, those fields may be deleted.

**PATCH** partially updates an existing resource — you provide only the fields you want to change, and the rest remain as they were. This is more precise and efficient than PUT when you only need to change one or two attributes of a large object.

**DELETE** removes a resource. When you want to delete a VLAN, remove a device from inventory, or revoke an access policy, you use DELETE.

```
HTTP Method   CRUD Operation   Example Use Case
───────────   ──────────────   ─────────────────────────────────────────
GET           Read             Get list of all devices in DNA Center
POST          Create           Add a new VLAN (VLAN 50, name "Marketing")
PUT           Update/Replace   Replace entire interface configuration
PATCH         Partial Update   Change only the description on an interface
DELETE        Delete           Remove VLAN 50 from the network

URL structure (REST uses URLs to identify resources):
  Base URL:       https://dnac-ip/dna/intent/api/v1/
  Resource type:  network-device
  Specific item:  network-device/{device-id}

Examples:
  GET  /dna/intent/api/v1/network-device         → list all devices
  GET  /dna/intent/api/v1/network-device/abc123  → get device abc123
  POST /dna/intent/api/v1/vlan                   → create new VLAN
  PUT  /dna/intent/api/v1/vlan/50                → replace VLAN 50 config
  DELETE /dna/intent/api/v1/vlan/50              → delete VLAN 50
```

### HTTP Status Codes — Understanding Responses

When you make a REST API call, the server always responds with an HTTP status code that tells you whether the request succeeded and, if not, what went wrong. The exam tests knowledge of these codes, and they are also essential for writing automation scripts that handle errors gracefully.

Status codes are organized into ranges. The **2xx range** (200–299) means success. The **4xx range** (400–499) means the client made an error — the request was malformed or unauthorized. The **5xx range** (500–599) means the server encountered an error — the problem is on the server side, not in your request.

```
Code    Meaning                  When You See It
──────  ──────────────────────   ────────────────────────────────────────────
200     OK                       Successful GET or PATCH request
201     Created                  Successful POST — new resource was created
204     No Content               Successful DELETE — no body returned
400     Bad Request              Your JSON/XML was malformed or missing fields
401     Unauthorized             You did not provide authentication credentials
403     Forbidden                Credentials OK but you lack permission for this
404     Not Found                The resource URL does not exist
405     Method Not Allowed       You used GET on a write-only endpoint, etc.
409     Conflict                 Resource already exists (duplicate POST)
500     Internal Server Error    Bug or crash on the server side
503     Service Unavailable      Server is overloaded or in maintenance
```

### A Real REST API Example — DNA Center

This example walks through a typical automation workflow with Cisco DNA Center, showing exactly how REST API calls work in practice.

```python
import requests  # Python library for making HTTP requests
import json

# Step 1: Disable SSL warnings (in production, use proper certificates!)
requests.packages.urllib3.disable_warnings()
DNAC = "https://10.0.0.100"  # DNA Center IP address

# Step 2: Authenticate — get a token (all subsequent requests use this token)
auth_response = requests.post(
    f"{DNAC}/dna/system/api/v1/auth/token",
    auth=("admin", "Admin@Password123"),  # username and password
    headers={"Content-Type": "application/json"},
    verify=False  # skip SSL cert validation (bad practice in production)
)

# The response contains a JSON object with a "Token" field
token = auth_response.json()["Token"]
print(f"Got auth token: {token[:20]}...")  # print first 20 chars

# Step 3: Use the token in all subsequent requests via a header
headers = {
    "x-auth-token": token,           # DNA Center auth header
    "Content-Type": "application/json"
}

# Step 4: GET — retrieve list of all network devices
devices_response = requests.get(
    f"{DNAC}/dna/intent/api/v1/network-device",
    headers=headers,
    verify=False
)

# HTTP 200 means success
print(f"Status code: {devices_response.status_code}")  # should print 200

# Parse the JSON response body
devices = devices_response.json()["response"]  # list of device objects
for device in devices:
    # Each device is a dict with hostname, management IP, type, etc.
    print(f"Device: {device['hostname']:20s} | IP: {device['managementIpAddress']}")

# Step 5: GET — retrieve a specific device by its ID
device_id = devices[0]["id"]  # take the first device's ID
detail_response = requests.get(
    f"{DNAC}/dna/intent/api/v1/network-device/{device_id}",
    headers=headers,
    verify=False
)
device_detail = detail_response.json()["response"]
print(f"Software version: {device_detail['softwareVersion']}")

# Step 6: POST — add a new VLAN (example of creating a resource)
new_vlan = {
    "name": "Marketing",
    "id": "60",
    "description": "Marketing Department VLAN"
}
create_response = requests.post(
    f"{DNAC}/dna/intent/api/v1/vlan",
    headers=headers,
    json=new_vlan,   # json= parameter serializes dict to JSON automatically
    verify=False
)
# HTTP 201 = Created successfully
print(f"Create VLAN status: {create_response.status_code}")
```

---

## 7. Data Formats — JSON, XML, YAML

### Why Data Formats Matter

When automation systems, controllers, and network devices exchange information, they need a shared language — a standard way to structure and represent data so that the sender and receiver can both understand it. Three formats dominate network automation: JSON, XML, and YAML.

Understanding these formats is both an exam requirement and a daily practical need. When you look at an API response from DNA Center, it comes back as JSON. When you configure a device via NETCONF, you send XML. When you write an Ansible playbook, you write YAML. Being able to read and write each format fluently is a fundamental automation skill.

### JSON — JavaScript Object Notation

JSON has become the dominant data format for REST APIs because it strikes the right balance between human readability and machine parsability. It was originally designed for JavaScript but has become language-agnostic — every programming language has built-in JSON support.

JSON represents data using two structures: **objects** (collections of key-value pairs, enclosed in curly braces) and **arrays** (ordered lists of values, enclosed in square brackets). These two structures can be nested inside each other to represent any level of complexity.

JSON supports six data types, and knowing these types is important because they affect how you work with API responses. A string must always be enclosed in double quotes. A number has no quotes. A boolean is literally `true` or `false` (lowercase, no quotes). Null represents an absent value. An object is a key-value collection. An array is an ordered list.

```json
{
  "device": {
    "hostname": "Core-Router-01",
    "managementIpAddress": "192.168.99.10",
    "softwareVersion": "16.9.5",
    "reachabilityStatus": "Reachable",
    "serialNumber": "FTX1234ABCD",
    "upTime": "52 days, 3 hours",
    "platformId": "ISR4431",
    "interfaces": [
      {
        "name": "GigabitEthernet0/0",
        "status": "up",
        "ipAddress": "192.168.1.1",
        "speed": 1000000000
      },
      {
        "name": "GigabitEthernet0/1",
        "status": "up",
        "ipAddress": "10.0.0.1",
        "speed": 1000000000
      }
    ],
    "managedByDNAC": true,
    "lastUpdated": null
  }
}
```

Looking at this JSON, you can identify the data types in use. "hostname" has a string value (note the double quotes around "Core-Router-01"). "speed" has a numeric value (1000000000 — no quotes). "managedByDNAC" has a boolean value (true — no quotes, lowercase). "lastUpdated" has a null value. "interfaces" has an array value (the square brackets contain multiple objects). "device" itself is an object (the curly braces contain key-value pairs).

The key rule to remember: **strings always require double quotes in JSON, but numbers, booleans, and null never use quotes.** Mixing these up is a very common source of JSON syntax errors.

### XML — Extensible Markup Language

XML uses tags (similar to HTML) to structure data. Every piece of data is wrapped in opening and closing tags, and tags can be nested to create hierarchy. XML is more verbose than JSON but supports features like namespaces, schemas, and attributes that make it suitable for complex document formats.

XML is the format used by NETCONF (the network configuration protocol), which is why network engineers encounter it even in environments that primarily use JSON for REST APIs.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<device>
    <hostname>Core-Router-01</hostname>
    <managementIpAddress>192.168.99.10</managementIpAddress>
    <softwareVersion>16.9.5</softwareVersion>
    <reachabilityStatus>Reachable</reachabilityStatus>
    <interfaces>
        <interface>
            <name>GigabitEthernet0/0</name>
            <status>up</status>
            <ipAddress>192.168.1.1</ipAddress>
            <speed>1000000000</speed>
        </interface>
        <interface>
            <name>GigabitEthernet0/1</name>
            <status>up</status>
            <ipAddress>10.0.0.1</ipAddress>
            <speed>1000000000</speed>
        </interface>
    </interfaces>
    <managedByDNAC>true</managedByDNAC>
</device>
```

Notice how XML represents the same information as the JSON above, but uses tags instead of key-value pairs. The structure is equivalent — "interfaces" contains multiple "interface" elements just as the JSON array contains multiple objects — but XML requires explicit closing tags for every opening tag, making it significantly more verbose.

XML also supports **attributes** — additional information added directly inside a tag's angle brackets:

```xml
<interface id="1" type="physical">
    <name>GigabitEthernet0/0</name>
</interface>
```

Here `id="1"` and `type="physical"` are attributes of the interface element. YANG data models often use attributes to carry metadata.

### YAML — YAML Ain't Markup Language

YAML is the most human-readable of the three formats and is the primary format for configuration files in automation tools like Ansible, Kubernetes, and Docker Compose. Where JSON uses curly braces, colons, and quote marks to structure data, YAML uses indentation alone — it looks almost like plain English.

The critical rule in YAML is that **indentation creates structure**, and indentation must use spaces, never tabs. A YAML parsing error caused by a tab instead of spaces is one of the most frustrating debugging experiences in automation, so develop the habit of checking your editor's whitespace settings.

```yaml
# YAML uses # for comments (JSON has no comment syntax)
device:
  hostname: Core-Router-01          # string (no quotes needed)
  managementIpAddress: 192.168.99.10
  softwareVersion: "16.9.5"         # quotes optional for strings
  reachabilityStatus: Reachable
  managedByDNAC: true               # boolean (lowercase, no quotes)
  lastUpdated: null                  # null value
  
  interfaces:                        # list starts with - for each item
    - name: GigabitEthernet0/0
      status: up
      ipAddress: 192.168.1.1
      speed: 1000000000             # number (no quotes)
    
    - name: GigabitEthernet0/1
      status: up
      ipAddress: 10.0.0.1
      speed: 1000000000
```

The YAML above represents the identical data as the JSON and XML examples, but it is visibly more readable. A network engineer who has never seen YAML can read and understand this immediately. The `- ` prefix before `name: GigabitEthernet0/0` indicates that this item is part of a list (like a JSON array).

### Three-Format Comparison

```
Feature            JSON                    XML                     YAML
────────────────   ─────────────────────   ─────────────────────   ──────────────────────
Readability        Good                    Fair (verbose)          Excellent
Verbosity          Medium                  High (tags everywhere)  Lowest
Comments           Not supported           Yes (<!-- -->)          Yes (# symbol)
Data types         Native (string, num,    All strings by default  Native types
                   bool, null, array, obj) (attributes help)
Primary use        REST APIs               NETCONF protocol        Ansible, config files
Parser available   Universal               Universal               Universal
Whitespace matters No                      No                      YES (indentation!)
Schema support     JSON Schema             DTD, XSD (strong)       None formal
Learning curve     Easy                    Moderate                Easy
```

---

## 8. NETCONF & RESTCONF

### NETCONF — Network Configuration Protocol

NETCONF (RFC 6241) is a protocol specifically designed for configuring network devices programmatically. It was created to address the fundamental problems with using the CLI for automation: CLI output is designed for human reading (unstructured text), changes are hard to validate before applying, and there is no built-in way to roll back a failed change.

NETCONF uses **XML** to encode both the configuration data and the protocol operations. It runs over **SSH** (using TCP port 830), providing inherent security through SSH's encryption and authentication. NETCONF supports configuration validation, transaction-based changes (apply all or nothing), and a candidate configuration (make changes in a staging area, then commit them all at once or roll back).

The key concept in NETCONF is the **datastore** — a named repository of configuration or state data. The most important datastores are the **running** datastore (the active configuration currently applied to the device), the **candidate** datastore (a staging area where you can make changes before committing them), and the **startup** datastore (the configuration loaded on boot).

```
NETCONF Operations:
  <get>           → Retrieve running config AND operational state
  <get-config>    → Retrieve configuration only (from a specific datastore)
  <edit-config>   → Modify configuration (in candidate or running datastore)
  <commit>        → Apply candidate config to running (transaction commit)
  <discard-changes> → Throw away candidate config changes (rollback)
  <delete-config> → Delete a datastore
  <lock>          → Lock a datastore to prevent concurrent changes
  <unlock>        → Release a datastore lock
  <close-session> → End the NETCONF session

NETCONF Example — getting interface config in XML:
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <get-config>
    <source>
      <running/>           ← get from running configuration
    </source>
    <filter type="subtree">
      <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
        <interface>
          <name>GigabitEthernet0/0</name>  ← filter: only this interface
        </interface>
      </interfaces>
    </filter>
  </get-config>
</rpc>
```

### RESTCONF — REST-Based Network Configuration

RESTCONF (RFC 8040) is a newer protocol that provides the same functionality as NETCONF but through a REST API interface instead of the XML-over-SSH NETCONF protocol. RESTCONF uses **HTTP/HTTPS** (typically port 443) and supports both **JSON and XML** encoding, making it much easier to use with modern programming tools and automation frameworks.

RESTCONF maps NETCONF concepts to REST concepts: NETCONF datastores become URL paths, NETCONF operations become HTTP methods (GET for get-config, POST for create, PUT/PATCH for edit-config, DELETE for delete). This makes RESTCONF immediately approachable for developers already familiar with REST APIs.

```
RESTCONF URL structure:
  https://{device}/restconf/data/{yang-module}:{container}/{leaf}

Examples:
  GET interfaces:
  GET https://router/restconf/data/ietf-interfaces:interfaces

  GET specific interface:
  GET https://router/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet0%2F0

  PATCH interface description:
  PATCH https://router/restconf/data/ietf-interfaces:interfaces/interface=GigabitEthernet0%2F0

  Body (JSON):
  {
    "ietf-interfaces:interface": {
      "name": "GigabitEthernet0/0",
      "description": "Updated via RESTCONF automation"
    }
  }

  DELETE an interface:
  DELETE https://router/restconf/data/ietf-interfaces:interfaces/interface=Loopback99
```

### NETCONF vs RESTCONF Comparison

```
Feature              NETCONF                    RESTCONF
────────────────     ─────────────────────────  ─────────────────────────────
Transport            SSH (TCP 830)              HTTPS (TCP 443)
Data encoding        XML only                   JSON or XML
Ease of use          Requires XML knowledge     Familiar to REST API users
Transaction support  Yes (candidate + commit)   Partial (depends on device)
Operations model     Explicit operations         HTTP methods (GET/POST/PUT)
Standard             RFC 6241                   RFC 8040
Use case             Traditional automation     Modern automation, scripting
Tool support         Ncclient Python library    Any HTTP library (requests)
```

---

## 9. YANG Data Models

### What YANG Is

YANG (Yet Another Next Generation, RFC 7950) is a **data modeling language** used to define the structure, syntax, and semantics of configuration and operational data for network devices. If NETCONF and RESTCONF are the protocols for exchanging data, YANG is the grammar that defines what that data can look like.

The relationship is analogous to a database. SQL is the language for querying and modifying a database (like NETCONF/RESTCONF is the protocol for managing devices), but a database schema defines what tables and columns exist and what data types they hold (like YANG defines what configuration and state data a device supports).

Every interface you can configure via NETCONF or RESTCONF is defined by a YANG model. The model specifies: this device has a container called "interfaces" that contains a list called "interface", each interface has a leaf called "name" (type string), a leaf called "enabled" (type boolean), a container called "ipv4" that contains an address, and so on.

```
YANG Module (simplified example showing structure):

module ietf-interfaces {
  
  // The module defines a "interfaces" container
  container interfaces {
    
    // Inside, there's a list of interface objects
    list interface {
      key "name";           // "name" uniquely identifies each interface
      
      leaf name {
        type string;        // name is a text string
        description "The interface name, e.g. GigabitEthernet0/0";
      }
      
      leaf description {
        type string;        // description is optional text
      }
      
      leaf enabled {
        type boolean;       // true = no shutdown, false = shutdown
        default true;
      }
      
      container ipv4 {      // container groups IPv4-related config
        list address {
          key "ip";
          
          leaf ip {
            type inet:ipv4-address;    // must be valid IPv4 address
          }
          leaf prefix-length {
            type uint8 {
              range "0..32";           // constrained to 0-32
            }
          }
        }
      }
    }
  }
}
```

YANG models are standardized by the IETF — the same YANG model can describe the same configuration across devices from different vendors, enabling vendor-agnostic automation. When you use RESTCONF to configure an interface on a Cisco router, you use the same IETF YANG model you would use on a Juniper router. This is a significant advance over CLI-based automation, where every vendor has a completely different command syntax.

---

## 10. Configuration Management Tools

### The Role of Configuration Management

Configuration management tools extend the concept of automation beyond simple API calls to provide a complete framework for managing network (and server) configuration at scale. They provide features like idempotency (running the same script twice produces the same result — it doesn't double-apply changes), templating (define a standard configuration template and fill in device-specific values), inventory management (maintain a list of devices to manage), and version-controlled configuration (integrate with Git to track all changes).

### Ansible — The Network Automation Leader

Ansible is currently the most widely used configuration management tool for network automation. Its design philosophy prioritizes simplicity and accessibility — you do not need to be a software developer to use Ansible effectively.

The most important characteristic of Ansible is that it is **agentless**. Unlike some other tools, Ansible does not require any software to be installed on the devices it manages. It communicates with network devices using SSH (for CLI-based automation) or NETCONF/RESTCONF APIs (for modern model-driven automation). This makes Ansible immediately usable with existing network devices without any prerequisites.

Ansible is built around **playbooks** — YAML files that describe the desired state of the network. A playbook contains one or more "plays," each of which targets a group of devices (defined in the inventory) and specifies a series of tasks to execute on those devices. Each task uses a **module** — a pre-built unit of automation that knows how to perform a specific action (configure an interface, manage a VLAN, push an ACL).

The concept of **idempotency** is fundamental to Ansible. An idempotent operation produces the same result whether it is applied once or a hundred times. If you run an Ansible playbook that configures a VLAN and the VLAN already exists with the correct settings, Ansible detects this and does nothing — it does not add the VLAN again or generate an error. If the VLAN is missing or misconfigured, Ansible fixes it. This means you can run playbooks repeatedly (for compliance checking, for example) without worrying about unintended side effects.

```yaml
# Example Ansible Playbook — Configure VLANs and interfaces on campus switches
---
- name: Configure Campus Access Switches      # name of this play
  hosts: access_switches                      # target: devices in "access_switches" group
  gather_facts: no                            # skip automatic fact gathering
  
  vars:                                       # variables used in this play
    vlans:
      - { id: 10, name: Sales }
      - { id: 20, name: HR }
      - { id: 30, name: Finance }
      - { id: 99, name: Management }
  
  tasks:
    # Task 1: Ensure all VLANs exist
    - name: Create VLANs
      cisco.ios.ios_vlans:                    # Cisco IOS VLAN module
        config:
          - vlan_id: "{{ item.id }}"          # {{ }} = Jinja2 template variable
            name: "{{ item.name }}"
        state: merged                         # merged = add if missing, don't delete extras
      loop: "{{ vlans }}"                     # loop runs this task for each VLAN
    
    # Task 2: Configure access ports
    - name: Configure access ports for Sales VLAN
      cisco.ios.ios_l2_interfaces:
        config:
          - name: GigabitEthernet0/1
            access:
              vlan: 10
            mode: access
        state: replaced
    
    # Task 3: Configure management SVI
    - name: Configure management SVI
      cisco.ios.ios_l3_interfaces:
        config:
          - name: Vlan99
            ipv4:
              - address: "192.168.99.{{ mgmt_octet }}/24"  # device-specific var
        state: merged
    
    # Task 4: Save configuration
    - name: Save running config to startup config
      cisco.ios.ios_command:
        commands: write memory
```

The inventory file tells Ansible which devices exist and how to connect to them:

```ini
# inventory.ini — defines the groups and devices Ansible manages

[access_switches]          # group name
sw-floor1 ansible_host=192.168.99.11 mgmt_octet=11
sw-floor2 ansible_host=192.168.99.12 mgmt_octet=12
sw-floor3 ansible_host=192.168.99.13 mgmt_octet=13

[core_switches]
sw-core1 ansible_host=192.168.99.1
sw-core2 ansible_host=192.168.99.2

[all:vars]                 # variables that apply to ALL devices
ansible_user=admin
ansible_password=Admin@Pass
ansible_network_os=ios     # tells Ansible these are Cisco IOS devices
ansible_connection=network_cli  # use CLI connection method
```

### Puppet — Agent-Based Configuration Management

Puppet takes a different approach from Ansible. Rather than pushing changes from a central system, Puppet installs an **agent** on each managed device that regularly "checks in" with the Puppet Master server and downloads the desired configuration. This is called a **pull** model — devices pull their configuration from the server.

The advantage of the pull model is that devices continuously enforce their own configuration. If someone manually changes a setting on a device, the Puppet agent will detect the drift from desired state and automatically correct it on the next check-in cycle. This self-healing property is valuable in environments where configuration drift is a concern.

Puppet uses its own domain-specific language for describing desired state. The configuration files are called **manifests**, and collections of manifests are organized into **modules**. A Puppet module typically includes manifests (desired state definitions), templates (for generating configuration files), and files (static files to distribute).

The practical limitation for network device management is that Puppet agents need to be installable on managed devices — something that is possible on Linux servers but not straightforward on traditional network device operating systems like Cisco IOS or NX-OS. For this reason, Puppet is less commonly used for network device automation than Ansible, though it is prevalent in server infrastructure management.

### Chef — Enterprise Configuration Management

Chef shares a similar agent-based pull architecture with Puppet but uses **Ruby** as its configuration language. Chef's terminology differs: configuration files are called **recipes** (collections of resources that describe desired state), and related recipes are grouped into **cookbooks**. A **Chef Server** acts as the central repository, and nodes (managed systems) run the **chef-client** agent that checks in with the server.

Chef is known for being powerful and flexible, particularly for complex orchestration scenarios, but it has a steeper learning curve than both Ansible and Puppet due to the Ruby-based DSL. It is predominantly used for server infrastructure management in large enterprises.

### Tool Comparison

```
Feature              Ansible            Puppet             Chef
────────────────     ─────────────────  ─────────────────  ─────────────────────
Architecture         Agentless          Agent-based        Agent-based
Model                Push               Pull               Pull
Language             YAML (playbooks)   Puppet DSL         Ruby
Learning curve       Low                Moderate           High
Network device use   Excellent          Limited            Limited
Self-healing         Manual (re-run)    Automatic          Automatic
Primary use case     Network + servers  Server infra       Enterprise servers
Installation         Simple             Complex            Complex
Community/modules    Very large         Large              Large
Idempotent           Yes                Yes                Yes
```

---

## 11. Key Takeaways & Exam Tips

### Key Takeaways

Network automation addresses the fundamental scalability problem of manual device-by-device management. The value of automation comes from speed (minutes instead of days), consistency (no human error or variation), scalability (1000 devices takes the same effort as 10), and accuracy (changes are tested and repeatable).

The three planes of networking serve distinct purposes: the **data plane** forwards packets at wire speed using ASICs; the **control plane** builds routing and forwarding tables using protocols like OSPF and STP; and the **management plane** provides the interface for configuration and monitoring via SSH, SNMP, and APIs. SDN separates the control plane from the data plane — centralizing it in a software controller — while leaving the data plane in physical devices.

SDN controllers use **northbound APIs** (REST) to communicate with applications above, and **southbound APIs** (OpenFlow, NETCONF, RESTCONF, YANG) to program devices below. Cisco DNA Center is the enterprise implementation — it provides design, policy, provisioning, and AI-powered assurance capabilities. SD-Access uses VXLAN (data plane), LISP (control plane), and ISE (policy) to create an identity-based campus network.

REST APIs use HTTP methods for all operations: **GET** reads, **POST** creates, **PUT** replaces, **PATCH** partially updates, **DELETE** removes. Status codes indicate outcomes: 2xx means success, 4xx means client error, 5xx means server error. **JSON** is used by REST APIs and is the most common format. **XML** is used by NETCONF. **YAML** is used by Ansible and other automation tools. NETCONF runs over SSH on port 830 and uses XML; RESTCONF runs over HTTPS and supports both JSON and XML.

**Ansible** is the leading network automation tool — agentless (no software needed on devices), push-based, uses YAML playbooks, and idempotent by design. **Puppet** and **Chef** are agent-based (pull model) and better suited to server infrastructure than network devices.

### Exam Tips

> 🎯 **Most Tested Topics:**

1. **"What SDN interface connects the controller to network applications above it?"** → **Northbound API** (REST API, gRPC)
2. **"What SDN interface connects the controller to physical network devices below it?"** → **Southbound API** (OpenFlow, NETCONF, RESTCONF, YANG)
3. **"What does the data plane do?"** → **Forwards packets** at wire speed — IP forwarding, MAC switching, ACL enforcement, NAT, QoS
4. **"What does the control plane do?"** → **Builds routing and forwarding tables** — OSPF, BGP, STP, ARP
5. **"What does the management plane do?"** → Provides the **interface for configuration and monitoring** — SSH, SNMP, NETCONF, syslog
6. **"Which HTTP method retrieves data without making changes?"** → **GET**
7. **"Which HTTP method creates a new resource?"** → **POST**
8. **"What HTTP status code means the request succeeded?"** → **200 OK** (or 201 for resource creation)
9. **"What HTTP status code means authentication is required?"** → **401 Unauthorized**
10. **"What HTTP status code means the resource was not found?"** → **404 Not Found**
11. **"Which data format does NETCONF use?"** → **XML** (over SSH, port 830)
12. **"Which data format does RESTCONF use?"** → **JSON or XML** (over HTTPS)
13. **"What is YANG?"** → A **data modeling language** that defines the structure of configuration and operational data for NETCONF/RESTCONF
14. **"Which automation tool is agentless?"** → **Ansible** — uses SSH/API directly, no agent software needed on devices
15. **"What does idempotent mean in automation?"** → Running the same operation multiple times produces the **same result** — if the desired state already exists, nothing changes
16. **"What is the Cisco SD-WAN management plane component?"** → **vManage** (GUI and REST API)
17. **"What protocol does Cisco SD-WAN use for the control plane?"** → **OMP** (Overlay Management Protocol)
18. **"Which automation tool uses a pull model with agents?"** → **Puppet** and **Chef** (Ansible uses push, agentless)

---

✅ **Module 11 — All 11 Sections Complete.**

*← Previous: [10 — IP Services](./10_IP_Services.md)*  
*Next: [12 — Wireless Networking →](./12_Wireless.md)*
