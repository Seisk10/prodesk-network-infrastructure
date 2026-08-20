# Development History — ProDesk Corporate Network Project

> **Context note:** this document was written during the original development of the project (*Computing Environments and Connectivity* course, UNISUL), before the security hardening phase documented in the [main README](../../README.en.md). It records the design reasoning, architectural decisions, and troubleshooting process of the original academic version of the project. Some implementation details described here (e.g., attempts with Layer 3 switches at the distribution layer) were tested and discarded — the final architecture is the one documented in the README. Kept in full for its value as a record of the engineering process.

---

## 📌 Overview

This project consists of planning and implementing a network infrastructure for a company distributed across **two buildings**, named:

* **Production**
* **Logistics**

The goal was to develop a functional, organized, and scalable network, applying corporate networking concepts such as:

* Hierarchical network architecture;
* VLANs for logical segmentation;
* Layer 2 switches;
* Inter-VLAN routing;
* Dynamic routing with OSPF;
* Centralized DHCP;
* `ip helper-address`;
* IPv4 addressing with `/26` subnets;
* Point-to-point `/30` link;
* Fiber optic interconnection between buildings;
* SFP modules/interfaces;
* Structured cabling;
* Connectivity testing and service validation.

The project was built and tested in **Cisco Packet Tracer**, going through several implementation and troubleshooting stages before reaching the functional architecture presented in the final version.

---

## 1. Project objective

The goal was to build a network infrastructure capable of:

* connecting the company's different departments;
* logically separating departments;
* allowing communication between departments when needed;
* allowing communication between the two buildings;
* automatically providing IP addressing;
* centralizing network services;
* using dynamic routing;
* physically organizing the equipment;
* allowing future infrastructure expansion;
* demonstrating, through Packet Tracer, the operation of the main services.

The network needed to support **195 end-user devices** distributed across departments.

---

## 2. Overall architecture

The topology was structured following a three-layer hierarchical architecture:

```text
                 CORE
          ┌───────────────┐
          │ R1 ─────── R2 │
          └───────┬───────┘
                  │
        ┌─────────┴─────────┐
        │                   │
 DISTRIBUTION          DISTRIBUTION
  Building 1              Building 2
        │                   │
   ┌────┼────┐         ┌────┼────┐
   │    │    │         │    │    │
 ACCESS ACCESS ACCESS  ACCESS ACCESS
   │    │    │         │    │    │
 HOSTS HOSTS HOSTS     HOSTS HOSTS
```

### Access layer

Responsible for directly connecting end devices: computers, department hosts, and other end-user devices.

Each department has its own access switches.

### Distribution layer

Responsible for aggregating connections from each building's access switches.

In the final architecture used in the project, distribution switches remained **Layer 2**.

This kept routing centralized on the routers, simplifying the architecture and maintaining a clear division of responsibilities:

```text
L2 Switches → VLAN transport
        ↓
Router → routing between networks/VLANs
        ↓
OSPF → routing between buildings
```

### Core layer

Represented by the routers:

* **R1 — Production**
* **R2 — Logistics**

Both routers are responsible for interconnecting the buildings and routing between their respective networks.

---

## 3. Physical structure

### 3.1 Production building

The Production building has the following departments:

| Department | Devices | VLAN |
| --------- | -----------: | ---: |
| Purchasing |           40 |   10 |
| Quality |           40 |   20 |
| R&D       |           40 |   30 |

Total: **120 devices**

The physical structure uses two access switches per department.

The distribution switch sits in a technical room in the building and aggregates connections from the access switches.

Router **R1** connects to the distribution switch.

A dedicated services VLAN also exists:

* VLAN 99 — Services

This VLAN hosts the DHCP server.

---

## 4. Logistics building

The Logistics building has:

| Department | Devices | VLAN |
| --------------- | -----------: | ---: |
| Administrative |           35 |   40 |
| Development |           40 |   50 |

Total: **75 devices**

Combining both buildings:

```text
Production: 120
Logistics:   75
----------------
Total:      195 devices
```

The project therefore meets the specified device count.

The building has its own distribution switch and access switches.

Router **R2** interconnects the building with the rest of the network.

---

## 5. Physical topology vs. logical topology

Both concepts were used during development.

### Physical topology

Represents **where equipment is physically located**.

```text
Computer
    │
Access switch
    │
Distribution switch
    │
Router
    │
Fiber
    │
Router
```

Also represents distribution across floors and technical rooms.

### Logical topology

Represents **how devices and networks communicate logically**, independent of physical layout.

In this project, this mainly involves:

* VLANs;
* subnets;
* gateways;
* routing;
* OSPF;
* DHCP;
* inter-building communication.

The most appropriate term for the diagram showing VLANs, networks, and communication is therefore **logical topology**.

---

## 6. VLANs

Each department received its own VLAN.

| VLAN | Department | Network |
| ---: | --------------- | --------------- |
|   10 | Purchasing         | 192.168.0.0/26  |
|   20 | Quality       | 192.168.10.0/26 |
|   30 | R&D             | 192.168.20.0/26 |
|   40 | Administrative  | 192.168.30.0/26 |
|   50 | Development | 192.168.40.0/26 |
|   99 | Services        | 192.168.99.0/26 |

Using VLANs allows departments to be logically separated even while sharing the same physical switching infrastructure.

This provides:

* segmentation;
* organization;
* reduced broadcast domain;
* traffic control;
* easier administration;
* the ability to apply security policies later.

---

## 7. Why one VLAN per department?

VLAN separation was adopted to avoid keeping every device in the company on a single logical network.

With VLANs:

```text
Purchasing      → VLAN 10
Quality         → VLAN 20
R&D             → VLAN 30
Administrative  → VLAN 40
Development     → VLAN 50
Services        → VLAN 99
```

This makes the infrastructure more organized and makes traffic easier to control.

Important: a VLAN alone does not mean absolute security isolation. Inter-VLAN routing can still allow communication when configured. A VLAN mainly provides **logical segmentation**.

---

## 8. IPv4 addressing

The addressing plan mainly used `/26` networks.

```text
/26 = 255.255.255.192
64 total addresses
  1 → network address
 62 → host addresses
  1 → broadcast
```

---

## 9. Why use /26?

The project needed to support multiple devices per department. A `/30` network only has 4 total addresses (2 usable hosts) — unsuitable for a user network.

Usage was:

```text
/26 → department networks
/30 → point-to-point link between routers
```

---

## 10. Addressing ranges

**VLAN 10 — Purchasing:** network `192.168.0.0/26`, gateway `192.168.0.1`, hosts `.1–.62`, broadcast `.63`

**VLAN 20 — Quality:** network `192.168.10.0/26`, gateway `192.168.10.1`, hosts `.1–.62`, broadcast `.63`

**VLAN 30 — R&D:** network `192.168.20.0/26`, gateway `192.168.20.1`, hosts `.1–.62`, broadcast `.63`

**VLAN 40 — Administrative:** network `192.168.30.0/26`, gateway `192.168.30.1`, hosts `.1–.62`, broadcast `.63`

**VLAN 50 — Development:** network `192.168.40.0/26`, gateway `192.168.40.1`, hosts `.1–.62`, broadcast `.63`

**VLAN 99 — Services:** network `192.168.99.0/26`, gateway `192.168.99.1`, server `192.168.99.10`, broadcast `.63`

The server has a static address (`192.168.99.10/26`) and does not receive its own address via DHCP.

---

## 11. Link between R1 and R2

```text
Network:   10.0.0.0/30
Mask:      255.255.255.252
R1:        10.0.0.1
R2:        10.0.0.2
Broadcast: 10.0.0.3
```

This network has exactly two usable hosts — precisely what a point-to-point link needs.

---

## 12. Fiber optic interconnection

The routers were set up with a fiber-compatible module/interface, showing up as:

```text
GigabitEthernet0/3/0
```

```text
R1 GigabitEthernet0/3/0 → 10.0.0.1/30
R2 GigabitEthernet0/3/0 → 10.0.0.2/30
```

In the final implementation, this interface was enabled with an `HWIC-1GE-SFP` module and a `GLC-SH-SMD` transceiver, consistent with the case study specification (two units 600 meters apart). The `/30` was used because the link only needs two IP addresses, one per endpoint.

---

## 13. Structured cabling

The project considered structured cabling following TIA/EIA standards.

```text
TIA/EIA → infrastructure standard/norm (not a cable name)
Cat5e   → category of twisted-pair cable
```

---

## 14. DHCP

The DHCP server was placed on VLAN 99:

```text
IP:      192.168.99.10
Mask:    255.255.255.192
Gateway: 192.168.99.1
```

VLAN 99 was used as a services network, separating the server from user networks.

---

## 15. Why is the DHCP server on its own VLAN?

VLAN 99 was used as a dedicated services network, logically separating users from infrastructure. Besides DHCP, a services network could later host DNS, file servers, applications, monitoring, and other corporate resources.

---

## 16. Centralized DHCP

Instead of creating a DHCP server for each department or building, a **single centralized DHCP server** was used, with scopes matching the different networks:

```text
                 DHCP
             192.168.99.10
                   │
        ┌──────────┼──────────┬──────────┬──────────┐
        │          │          │          │          │
     VLAN 10    VLAN 20    VLAN 30    VLAN 40    VLAN 50
        │          │          │          │          │
     Hosts       Hosts       Hosts      Hosts      Hosts
```

This simplifies administration, maintenance, scope creation, address control, and infrastructure expansion.

---

## 17. DHCP scopes

A scope was created for each user network, delivering to clients: IP address, mask, default gateway, and DNS.

---

## 18. The problem with DHCP across VLANs

Initial DHCP requests use broadcast. A router normally **does not forward broadcasts from one network to another**:

```text
PC on VLAN 10 → DHCP broadcast → Gateway → ✗ → Server on VLAN 99
```

Without additional configuration, the request would never reach the server.

---

## 19. `ip helper-address`

The fix applied on the interfaces/sub-interfaces responsible for client networks:

```text
ip helper-address 192.168.99.10
```

Resulting behavior:

```text
PC → DHCP Discover → Gateway → ip helper-address → DHCP Server 192.168.99.10 → DHCP Offer → PC
```

This lets the centralized server serve clients from other VLANs and even the other building, as long as routing between networks is working.

---

## 20. Does the DHCP server support 195 devices?

Yes. A `/26` doesn't mean the server can only provide IPs to 62 devices company-wide — the 62 hosts are **per subnet/VLAN**.

```text
5 user VLANs × 62 hosts = 310 available addresses
```

Enough for the project's 195 devices, with room for growth.

---

## 21–24. OSPF

Dynamic routing protocol chosen: **OSPF**, configured in **Area 0** (backbone area), since the structure is small and doesn't require multiple areas.

Reasons for the choice: link-state based operation, best-path calculation, fast convergence, support for multiple areas, cost-based metric, better scalability than protocols like RIP (limited to 15 hops with hop-count metric).

---

## 25. R1 historical configuration

```text
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.0.1 255.255.255.192
 ip helper-address 192.168.99.10

interface GigabitEthernet0/1.20
 encapsulation dot1Q 20
 ip address 192.168.10.1 255.255.255.192
 ip helper-address 192.168.99.10

interface GigabitEthernet0/1.30
 encapsulation dot1Q 30
 ip address 192.168.20.1 255.255.255.192
 ip helper-address 192.168.99.10

interface GigabitEthernet0/1.99
 encapsulation dot1Q 99
 ip address 192.168.99.1 255.255.255.192

interface GigabitEthernet0/3/0
 ip address 10.0.0.1 255.255.255.252

router ospf 1
 log-adjacency-changes
 network 192.168.0.0 0.0.0.63 area 0
 network 192.168.10.0 0.0.0.63 area 0
 network 192.168.20.0 0.0.0.63 area 0
 network 10.0.0.0 0.0.0.3 area 0
 network 192.168.99.0 0.0.0.63 area 0
```

---

## 26. R2 historical configuration

```text
interface GigabitEthernet0/1.40
 encapsulation dot1Q 40
 ip address 192.168.30.1 255.255.255.192
 ip helper-address 192.168.99.10

interface GigabitEthernet0/1.50
 encapsulation dot1Q 50
 ip address 192.168.40.1 255.255.255.192
 ip helper-address 192.168.99.10

interface GigabitEthernet0/3/0
 ip address 10.0.0.2 255.255.255.252

router ospf 1
 log-adjacency-changes
 network 192.168.30.0 0.0.0.63 area 0
 network 192.168.40.0 0.0.0.63 area 0
 network 10.0.0.0 0.0.0.3 area 0
```

---

## 27. OSPF verification

Main commands used for troubleshooting:

```bash
show ip ospf neighbor   # checks adjacency (e.g., FULL/BDR state)
show ip route           # checks dynamically learned routes
```

---

## 28. Troubleshooting methodology used

A significant part of development was spent diagnosing connectivity issues — across VLANs, trunks, switches, routers, OSPF, DHCP, interfaces, addressing, and inter-building communication. The specific incidents, with symptom → diagnosis → fix, are documented in **[TROUBLESHOOTING.md](TROUBLESHOOTING.md)**.

The general approach adopted, layer by layer:

```text
1. Physical → 2. VLAN → 3. Trunk → 4. IP → 5. Routing → 6. Services
```

With the commands:

```bash
show ip interface brief
show running-config
show vlan brief
show interfaces trunk
show ip ospf neighbor
show ip route
ping <address>
```

---

## 29. Approaches tested and discarded

### Layer 3 switches at the distribution layer

At one point during development, an architecture using Layer 3 switches at the distribution layer was considered, with inter-VLAN routing handled directly on those switches (via `ip routing`, SVIs, VLAN interfaces).

This approach introduced additional complexity and operational problems in the Packet Tracer scenario — interfaces stuck in `protocol down`, addressing conflicts, and difficulty maintaining stable communication with the central router and OSPF.

**Why it was abandoned:** the final architecture was kept with Layer 2 switches at the distribution layer, as a design decision to preserve simplicity, predictability, compatibility with the topology already implemented, centralized routing on the routers, and a clear separation between switching and routing.

### Broad infrastructure rebuild

At one point, a full wipe of switches and routers for a from-scratch rebuild was considered. This approach showed how making simultaneous changes across multiple points makes diagnosis harder — the later strategy was to return to the known-working model and verify each component individually, one at a time.

**Principle adopted for the final version:**

> A simple, predictable, working architecture is preferable to a theoretically more sophisticated one that isn't functioning correctly in the implementation environment.

---

## 30. Key project decisions

| Decision | Reason |
|---|---|
| One VLAN per department | Logical segmentation and organization |
| `/26` for department networks | Up to 62 hosts per VLAN, meeting department needs with room for growth |
| `/30` between routers | Point-to-point link only needs two valid addresses |
| Centralized DHCP | Reduces complexity, simplifies administration |
| Dedicated services VLAN (99) | Separates infrastructure from users |
| `ip helper-address` | Needed to forward DHCP across different VLANs/networks |
| OSPF | Dynamic routing between buildings, no manual static routes |
| Single area (0) | Small structure, doesn't require multiple OSPF areas |
| L2 switches at distribution | Architectural simplicity, centralized routing on routers |
| Fiber between buildings | Represents the real link between units 600 meters apart |

---

## 31. Fundamental configuration commands

**Create VLAN:**
```bash
vlan 10
 name PURCHASING
```

**Access port:**
```bash
interface GigabitEthernet0/1
 switchport mode access
 switchport access vlan 10
```

**Trunk:**
```bash
interface GigabitEthernet0/1
 switchport mode trunk
```

**Router-on-a-stick (sub-interface):**
```bash
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.0.1 255.255.255.192
```

**OSPF — wildcard mask:**
```text
/26 → 0.0.0.63
/30 → 0.0.0.3
```

---

## 32. Network capacity — summary

```text
Each /26: 64 total addresses, 62 usable hosts

VLAN 10 → 62
VLAN 20 → 62
VLAN 30 → 62
VLAN 40 → 62
VLAN 50 → 62
----------------
Total   → 310 available addresses

Requirement: 195 devices
Margin: 115 addresses (future growth without re-planning addressing)
```

---

## 33. Full flow of a computer (new host joining the network)

```text
1. PC connects to the access switch
2. Port belongs to the department's VLAN
3. PC sends a DHCP request (broadcast)
4. Gateway (router sub-interface) receives the request
5. ip helper-address forwards it to the server
6. DHCP server identifies the correct scope
7. IP/mask/gateway/DNS are sent
8. PC starts using the network
```

If the computer needs to reach the other building:

```text
PC → Gateway → Local router → OSPF/routing table →
Link between R1 and R2 → Remote router → Remote VLAN → Destination
```

---

## 34. Technologies and concepts used

Cisco Packet Tracer · IPv4 · Subnetting · CIDR · VLAN · IEEE 802.1Q · Trunking · Layer 2 Switch · Router-on-a-stick · OSPF · OSPF Area 0 · DHCP · DHCP Relay · `ip helper-address` · DNS · Fiber optics · SFP · Structured cabling · TIA/EIA · Cat5e · STP · CDP · ICMP/Ping

---

## 35. Original implementation checklist

**Infrastructure:** both buildings, access switches, distribution switches, R1 and R2, link interfaces between routers, physical connection between buildings.

**VLANs:** creation of VLANs 10/20/30/40/50/99, access port assignment, compatible trunk configuration between switches.

**Addressing:** configuration of each VLAN, `10.0.0.1/30` on R1, `10.0.0.2/30` on R2, `192.168.99.10/26` on the server.

**Routing:** OSPF on R1 and R2, networks in area 0, adjacency and routing table verification.

**DHCP:** scopes for each VLAN, gateways, masks, DNS, `ip helper-address`, host testing.

**Validation:** `show vlan brief`, `show interfaces trunk`, `show ip interface brief`, `show ip ospf neighbor`, `show ip route`, ping gateway, ping R1↔R2, ping between VLANs, ping between buildings, DHCP validation per department.

---

## 36. Conclusion of the original design phase

The project demonstrated a complete corporate infrastructure implementation in a simulated environment, integrating VLAN, switching, routing, OSPF, DHCP relay, and end-to-end connectivity between two physical sites.

The troubleshooting process was a central part of development, allowing the identification and resolution of Layer 2, Layer 3, addressing, STP, VLAN, OSPF, and DHCP issues — a process documented in detail in [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

The architecture resulting from this phase was later reinforced with a full security hardening layer, documented in the [main README](../../README.en.md).
