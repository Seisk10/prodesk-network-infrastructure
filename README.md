[🇧🇷 Português](README.md) | 🇬🇧 English

# ProDesk — Corporate Network Infrastructure

Network infrastructure project for the fictional company **ProDesk**, built in Cisco Packet Tracer as part of the *Computing Environments and Connectivity* course (UNISUL, Brazil). Later revisited and reinforced with a full security hardening layer, focused on mitigating common Layer 2 and Layer 3 network attacks.

## Table of contents

- [Project context](#project-context)
- [Topology](#topology)
- [IP addressing](#ip-addressing)
- [Routing](#routing)
- [Network services](#network-services)
- [Security hardening](#security-hardening)
- [Real troubleshooting](#real-troubleshooting)
- [How to test](#how-to-test)
- [Limitations and design decisions](#limitations-and-design-decisions)

---

## Project context

ProDesk received investor funding to build its data network across two physical sites, 600 meters apart, sharing the same internal communication infrastructure:

- **Production unit** — R&D, Quality, and Purchasing departments (120 hosts projected)
- **Logistics unit** — Administrative and Development departments (75 hosts projected)

The original brief called for 195 total hosts. Following the assignment's explicit premise — *"not all listed hosts need to be present in the project, but they must be accounted for when sizing switches and other network equipment"* — the topology was implemented with one representative host per access switch, while addressing and broadcast domains were sized to support each department's real headcount.

## Topology

Three-layer hierarchical architecture (core / distribution / access), replicated at each physical site:

```
                    [Core - Production] ── fiber optic (Gi0/3/0) ── [Core - Logistics]
                       Router 2911                                    Router 2911
                           │                                              │
                  [Distribution Switch]                          [Distribution Switch]
                    2960-24TT + Server                              2960-24TT
                           │                                              │
              ┌────────────┼────────────┐                   ┌────────────┴────────────┐
          [Purchasing]  [Quality]    [R&D]                [Administrative]    [Development]
          2 access      2 access    2 access                2 access             2 access
          switches      switches    switches                switches             switches
```

**Core layer** — two Cisco 2911 routers, one per site, connected via fiber optic (HWIC-1GE-SFP module + GLC-SH-SMD transceiver on interface `GigabitEthernet0/3/0`), running OSPF.

**Distribution layer** — one 2960-24TT switch per site, aggregating traffic from access switches via 802.1Q trunks. The Production distribution switch also hosts the centralized network server (DHCP/DNS).

**Access layer** — 10 2960-24TT switches total (2 per department, 5 departments), each representing one host from the corresponding department. Port `Fa0/1` is always trunk (uplink to distribution), port `Fa0/2` is always access (host).

## IP addressing

Class C addressing, `/26` subnets (255.255.255.192) per VLAN/department, allowing up to 62 usable hosts per subnet — enough to accommodate each department's real headcount (maximum 40 hosts projected per area).

| VLAN | Department | Site | Network | Gateway | Host range |
|---|---|---|---|---|---|
| 10 | Purchasing | Production | 192.168.0.0/26 | 192.168.0.1 | .2 – .62 |
| 20 | Quality | Production | 192.168.10.0/26 | 192.168.10.1 | .2 – .62 |
| 30 | R&D | Production | 192.168.20.0/26 | 192.168.20.1 | .2 – .62 |
| 40 | Administrative | Logistics | 192.168.30.0/26 | 192.168.30.1 | .2 – .62 |
| 50 | Development | Logistics | 192.168.40.0/26 | 192.168.40.1 | .2 – .62 |
| 99 | Management / Server | Production | 192.168.99.0/26 | 192.168.99.1 | .2 – .62 |
| 999 | Native (unused) | Both | — | — | trunk isolation |
| — | Router↔Router link | Both | 10.0.0.0/30 | — | .1 / .2 |

> VLAN 999 has no hosts assigned — it exists exclusively as the native VLAN on trunk links, a recommended practice to mitigate VLAN hopping attacks (double tagging).

## Routing

**OSPF (area 0)** runs on both routers, advertising all local subnets and the inter-site link network. Inter-VLAN routing within each site is handled via **router-on-a-stick**: a single physical interface (`GigabitEthernet0/1`) subdivided into sub-interfaces with `dot1Q` encapsulation, one per VLAN.

Example (Production Router):
```
interface GigabitEthernet0/1.10
 encapsulation dot1Q 10
 ip address 192.168.0.1 255.255.255.192
 ip helper-address 192.168.99.10
```

The `ip helper-address` on each sub-interface redirects DHCP broadcasts to the centralized server, allowing hosts on any VLAN — including in the Logistics site, on the other side of the OSPF link — to obtain network configuration automatically.

## Network services

Single server (`192.168.99.10`), physically connected to the Production distribution layer, centralizing:

- **DHCP** — one pool per VLAN, with dedicated gateway and address range, serving both sites via `ip helper-address`
- **DNS** — A record configured (`intranet.empresa.local` → `192.168.99.10`), supporting internal name resolution

## Security hardening

The original topology (functional but with no security controls) was reinforced with six layers of protection, each mitigating a specific class of attack relevant to enterprise local networks:

| # | Control | Where it was applied | Attack mitigated |
|---|---|---|---|
| 1 | Console, VTY, and enable secret passwords (with `service password-encryption`) | All devices | Unauthorized access to configuration via console or network |
| 2 | SSH (RSA 1024-bit) + local user, `login local` | Both routers | Credential interception in plaintext (Telnet) |
| 3 | Isolated native VLAN (VLAN 999) on all trunk links | 2 distribution switches + 10 access switches | VLAN hopping via double tagging |
| 4 | Port Security (`maximum 1`, `sticky`, `violation restrict`) | Host port (Fa0/2) on each access switch | MAC flooding / CAM table overflow, and unauthorized device connection |
| 5 | Shutdown of all unused ports | All access switches (Fa0/3–24, Gi0/1–2) | Unauthorized physical access via open port |
| 6 | DHCP Snooping (uplink ports as *trusted*, option 82 disabled) | 2 distribution switches + 10 access switches | Rogue DHCP server / DHCP starvation |

### Technical detail — DHCP Snooping in this scenario

Since inter-VLAN routing and DHCP relay are handled by the **router** (via `ip helper-address`), not the switch, the default insertion of **option 82** in DHCP Snooping had to be disabled (`no ip dhcp snooping information option`). Without this adjustment, the switch dropped legitimate DHCP packets coming from the router's relay, treating the non-zero `giaddr` field on an untrusted port as untrustworthy. This behavior — and the diagnostic process to the root cause — is detailed in the following section.

## Real troubleshooting

Two real issues surfaced during the hardening implementation, documented below — not just the fix, but the reasoning that led to it.

### 1. Native VLAN mismatch stuck in blocked state (Spanning Tree)

**Symptom:** after applying `switchport trunk native vlan 999` to all trunk ports on the distribution switches, `show interfaces trunk` kept showing some ports missing VLAN 999 in the *forwarding state* table, even with identical configuration on both ends.

**Diagnosis:** comparing port-by-port configuration (`show running-config`), it was found that only the first trunk port on each switch (`Fa0/1`) had an explicit `switchport mode trunk` command — inherited from the project's original configuration. The remaining ports were in **dynamic** trunk mode (negotiated via DTP), which made Spanning Tree treat the native VLAN change more cautiously, keeping a residual blocked state even after the fix was applied.

**Fix:**
1. Explicitly fixing trunk mode on all ports: `switchport mode trunk`
2. On ports that remained blocked even after the fix, it was necessary to force STP reconvergence with a manual interface bounce (`shutdown` / `no shutdown`), since the protocol did not automatically recalculate state from an already-established block.

### 2. DHCP Snooping blocking legitimate requests (non-zero giaddr)

**Symptom:** right after enabling `ip dhcp snooping` on the switches, hosts stopped being able to renew their IP (`ipconfig /renew` returning "DHCP request failed").

**Diagnosis:** the distribution switch log revealed the cause:
```
%DHCP_SNOOPING-5-DHCP_SNOOPING_NONZERO_GIADDR: DHCP_SNOOPING drop message 
with non-zero giaddr or option82 value on untrusted port
```
By default, IOS DHCP Snooping inserts option 82 (relay agent information) into packets and rejects, on ports not fully trusted for this purpose, any packet that already arrives with a non-zero `giaddr` (gateway IP address) — exactly what happens here, since the **router** sets this field when relaying via `ip helper-address`. The switch was interpreting the router's legitimate relay as potentially malicious.

**Fix:** disabling option 82 insertion on all switches with snooping enabled — `no ip dhcp snooping information option` — keeping uplink ports as *trusted*, which is sufficient for this scenario's threat model (relay performed by a trusted Layer 3 device).

## How to test

1. Open the latest `.pkt` file in Cisco Packet Tracer (version 8.x recommended)
2. On any PC, run `ipconfig /release` followed by `ipconfig /renew` — should obtain a correct IP, gateway, and DNS for the corresponding VLAN
3. Test connectivity between different departments on the same site (e.g., Purchasing → R&D) — validates inter-VLAN routing
4. Test connectivity between sites (e.g., Purchasing, in Production → Administrative, in Logistics) — validates OSPF and the fiber link between R1 and R2
5. On a distribution switch, run `show ip dhcp snooping binding` — should list hosts with active leases
6. Try SSH to either router: `ssh -l admin <ip>` — should authenticate with the configured local credential

## Limitations and design decisions

- **Single, non-redundant server:** since both sites belong to the same company and are close to each other (600m), DHCP/DNS was centralized on a single server in Production, with relay via `ip helper-address` to Logistics — a deliberate choice favoring administrative simplicity over high availability, acceptable given the network's size.
- **One host per access switch:** the topology represents the full logical structure (VLANs, subnets, switches sized per department), but does not populate all hosts projected in the assignment (195 total) — per the assignment's explicit premise, which waives the need for all hosts to be physically present, requiring only that the design account for them.
- **Plaintext passwords in `enable secret`/documentation:** the credentials used (`Cisco123!`) are for lab/demonstration purposes only and should not be reused in a real environment.
- **eJPT / offensive security study:** the controls applied (port security, DHCP snooping, native VLAN isolation) were deliberately chosen because they correspond to Layer 2 attack vectors commonly tested in offensive certifications (MAC flooding, VLAN hopping, rogue DHCP), reinforcing this project's direct relevance to offensive security studies.

## Extended documentation

This README covers the project's final state. For anyone wanting to dig into the design process and technical incidents throughout development (including approaches tested and discarded), see:

- **[docs/en/DEVELOPMENT-HISTORY.md](docs/en/DEVELOPMENT-HISTORY.md)** — full record of the original planning phase: why each architectural decision was made (VLANs, `/26`, OSPF, centralized DHCP), including the attempt to move routing to Layer 3 switches at the distribution layer, tested and reverted in favor of the current L2 + router-on-a-stick architecture.
- **[docs/en/TROUBLESHOOTING.md](docs/en/TROUBLESHOOTING.md)** — every real incident faced in the project, across both development phases (original design and later security hardening): symptom, diagnosis, and fix.

---

**Tools:** Cisco Packet Tracer · OSPF · VLAN/802.1Q · DHCP/DNS · Port Security · DHCP Snooping
