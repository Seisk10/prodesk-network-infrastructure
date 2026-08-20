# Troubleshooting — Incident Log

This document brings together every real problem faced during the development of the ProDesk project, in chronological order, covering both phases: the network's original design (*Computing Environments and Connectivity* coursework) and the security hardening carried out later.

Each incident follows the structure: **symptom → diagnosis → fix**, prioritizing the reasoning behind the root cause, not just the final command.

---

## Phase 1 — Original network design

### 1.1 OSPF adjacency — confusion between Router ID and link address

**Symptom:** analyzing `show ip ospf neighbor`, the neighbor's address showed as `10.0.0.2` (the interface address on the `/30` link), while the displayed Router ID was something like `192.168.40.1` — raising doubt about whether this indicated a configuration error.

**Diagnosis:** the Router ID and the link address **don't need to match**. The Router ID is a logical identifier for the OSPF process (by default, the highest IP configured on the router at the moment the OSPF process comes up, unless set manually), while the reported neighbor address is the IP of the interface used to form the adjacency.

**Outcome:** there was no error — just a need to understand the conceptual difference between the two fields.

---

### 1.2 Sub-interfaces stuck at `protocol down`

**Symptom:** router sub-interfaces (`GigabitEthernet0/1.X`) showed `Status: up`, but `Protocol: down`.

**Diagnosis:** investigation into compatibility between the router's configuration (`encapsulation dot1Q`) and the corresponding switch port. In a router-on-a-stick topology, the router's physical interface and the connected switch trunk port need compatible configuration — checking encapsulation, allowed VLANs, physical port state, and the switch configuration on the other end.

**Fix:** encapsulation and trunk configuration adjusted until the sub-interface came up with `Protocol: up`.

---

### 1.3 IP address conflict on VLAN 99

**Symptom:** during infrastructure reorganization, an address on VLAN 99 was found already in use by another device.

**Diagnosis:** violation of the fundamental rule that every L3 interface on the same network must have a unique address. The conflict involved the `192.168.99.0/26` range (gateway `.1`, server `.10`).

**Fix:** reassigning a unique address to each device, eliminating the overlap.

---

### 1.4 Architectural question — should distribution switches know about the other building's VLANs?

**Symptom:** during planning, a question arose about whether each building's distribution switches should be aware of the remote building's VLANs.

**Diagnosis/decision:** a switch doesn't need to indiscriminately carry every VLAN in the company. Each building locally transports only its own VLANs; inter-building communication happens via the routers, which handle routing between the different networks:

```text
Local VLAN → Router → Inter-building link → Router → Other building's VLAN
```

**Outcome:** there's no need to extend a single Layer 2 domain across both buildings — a decision that reinforced the routed architecture between sites.

---

### 1.5 Attempt at Layer 3 switches at the distribution layer (discarded approach)

**Symptom:** when testing moving inter-VLAN routing to the distribution switches (via `ip routing`, SVIs, and VLAN interfaces), multiple problems surfaced: interfaces stuck at `protocol down`, addressing conflicts, and unstable communication with the routers and OSPF.

**Diagnosis:** the added complexity (multiple routing points, SVIs, redistribution) did not stabilize in the Packet Tracer scenario within the available time.

**Fix/decision:** reverted to the architecture with pure L2 switches at distribution and centralized routing on the routers (router-on-a-stick), prioritizing a simple, predictable solution over a technically more sophisticated one that was unstable in the implementation environment.

---

### 1.6 Native VLAN Mismatch (original occurrence, design phase)

**Symptom:** log messages such as:
```
%CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered...
```
identifying different native VLANs on the two ends of a link (e.g., one port with native VLAN 1, the other with native VLAN 10).

**Diagnosis:** incompatible native VLAN configuration between the two ends of a trunk port — generally stemming from configuration made during the Layer 3 testing period (section 1.5).

**Fix:** manual alignment of the native VLAN between the ends of the link.

> **Note:** this is a different incident from the native VLAN mismatch that occurred during hardening (section 2.3) — same error class, but distinct contexts and causes, separated in time by the complete architecture rework between the two phases.

---

### 1.7 STP/PVID — BPDU received on a non-trunk port

**Symptom:** on the Logistics distribution switch, messages appeared:
```
%SPANTREE-2-RECV_PVID_ERR
%SPANTREE-2-BLOCK_PVID_LOCAL
```
indicating an 802.1Q BPDU received on a port not configured as trunk.

**Diagnosis:** one end of the link wasn't configured as trunk, but was receiving tagged BPDUs — a role mismatch between the two ends of the link.

**Fix:** aligning port mode (trunk/access) on both ends of the link, clearing the STP protective block.

---

## Phase 2 — Security hardening

### 2.1 `crypto key generate rsa` rejected — default hostname

**Symptom:**
```
% Please define a hostname other than Router.
```
when attempting to generate an RSA key pair to enable SSH, followed by subsequent commands (`admin`, `1024`) being interpreted as invalid configuration commands, since the prompt had returned to `(config)#` mode without the key being generated.

**Diagnosis:** IOS requires a customized hostname (different from the default `Router`) before generating RSA keys, since the device name factors into the key's domain calculation.

**Fix:**
```
hostname R-Producao
ip domain-name prodesk.local
crypto key generate rsa
```
followed by the module size (`1024`) when prompted interactively.

---

### 2.2 `switchport port-security` rejected on a dynamic port

**Symptom:**
```
Command rejected: FastEthernet0/2 is a dynamic port.
```

**Diagnosis:** the `switchport port-security` command requires the port to be in a fixed mode (`access` or `trunk`) — it doesn't work on ports in the default dynamic mode (`dynamic desirable/auto`). The subsequent commands (`maximum`, `violation`, `mac-address sticky`) were accepted without visible error, but had no real effect, since the root command had failed.

**Fix:** fixing the port mode before applying port security:
```
switchport mode access
switchport port-security
switchport port-security maximum 1
switchport port-security violation restrict
switchport port-security mac-address sticky
```

---

### 2.3 Native VLAN Mismatch and persistent STP block (hardening phase)

**Symptom:** after applying `switchport trunk native vlan 999` to all trunk ports on the distribution switches, `show interfaces trunk` kept showing some ports missing VLAN 999 in the *forwarding state* table, even with identical configuration on both ends. In a more severe case, STP actively blocked VLAN 999 traffic on a specific port:
```
%SPANTREE-2-RECV_PVID_ERR: Received BPDU with inconsistent peer vlan id 1 on FastEthernet0/2 VLAN999.
%SPANTREE-2-BLOCK_PVID_LOCAL: Blocking FastEthernet0/2 on VLAN0999. Inconsistent local vlan.
```

**Diagnosis:** comparing port-by-port configuration via `show running-config`, it was found that only the first trunk port on each distribution switch (`Fa0/1`) had explicit `switchport mode trunk`, inherited from the project's original configuration. The remaining trunk ports were in **dynamic** mode (negotiated via DTP) — which made Spanning Tree treat the native VLAN change more cautiously, keeping a residual blocked state even after the remote port had already been fixed.

**Fix:**
1. Explicitly fixing trunk mode on all ports: `switchport mode trunk`.
2. On ports that remained blocked even after the mode fix, it was necessary to force STP reconvergence with a manual interface bounce:
```
interface range FastEthernet0/X-Y
 shutdown
 no shutdown
```
The protocol did not automatically recalculate state from an already-established block — the bounce was needed to restart the convergence process.

---

### 2.4 DHCP Snooping blocking legitimate requests (`NONZERO_GIADDR`)

**Symptom:** right after enabling `ip dhcp snooping`, hosts stopped being able to renew their IP (`ipconfig /renew` returning "DHCP request failed").

**Diagnosis:** the distribution switch log revealed the exact cause:
```
%DHCP_SNOOPING-5-DHCP_SNOOPING_NONZERO_GIADDR: DHCP_SNOOPING drop message 
with non-zero giaddr or option82 value on untrusted port
```
By default, IOS DHCP Snooping inserts option 82 (relay agent information) into packets and rejects, on ports not fully trusted for this purpose, any packet that already arrives with a non-zero `giaddr` (gateway IP address) — exactly what happens in this scenario, since the **router** sets this field when relaying via `ip helper-address`. The switch was interpreting the router's legitimate relay as potentially malicious.

**Fix:** disabling option 82 insertion on all switches with snooping enabled, keeping only the uplink ports as *trusted* — sufficient for this scenario's threat model (relay performed by a trusted Layer 3 device):
```
no ip dhcp snooping information option
```

---

## What these incidents, together, illustrate

A network can have every piece of equipment physically connected and still not work correctly — each layer needs to be validated individually:

```text
1. Physical → 2. VLAN → 3. Trunk → 4. IP → 5. Routing → 6. Services → 7. Security
```

The Phase 1 (design) and Phase 2 (hardening) incidents have different natures — the first mainly deals with basic connectivity and architectural choices; the second, with security controls that, if misapplied, can break legitimate services (as seen in the DHCP Snooping case). In both, the diagnostic pattern was the same: isolate the affected layer, compare configuration port-by-port or device-by-device, and use IOS log messages (CDP, STP, DHCP Snooping) as the primary diagnostic source instead of trial and error.
