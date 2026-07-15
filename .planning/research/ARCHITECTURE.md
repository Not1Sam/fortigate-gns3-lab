# Architecture Research

**Domain:** FortiGate SD-WAN / GNS3 Lab Topology
**Researched:** 2026-07-15
**Confidence:** HIGH

## Standard Architecture

### System Overview

```
+======================================================================+
|                        GNS3 HYPERVISOR (KVM)                         |
|  Host: Local workstation, GNS3 VM manages QEMU/Docker appliance     |
+======================================================================+
|                                                                       |
|  +------------+       +------------+       +----------------------+   |
|  | FortiGate  |       | FortiGate  |       | OVS Open vSwitch     |   |
|  | FGT-Primary|<------| FGT-Secondary|     | (L2 Fabric)          |   |
|  | (VDOM:     |  IPsec| (standalone|      | br0: 802.1Q trunk    |   |
|  |  Edge +    |  site |  or backup|      | VLAN 10,20,30        |   |
|  |  Internal) |  VPN  |  unit)    |      | tagged on trunk ports |   |
|  +-----+------+       +----+------+      +----------+-----------+   |
|        |                    |                        |               |
|        |  WAN1     WAN2     |                        |               |
|  +-----+------+    +-------+--------+                |               |
|  | GNS3 NAT   |    | Physical NIC   |                |               |
|  | Cloud      |    | Bridge (TBD)   |                |               |
|  | (virtual   |    | or 2nd NAT     |                |               |
|  | internet)  |    | Cloud          |                |               |
|  +-----+------+    +-------+--------+                |               |
|        |                    |                        |               |
+========|====================|========================|===============+
         |                    |                        |
         |     (IPsec tunnels over internet)           |
         |                    |                        |
+========|====================|========================|===============+
         |                    |                        |
  +------v--------------------v------------------------v------+
  |                    OCI VPS (Cloud)                      |
  |  +---------------+ +---------------+ +----------------+  |
  |  | Docker        | | IPsec VPN     | | Performance    |  |
  |  | digitaraJobs  | | Responder     | | SLA Probe      |  |
  |  | Mock API      | | (strongSwan   | | Target         |  |
  |  |               | |  or FGT VM)   | | (ping/HTTP)    |  |
  |  +---------------+ +---------------+ +----------------+  |
  |                                                          |
  |  Security List: UDP 500, UDP 4500, custom API ports     |
  +----------------------------------------------------------+
```

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| FGT-Primary | Primary SD-WAN edge, VDOM hosting, NGFW (IPS, App Ctrl, Web Filter), SSL inspection, NAT, dual IPsec tunnels to OCI | FortiGate VM 7.4.12 (KVM QCOW2) in GNS3, evaluation license, 10 virtual NICs |
| FGT-Secondary | Secondary/backup FortiGate for cross-device testing, route asymmetry validation, or HA pair | Same image, separate VM, second eval license |
| OVS (Open vSwitch) | Layer-2 fabric: 802.1Q VLAN trunking between FortiGates and downstream resources (VLAN 10/20/30) | GNS3 OVS appliance (Docker container), br0 as central bridge, per-port VLAN tagging |
| GNS3 NAT Cloud | WAN1: Provides virtual internet access via host NAT (VMnet8 / 192.168.122.0/24), DHCP, outbound connectivity for FortiGate evaluation license activation | GNS3 built-in "Cloud" node with NAT |
| Physical NIC Bridge | WAN2: Direct physical NIC passthrough for second WAN interface (optional, host-dependent) | GNS3 Cloud node bridged to host physical interface (requires virtio or e1000) |
| OCI VPS | Cloud termination: IPsec tunnel responder, Docker API host, SLA probe target, log receiver | OCI compute instance (Ubuntu), strongSwan IPsec or second FortiGate VM, Docker daemon with exposed API ports |
| VPCS/Client VMs | Downstream test endpoints in each VLAN; used for connectivity validation, HTTP/DNS testing through NGFW policies | GNS3 VPCS nodes (lightweight) or QEMU-based desktop VMs |

## GNS3 Project Layout

### Topology Structure

```
GNS3 Project: fortigate-sdwan-lab
├── Routers & Firewalls
│   ├── FGT-Primary (FortiGate 7.4.12, 10 NICs)
│   │   ├── port1  → WAN1 (GNS3 NAT Cloud)
│   │   ├── port2  → OVS eth1 (trunk: VLAN 10,20,30)
│   │   ├── port3  → WAN2 (Physical NIC Bridge / 2nd NAT)
│   │   ├── port4  → OVS eth2 (optional 2nd trunk or management)
│   │   └── port5-10 → reserved (future expansion)
│   │
│   └── FGT-Secondary (FortiGate 7.4.12, 10 NICs)
│       ├── port1  → WAN1 (same NAT Cloud or separate)
│       ├── port2  → OVS eth3 (trunk: VLAN 10,20,30)
│       └── port3+ → reserved
│
├── Switches
│   └── OVS-Core (Open vSwitch appliance, 16 NICs)
│       ├── eth0  → Management (separate mgmt network or NAT)
│       ├── eth1  → FGT-Primary port2  (trunk port, tagged VLANs 10,20,30)
│       ├── eth2  → FGT-Primary port4  (optional 2nd trunk)
│       ├── eth3  → FGT-Secondary port2 (trunk port, tagged VLANs 10,20,30)
│       ├── eth4  → VLAN 10 access port (LAN clients)
│       ├── eth5  → VLAN 20 access port (AD/LDAP server)
│       ├── eth6  → VLAN 30 access port (local Docker)
│       └── eth7-15 → reserved
│
├── End Devices
│   ├── PC-LAN (VPCS) → OVS eth4  (VLAN 10 — 192.168.10.0/24)
│   ├── SRV-AD  (Linux/Win) → OVS eth5  (VLAN 20 — 192.168.20.0/24)
│   └── SRV-Docker (Linux) → OVS eth6  (VLAN 30 — 192.168.30.0/24)
│
├── Cloud Nodes
│   ├── WAN1-Cloud (NAT — 192.168.122.0/24 host NAT)
│   └── WAN2-Cloud (Physical NIC Bridge — host-dependent subnet)
│
└── Links
    ├── All OVS-FortiGate links → 802.1Q trunk (tagged VLANs on both sides)
    └── All OVS-endpoint links → access ports (untagged, single VLAN)
```

### Structure Rationale

- **Nodes organized by role** (firewalls / switches / endpoints / clouds) because GNS3 workspace is flat — grouping by role minimizes visual clutter and makes cable tracing predictable.
- **OVS as central fabric** rather than direct FortiGate-to-endpoint links because the lab needs trunked multi-VLAN connectivity between both FortiGates and downstream resources. OVS handles VLAN tagging/untagging at the port level, eliminating the need for an external managed switch.
- **10 NICs on FortiGate** because the KVM template supports it; unused ports stay disconnected but reserving them avoids having to shut down the VM to add interfaces later.

## FortiGate-OVS VLAN Trunk Topology

### Physical-to-Logical Mapping

```
FGT-Primary port2  ─── OVS eth1
  │                        │
  ├── port2.10 (VLAN 10)   ├── tagged trunk (VLANs 10,20,30)
  ├── port2.20 (VLAN 20)   └── OVS config:
  └── port2.30 (VLAN 30)       ovs-vsctl set port eth1 trunks=10,20,30
                               ovs-vsctl set port eth4 tag=10     (access)
                               ovs-vsctl set port eth5 tag=20     (access)
                               ovs-vsctl set port eth6 tag=30     (access)
```

### VLAN Scheme

| VLAN ID | Name | Subnet | Gateway (FGT-Primary) | Purpose |
|---------|------|--------|----------------------|---------|
| 10 | LAN | 192.168.10.0/24 | 192.168.10.254 | General client access / web browsing test |
| 20 | AD/LDAP | 192.168.20.0/24 | 192.168.20.254 | Active Directory / LDAP server simulation |
| 30 | Docker | 192.168.30.0/24 | 192.168.30.254 | Local Docker services (testing, not API) |

### OVS Configuration Pattern

```bash
# On OVS appliance container (Debian + openvswitch-switch)
ovs-vsctl del-br br0                    # reset if needed
ovs-vsctl add-br br0

# Configure trunk ports (FortiGate connections)
ovs-vsctl add-port br0 eth1
ovs-vsctl set port eth1 trunks=10,20,30

ovs-vsctl add-port br0 eth3
ovs-vsctl set port eth3 trunks=10,20,30

# Configure access ports (endpoint connections)
ovs-vsctl add-port br0 eth4
ovs-vsctl set port eth4 tag=10         # VLAN 10 access

ovs-vsctl add-port br0 eth5
ovs-vsctl set port eth5 tag=20         # VLAN 20 access

ovs-vsctl add-port br0 eth6
ovs-vsctl set port eth6 tag=30         # VLAN 30 access

# Verify
ovs-vsctl show
```

### FortiGate VLAN Subinterface Configuration

```fortigate
config system interface
    edit "port2"
        set allowaccess ping
    next
    edit "port2.10"
        set vdom "Internal-VDOM"
        set interface "port2"
        set vlanid 10
        set ip 192.168.10.254 255.255.255.0
        set allowaccess ping https http ssh
    next
    edit "port2.20"
        set vdom "Internal-VDOM"
        set interface "port2"
        set vlanid 20
        set ip 192.168.20.254 255.255.255.0
        set allowaccess ping
    next
    edit "port2.30"
        set vdom "Internal-VDOM"
        set interface "port2"
        set vlanid 30
        set ip 192.168.30.254 255.255.255.0
        set allowaccess ping
    next
end
```

## Dual WAN Design

### WAN Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    FGT-Primary Edge-VDOM                   │
│                                                            │
│  wan1 (port1) ───── GNS3 NAT Cloud ──── Host Internet      │
│    │                     │                                  │
│    │  IP: 192.168.122.x  │  DHCP from host                 │
│    │  (DHCP)             │  Used for: eval license,        │
│    │                     │  general internet, primary WAN  │
│    │                     │                                  │
│  wan2 (port3) ───── Physical NIC Bridge ── Host LAN / ISP  │
│    │                     │                                  │
│    │  IP: <host LAN      │  Direct physical NIC passthrough│
│    │       subnet>/DHCP  │  Used for: secondary WAN,       │
│    │                     │  IPsec tunnel diversity          │
│    │                     │                                  │
│    │  FALLBACK:          │  If no physical bridge available │
│    │  2nd NAT Cloud      │  on different host subnet       │
└────────────────────────────────────────────────────────────┘
```

### WAN Interface Comparison

| WAN | GNS3 Cloud Type | IP Assignment | Pros | Cons |
|-----|----------------|---------------|------|------|
| WAN1 (port1) | NAT cloud | DHCP from host (192.168.122.0/24) | Works immediately, no host config needed, outbound internet | Double NAT, limited port control |
| WAN2 (port3) | Physical NIC bridge | DHCP from host LAN/router | Single NAT, real public IP if bridged, true multi-homing | Requires host NIC config, may lose host connectivity if misconfigured |
| WAN2 (fallback) | 2nd NAT cloud on VMnet | Different subnet via host | Works without NIC passthrough | Both WANs behind same host NAT — no true diversity |

**Recommendation**: Start with dual NAT clouds (WAN1 + WAN2 on separate GNS3 NAT networks via separate VMware VMnet adapters). This gives you WAN diversity without host NIC complications. Switch to physical NIC bridge for WAN2 only if you need real public IP termination or asymmetric routing testing.

## VDOM Segmentation

### VDOM Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      FGT-Primary (Multi-VDOM Mode)              │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  root VDOM (Management VDOM)                             │   │
│  │  Responsibilities:                                       │   │
│  │  • Global admin access                                   │   │
│  │  • FortiGuard updates (via wan1)                         │   │
│  │  • System-wide settings (DNS, NTP, HA)                   │   │
│  │  • Logging & reporting configuration                     │   │
│  │  Interfaces: wan1, wan2 (shared or dedicated mgmt port)  │   │
│  └──────────────┬───────────────────────────────────────────┘   │
│                 │ inter-vdom-link (vdom-link-root-to-edge)      │
│  ┌──────────────▼───────────────────────────────────────────┐   │
│  │  Edge-VDOM (Traffic VDOM — WAN Edge)                    │   │
│  │  Responsibilities:                                       │   │
│  │  • Default route outbound (pointing to WAN interfaces)   │   │
│  │  • Dual IPsec tunnels to OCI                             │   │
│  │  • SD-WAN zone (grouping both IPsec tunnels)             │   │
│  │  • Performance SLA probes                                │   │
│  │  • NAT (source NAT for LAN→Internet)                     │   │
│  │  Interfaces: wan1 (vdom-link), wan2, IPSec-OCI-1,        │   │
│  │              IPSec-OCI-2, SD-WAN zone                    │   │
│  └──────────────┬───────────────────────────────────────────┘   │
│                 │ inter-vdom-link (vdom-link-edge-to-int)       │
│  ┌──────────────▼───────────────────────────────────────────┐   │
│  │  Internal-VDOM (Traffic VDOM — LAN & Inspection)         │   │
│  │  Responsibilities:                                       │   │
│  │  • VLAN subinterfaces (port2.10, port2.20, port2.30)     │   │
│  │  • Inter-VLAN routing                                    │   │
│  │  • NGFW security policies (IPS, App Ctrl, Web Filter)    │   │
│  │  • SSL/TLS Deep Inspection                               │   │
│  │  • Traffic to Edge-VDOM via inter-VDOM link              │   │
│  │  Interfaces: port2.10, port2.20, port2.30,               │   │
│  │              vdom-link-int-to-edge                        │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### VDOM Configuration Pattern

```fortigate
# Enable multi-VDOM mode (requires reboot)
config system global
    set vdom-mode multi-vdom
end

# Create VDOMs
config vdom
    edit "Edge-VDOM"
        set type traffic
    next
    edit "Internal-VDOM"
        set type traffic
    next
end

# Create inter-VDOM links
config system vdom-link
    edit "root-edge-link"
        set vdom "root"
    next
    edit "edge-int-link"
        set vdom "Edge-VDOM"
    next
end

# Assign interfaces (done from config-vdom context)
# root VDOM: wan1, wan2
# Edge-VDOM: edge-int-link (one end), IPsec tunnel interfaces
# Internal-VDOM: port2.10, port2.20, port2.30, edge-int-link (other end)
```

### VDOM Rationale

- **root → Edge-VDOM separation**: root handles only global/system functions; Edge-VDOM owns all WAN-facing risk (public IPs, VPN termination, SD-WAN). Limits blast radius if Edge-VDOM is compromised.
- **Edge-VDOM → Internal-VDOM separation**: Edge-VDOM terminates tunnels and applies SD-WAN steering; Internal-VDOM applies NGFW inspection and LAN routing. Allows independent policy domains and clean inter-VDOM firewall rules.
- **Inter-VDOM links as the choke point**: All traffic between internal VLANs and the WAN/IPsec tunnels must cross the inter-VDOM link, making it a single point for security policy enforcement.

## Traffic Flows

### Flow 1: Management Access

```
Admin Laptop ──HTTPS──→ root VDOM (port1 or mgmt IP)
                            │
                            ├── CLI/GUI configuration
                            ├── FortiGuard updates (via wan1)
                            └── Logging to OCI (via IPsec tunnel or direct internet)
```

### Flow 2: LAN User → Internet (via SD-WAN)

```
PC-LAN (VLAN 10, 192.168.10.100)
    │
    ▼
OVS eth4 (access VLAN 10) ──802.1Q tagged──→ OVS eth1 (trunk)
    │
    ▼
FGT-Primary port2 → Internal-VDOM (port2.10)
    │
    ├── NGFW Inspection: SSL Interception → App Control → Web Filter → IPS
    │
    ▼
Inter-VDOM Link → Edge-VDOM
    │
    ├── SD-WAN Rule match (e.g., "general-internet")
    │   └── select wan1 or wan2 based on SLA
    │
    ▼
wan1 (NAT Cloud) → Host Internet
  or
wan2 (NIC Bridge) → ISP
```

### Flow 3: LAN User → OCI API (via IPsec SD-WAN)

```
PC-LAN (VLAN 10, 192.168.10.100)
    │
    ▼
Internal-VDOM → NGFW Inspection → Inter-VDOM Link
    │
    ▼
Edge-VDOM → SD-WAN Rule match ("cloud-traffic")
    │
    ├── SD-WAN zone contains IPSec-OCI-1 and IPSec-OCI-2
    ├── Performance SLA probe evaluates both tunnels
    └── Best-path selection based on latency/jitter
    │
    ▼
IPsec Tunnel (via wan1 or wan2) → Internet → OCI VPS
    │
    ▼
OCI Security List (UDP 500/4500 allowed)
    → Docker digitaraJobs API (port 8080)
```

### Flow 4: IPsec Tunnel Establishment

```
FGT-Primary Edge-VDOM
    │
    ├── IKEv2 Phase 1: UDP 500
    │   ├── wan1 IP → OCI Public IP (Tunnel 1)
    │   └── wan2 IP → OCI Public IP (Tunnel 2)
    │
    ├── IKEv2 Phase 2: ESP encrypted
    │   ├── Tunnel interface IPs: 169.254.1.0/30, 169.254.2.0/30
    │   └── Traffic selectors: 192.168.10.0/23 ↔ 10.x.x.x/32 (OCI)
    │
    └── Static route or BGP over tunnel
        ├── OCI subnet via tunnel1/tunnel2
        └── Metric/priority set for SD-WAN to decide
```

### Flow 5: Performance SLA Probe

```
Edge-VDOM SD-WAN Health-Check
    │
    ├── Probe target: OCI VPS IP (or 8.8.8.8 as fallback)
    ├── Protocol: ping (or HTTP to digitaraJobs endpoint)
    ├── Interval: 500ms
    ├── Measure: latency, jitter, packet loss
    │
    ├── Probe sent via wan1 → internet → OCI VPS
    └── Probe sent via wan2 → internet → OCI VPS
    │
    └── SLA thresholds:
        ├── Latency > 150ms → out-of-SLA
        ├── Jitter > 50ms   → out-of-SLA
        └── Packet loss > 5% → out-of-SLA
```

## IPsec Tunnel Topology

### Tunnel Routing

```
┌────────────────────────┐          ┌────────────────────────┐
│   FGT-Primary          │          │   OCI VPS              │
│   Edge-VDOM            │          │                        │
│                        │          │                        │
│  wan1 (192.168.122.x)──┤          │                        │
│    │                   │          │                        │
│    └── Tunnel 1 ───────┤──────────┤── Public IP            │
│        (IKEv2)         │  Internet │    (203.0.113.x)      │
│                        │          │                        │
│  wan2 (host subnet) ───┤          │                        │
│    │                   │          │                        │
│    └── Tunnel 2 ───────┤──────────┤                        │
│        (IKEv2)         │          │                        │
└────────────────────────┘          └────────────────────────┘

Tunnel Interface IPs:
  Tunnel1: 169.254.1.1/30 (FGT) ←→ 169.254.1.2/30 (OCI)
  Tunnel2: 169.254.2.1/30 (FGT) ←→ 169.254.2.2/30 (OCI)
```

### IPsec Configuration Pattern (FGT Side)

```fortigate
config vpn ipsec phase1-interface
    edit "OCI-TUNNEL1"
        set interface "wan1"
        set peertype any
        set net-device enable
        set proposal aes256-sha256
        set dhgrp 14
        set remote-gw <OCI_PUBLIC_IP>
        set psksecret <PSK1>
        set ike-version 2
    next
    edit "OCI-TUNNEL2"
        set interface "wan2"
        set peertype any
        set net-device enable
        set proposal aes256-sha256
        set dhgrp 14
        set remote-gw <OCI_PUBLIC_IP>
        set psksecret <PSK2>
        set ike-version 2
    next
end

config vpn ipsec phase2-interface
    edit "OCI-TUNNEL1"
        set phase1name "OCI-TUNNEL1"
        set proposal aes256-sha256
        set src-subnet 192.168.10.0 255.255.254.0
        set dst-subnet <OCI_VCN_CIDR> <OCI_VCN_MASK>
    next
    edit "OCI-TUNNEL2"
        set phase1name "OCI-TUNNEL2"
        set proposal aes256-sha256
        set src-subnet 192.168.10.0 255.255.254.0
        set dst-subnet <OCI_VCN_CIDR> <OCI_VCN_MASK>
    next
end

config system interface
    edit "OCI-TUNNEL1"
        set vdom "Edge-VDOM"
        set ip 169.254.1.1 255.255.255.252
        set remote-ip 169.254.1.2 255.255.255.252
    next
    edit "OCI-TUNNEL2"
        set vdom "Edge-VDOM"
        set ip 169.254.2.1 255.255.255.252
        set remote-ip 169.254.2.2 255.255.255.252
    next
end
```

## OCI VPS Role

### What Runs on OCI

| Service | Implementation | Port | Purpose |
|---------|---------------|------|---------|
| IPsec endpoint | strongSwan (Ubuntu) or lightweight FortiGate VM | UDP 500/4500 | Terminate both IPsec tunnels from GNS3 |
| Docker API | digitaraJobs mock (Node.js/Flask) | TCP 8080 | Simulated internal API for SD-WAN traffic steering |
| SLA probe target | ping/HTTP listener | ICMP / TCP 80 | Performance SLA health-check target for SD-WAN |
| Log receiver | syslog-ng or rsyslog | UDP 514 / TCP 6514 | Central syslog from FortiGate |
| NTP fallback | chronyd | UDP 123 | Time sync for lab (if needed) |

### OCI Security List Rules

| Direction | Source | Destination | Protocol | Port | Purpose |
|-----------|--------|------------|----------|------|---------|
| Ingress | 0.0.0.0/0 | OCI VPS | UDP | 500 | IKE Phase 1 |
| Ingress | 0.0.0.0/0 | OCI VPS | UDP | 4500 | NAT-T / IPsec |
| Ingress | 0.0.0.0/0 | OCI VPS | ESP (proto 50) | N/A | IPsec ESP |
| Ingress | <GNS3_NAT_SUBNET> | OCI VPS | TCP | 8080 | API access |
| Ingress | <GNS3_NAT_SUBNET> | OCI VPS | ICMP | N/A | SLA probes |
| Ingress | <GNS3_NAT_SUBNET> | OCI VPS | UDP | 514 | Syslog |

## SD-WAN Architecture

### SD-WAN Configuration

```
SD-WAN Zone: "cloud-vpn"
  Members:
    - OCI-TUNNEL1 (member 1, gateway: 169.254.1.2)
    - OCI-TUNNEL2 (member 2, gateway: 169.254.2.2)

Performance SLA: "oci-probe"
  Server: <OCI_VPS_PRIVATE_IP>  (or public IP if accessible)
  Participants: OCI-TUNNEL1, OCI-TUNNEL2
  Protocol: ping
  SLA Target:
    Latency threshold: 150ms
    Jitter threshold: 50ms
    Packet loss: 5%
  Update static route: enabled

SD-WAN Rules:
  Rule 1: "cloud-api" (priority 1)
    Source: 192.168.10.0/23
    Destination: <OCI_VCN_CIDR>
    Strategy: Best Quality (SLA)
    Members: OCI-TUNNEL1 (preferred), OCI-TUNNEL2 (backup)
    SLA: oci-probe

  Rule 2: "general-internet" (priority 2, implicit)
    Source: all
    Destination: all
    Strategy: Lowest Cost (SLA)
    Members: wan1 (primary), wan2 (secondary)
    SLA: oci-probe (or separate internet-probe)
```

### How Performance SLA Probes Work in the Lab

1. **Every 500ms**, Edge-VDOM sends a ping from each tunnel interface to the SLA probe server (OCI VPS or 8.8.8.8).
2. **Each tunnel's probe** follows a different path:
   - Tunnel1 probe → wan1 → NAT Cloud → internet → OCI
   - Tunnel2 probe → wan2 → NIC Bridge → ISP → OCI
3. **Latency, jitter, and packet loss** are calculated from the last 30-100 probes.
4. **If a tunnel misses SLA** (e.g., latency > 150ms), SD-WAN marks it out-of-SLA and stops steering traffic to it.
5. **When the tunnel recovers** (5 consecutive successful probes), SD-WAN restores it as an active member.
6. **"Update static route"** ensures that even non-SD-WAN traffic (e.g., management) follows the health check result — a tunnel whose static route is withdrawn becomes unreachable for ALL traffic, not just SD-WAN-rule traffic.

## Architectural Patterns

### Pattern 1: Hub-and-Spoke with Dual Path Diversity

**What:** Single on-premises hub (FGT-Primary) connecting to a cloud spoke (OCI VPS) over two independent IPsec tunnels, each over a different WAN transport. The hub terminates both tunnels into an SD-WAN zone.

**When to use:** Any time you need WAN diversity without BGP or dynamic routing complexity. The two tunnels provide path redundancy and load distribution.

**Trade-offs:**
- + Simple: static IPsec + SD-WAN rules, no BGP needed
- + Works with NAT on both sides (NAT-T handles it)
- - Both tunnels go to the same OCI public IP — no true provider diversity on the cloud side
- - SD-WAN traffic steering requires careful SLA tuning to avoid flapping

### Pattern 2: VDOM Segmentation for Edge/Internal Separation

**What:** Single physical FortiGate split into three logical firewalls: root (management), Edge-VDOM (WAN/SD-WAN), Internal-VDOM (LAN/inspection). Traffic crosses inter-VDOM links with explicit firewall policies.

**When to use:** When you want to practice enterprise FortiGate segmentation on a single device. Real-world MSSPs and multi-tenant deployments use this pattern.

**Trade-offs:**
- + Clean separation of concerns: WAN-edge policies don't mix with LAN-inspection policies
- + Inter-VDOM links are the only crossing point — easy to audit
- - Adds complexity: three routing tables to manage, policies on both sides of each link
- - Slight throughput penalty for inter-VDOM traffic (software switching vs hardware)

### Pattern 3: OVS as L2 Fabric

**What:** Open vSwitch provides 802.1Q VLAN trunking, replacing a physical managed switch. The FortiGate creates VLAN subinterfaces, and OVS handles per-port tagging/trunking.

**When to use:** When the GNS3 host doesn't have a separate managed switch appliance, or you want programmatic VLAN control from the OVS CLI.

**Trade-offs:**
- + Free, lightweight (Docker container in GNS3 VM)
- + Full CLI control (ovs-vsctl, ovs-ofctl) — no GUI dependency
- - No web UI — all configuration is CLI
- - No STP/RSTP by default (not an issue in a single-switch tree topology like this lab)
- - Cannot match feature set of a production managed switch (no L3 routing on this appliance by default)

## Data Flow

### Request Flow (LAN Client → OCI API over IPsec SD-WAN)

```
PC-LAN (192.168.10.100) → HTTP GET digitaraJobs API
    │
    ▼
OVS eth4 (access VLAN 10) → 802.1Q tagged → OVS eth1 (trunk)
    │
    ▼
FGT-Primary port2 → Internal-VDOM (port2.10)
    │
    ├── 1. Inter-VLAN routing check (dst not in 192.168.10.0/24)
    │
    ▼
    2. Firewall Policy lookup (src: VLAN10, dst: OCI, action: accept)
    │
    ▼
    3. SSL/TLS Deep Inspection (decrypt if HTTPS)
    │
    ▼
    4. App Control match ("digitaraJobs" or "web-api" signature)
    │
    ▼
    5. Web Filter check (category allow/block)
    │
    ▼
    6. IPS inspection (signature match, exploit prevention)
    │
    ▼
Inter-VDOM Link → Edge-VDOM
    │
    ├── 7. SD-WAN Rule match
    │   ├── Rule: "cloud-api" (src: 192.168.10.0/23, dst: OCI_VCN)
    │   ├── Strategy: Best Quality
    │   └── Check Performance SLA "oci-probe"
    │       ├── Tunnel1: in-SLA ✅ → steer traffic
    │       └── Tunnel2: in-SLA ✅ (standby)
    │
    ▼
    8. NAT (source NAT to IPsec tunnel IP or WAN IP)
    │
    ▼
OCI-TUNNEL1 (IPsec encrypted) → wan1 → internet → OCI
    │
    ▼
    9. OCI Security List allows UDP 500/4500 → IPsec decrypt
    │
    ▼
    10. Docker digitaraJobs API on port 8080
    │
    ▼
    11. Response follows reverse path through the same tunnel
```

### State Management

**FortiGate session table entries** (per-VDOM):
- Session state tracked per flow (not per packet for established sessions)
- Asymmetric routing not supported unless specifically configured (this lab uses symmetric paths)
- SD-WAN maintains per-tunnel SLA state (in-SLA/out-of-SLA flag, updated every probe interval)

## Scaling Considerations

| Scale Dimension | Architecture Adjustment |
|-----------------|------------------------|
| More VLANs | Add subinterfaces on FortiGate, add trunk VLANs to OVS trunk port configuration, add access ports on OVS. No topology change needed — pure config. |
| More branches (multi-site) | Add spoke FortiGates in separate GNS3 projects, each with their own IPsec tunnels to the OCI hub. ADVPN for direct spoke-to-spoke tunnels. |
| Higher throughput | In production: move to hardware FortiGate with NPU offload. In lab: not applicable — GNS3 is CPU-bound and throughput is not the goal. |
| High availability | Add FGT-Secondary into an HA cluster (active-passive). Requires HA heartbeat link and sync interface. VDOM config replicates. |

### Scaling Priorities

1. **First bottleneck: Host CPU** — GNS3 with two FortiGate VMs + OVS + OCI connection will consume significant CPU. If the lab becomes sluggish, reduce to one FortiGate VM.
2. **Second bottleneck: Evaluation license expiration** — 14-day license forces periodic rebuilds. Automate the rebuild process with config backup/restore scripts.

## Anti-Patterns

### Anti-Pattern 1: Direct Endpoint-to-FortiGate Links Without OVS

**What people do:** Connect each endpoint VM directly to a separate FortiGate port instead of trunking through OVS.

**Why it's wrong:** Wastes FortiGate interfaces (you get 10 total, but each VM endpoint consumes one). Makes VLAN expansion impossible without re-cabling. Loses the ability to have multiple devices on the same VLAN.

**Do this instead:** Trunk VLANs from OVS to FortiGate over a single (or LAG) link. Assign endpoints to OVS access ports.

### Anti-Pattern 2: Putting Management and WAN on the Same VDOM

**What people do:** Configure everything in root VDOM — WAN interfaces, LAN interfaces, IPsec tunnels, SD-WAN — all in one flat routing domain.

**Why it's wrong:** Defeats the purpose of VDOM segmentation. A misconfigured routing change on the WAN side can take down LAN management access. No isolation between security zones.

**Do this instead:** root VDOM for mgmt only, Edge-VDOM for WAN/SD-WAN, Internal-VDOM for LAN/inspection. Each gets its own routing table and firewall policies.

### Anti-Pattern 3: Single IPsec Tunnel for SD-WAN

**What people do:** Create one IPsec tunnel, add it as the only SD-WAN member, expect SD-WAN to work.

**Why it's wrong:** SD-WAN requires at least two members to make path selection decisions. A single tunnel gives you zero path diversity — the "best quality" strategy has nothing to compare.

**Do this instead:** Build two tunnels over distinct WAN transports, add both to the SD-WAN zone, define an SLA probe, and let SD-WAN choose between them.

### Anti-Pattern 4: OVS Default Bridge (br0) Without VLAN Trunk Configuration

**What people do:** Connect FortiGate and endpoints to OVS without setting any VLAN tags or trunks. Everything ends up in the default bridge with all ports on VLAN 0 (native).

**Why it's wrong:** All traffic is flat — there is no VLAN segmentation. Endpoints in "VLAN 10" and "VLAN 20" can communicate directly, bypassing the FortiGate. The entire point of the OVS is defeated.

**Do this instead:** Configure trunk ports (with `trunks=10,20,30`) for FortiGate-facing OVS ports, and access ports (with `tag=10`) for endpoint-facing ports. Verify with `ovs-vsctl show`.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| OCI VPS | IPsec Site-to-Site VPN (IKEv2, AES256-SHA256) | OCI side: CPE object + DRG + IPsec connection with two tunnels; On-prem: FortiGate phase1/phase2 with matching crypto profiles |
| digitaraJobs API | Docker container exposed on TCP 8080 | Reachable via IPsec tunnel from GNS3 LAN; Used for SD-WAN rule matching and SLA probing |
| FortiGuard | Outbound HTTPS from FortiGate root VDOM via wan1 | Required for evaluation license activation and UTM signature updates |
| Syslog | UDP 514 from FortiGate to OCI VPS | Centralized logging; configure on FortiGate under log settings |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| root VDOM ↔ Edge-VDOM | Inter-VDOM link (vdom-link-root-edge) | Management traffic only; root has reachability to Edge-VDOM tunnel endpoints for monitoring |
| Edge-VDOM ↔ Internal-VDOM | Inter-VDOM link (vdom-link-edge-int) | All user traffic crosses here; firewall policies on both sides required |
| FGT-Primary ↔ OVS | 802.1Q trunk (port2 → OVS eth1) | Single physical link carrying all VLAN traffic; no LACP in initial build |
| FGT-Primary ↔ FGT-Secondary | OVS trunk (port2 → OVS eth1, FGT-Sec port2 → OVS eth3) | Both firewalls share the same OVS fabric; future HA heartbeat would need dedicated link |

## Build Order Implications

The architecture reveals the following dependency chain, which directly informs phase ordering:

```
Phase 1: GNS3 Project & Base Connectivity
  ├── Create GNS3 project, import FortiGate images
  ├── Deploy OVS appliance, configure br0 and port trunking
  ├── Cable: OVS ↔ FortiGates, OVS ↔ endpoints
  └── Verify L2: VLANs pass between FortiGate and OVS

Phase 2: FortiGate Initial Configuration
  ├── Configure port IPs (wan1 DHCP, port2 trunk, VLAN subinterfaces)
  ├── Activate evaluation licenses (requires wan1 internet)
  ├── Configure static routes on root VDOM
  └── Verify: ping from endpoints to FortiGate VLAN gateways

Phase 3: OCI VPS Setup
  ├── Provision OCI compute instance
  ├── Configure Security Lists (UDP 500/4500, ESP, API ports)
  ├── Deploy Docker and digitaraJobs container
  └── Verify: OCI VPS reachable from GNS3 WAN (ping from FortiGate)

Phase 4: IPsec VPN to OCI
  ├── Configure phase1/phase2 on FGT-Primary (two tunnels)
  ├── Configure IPsec responder on OCI (strongSwan)
  ├── Assign tunnel interface IPs
  └── Verify: both tunnels up, ping across tunnel interfaces

Phase 5: VDOM Segmentation & Inter-VDOM Routing  ← DEPENDS ON Phase 2
  ├── Enable multi-VDOM mode (requires reboot)
  ├── Create Edge-VDOM and Internal-VDOM
  ├── Assign interfaces to VDOMs
  ├── Create inter-VDOM links
  └── Configure routing and firewall policies between VDOMs

Phase 6: SD-WAN Configuration  ← DEPENDS ON Phase 4
  ├── Create SD-WAN zone with both IPsec tunnel members
  ├── Configure Performance SLA probe
  ├── Create SD-WAN rules
  └── Verify: traffic steered across tunnels based on SLA

Phase 7: NGFW Security Profiles  ← DEPENDS ON Phase 6
  ├── SSL/TLS Deep Inspection (CA generation, certificate push)
  ├── Application Control signatures
  ├── Web Filtering policies
  ├── IPS profiles
  └── Verify: security profiles applied to SD-WAN policy

Phase 8: Advanced & Exploration
  ├── FGT-Secondary integration (cross-device testing)
  ├── HA cluster configuration (if desired)
  └── Free-form scenario exploration beyond course material
```

## Sources

- Fortinet Official Documentation: FortiOS 7.4 Administration Guide — SD-WAN, VDOM, IPsec VPN
- Fortinet Official Documentation: FortiOS 7.6 SD-WAN Architecture for Enterprise Guide
- Andrew Travis SD-WAN Lab Setup (2022/2023) — Reference topology designs with WANem and Hub/Spoke patterns
- KiwiTut GNS3 FortiGate Lab Guides — Practical LAG/VLAN topology patterns for FortiGate in GNS3
- OCI Site-to-Site VPN Best Practices whitepaper — CPE configuration and tunnel redundancy patterns
- Fortinet Documentation: VDOM Overview and Inter-VDOM Routing examples
- OVS in GNS3 labs (University of Naples) — OVS appliance configuration patterns for VLAN trunking

---
*Architecture research for: FortiGate SD-WAN / GNS3 Lab*
*Researched: 2026-07-15*
