# Technical Facts & Topology Specs

## Environment Baseline
- **Hypervisor**: GNS3 on Linux (QEMU/KVM via libvirt + Docker)
- **Firewall OS**: FortiOS 7.4.12 KVM (both FGTs, requires FortiCloud eval), FortiOS 7.0.9 pre-licensed also available
- **Host RAM**: 32 GB — ~7.3 GB budgeted for lab VMs
- **GNS3 Control**: `gns3-control` script at `~/.local/bin/gns3-control`
  - `start/stop/restart/status` — manages GNS3 server + docker + libvirtd
  - `forward-enable/disable` — toggles iptables MASQUERADE for NAT1 internet access
  - `force-stop` — kills orphan QEMU/ubridge/VPCS processes

## FortiGate Permanent Eval License — Exact Limits

### Hard Limits (enforced by license)
| Resource | Cap | Lab Allocation |
|---|---|---|
| Interfaces | **3** (port1, port2, port3) | port1=WAN, port2=LAN, port3=transit |
| Firewall Policies | **3** | LAN→WAN (SNAT), transit→WAN, IPsec/VPN |
| Routes | **3** | Default WAN route, LAN subnet, transit subnet |
| vCPU | **1** | Template configured to 1 |
| RAM | **2 GB** | Template configured to 2048 MB |
| VDOMs | **2** (root admin + 1 traffic) | Not used in current design |
| Encryption | **Low only** (data plane) | AES128-SHA1 for IPsec — functional, demo-able |

### What Is NOT Included (no workaround)
- ❌ **No FortiGuard updates** — AV signatures, IPS signatures, web filter categories, DNS filter ratings all require paid subscription
- ❌ **No FortiCare support**
- ❌ **Ports 4-8 on the VM are non-functional** — only port1, port2, port3 pass traffic

### UTM Workarounds (no FortiGuard needed)
| Feature | How It Still Works | Demo Impact |
|---|---|---|
| **Antivirus** | EICAR test file is hardcoded in the AV engine | Block page still displays ✅ |
| **IPS** | SQLi/XSS/port-scan signatures are factory-built in | Full detection ❌ |
| **Web Filter** | Use **static URL filter** instead of dynamic category lookup | Same block page effect ✅ |
| **DNS Filter** | Use **static domain block list** instead of FortiGuard rating | Same block effect ✅ |
| **App Control** | Factory app signatures work (browsers, common protocols) | Limited but functional ⚠️ |
| **SSL Inspection** | Deep inspection works without FortiGuard (self-signed CA) | Full functionality ✅ |

## Approach — Two Independent FGTs with OSPF

### Why No HA
With only 3 usable ports per FGT, HA requires port3 for heartbeat, leaving no port for transit routing between the two FGTs. Two independent FGTs with OSPF gives cross-LAN routing while keeping all 3 ports available.

### Interface Allocation (per FGT)
| Port | Connected To | Purpose |
|---|---|---|
| port1 | Switch1 | WAN (DHCP from NAT1) |
| port2 | OVS (LAN1 or LAN2) | LAN gateway |
| port3 | Other FGT (direct) | Transit link (OSPF) |

### Route Strategy (3 routes max per FGT)
| Route | Purpose |
|---|---|
| Default (0.0.0.0/0 via WAN) | Internet access |
| LAN subnet (direct) | Local clients |
| Transit subnet (direct) | Reach the other FGT |

## Network Addressing

| Segment | Subnet | Gateway | DHCP |
|---|---|---|---|
| WAN (NAT1) | `192.168.122.0/24` | `192.168.122.1` (virbr0) | virbr0/libvirt |
| LAN1 (FGT-Primary) | `192.168.10.0/24` | `192.168.10.1` (FGT port2) | FGT built-in DHCP |
| LAN2 (FGT-Secondary) | `192.168.20.0/24` | `192.168.20.1` (FGT port2) | FGT built-in DHCP |
| Transit | `10.0.0.0/30` | — | — |

## ESSENTIAL NODES — DO NOT REMOVE

These 12 nodes are the current topology. **Never delete these without explicit user approval.**

| Node | Type | Status |
|------|------|--------|
| FGT-Primary | qemu (fgt-v7.4.12.qcow2) | essential |
| FGT-Secondary | qemu (fgt-v7.4.12.qcow2) | essential |
| Switch1 | ethernet_switch | essential |
| NAT1 | nat | essential |
| Ubuntu | qemu | essential |
| PC1 | vpcs | essential |
| DHCP-Server | docker (alpine-dhcp) | essential |
| OpenvSwitch-1 | docker (gns3/openvswitch) | essential |
| OpenvSwitch-2 | docker (gns3/openvswitch) | essential |
| webterm-1 | docker (gns3/webterm) | essential |

### Current Docker Templates (GNS3 DB)
5 Docker templates in GNS3: **webterm**, **Alpine Linux**, **Open vSwitch**, **PostgreSQL-1**, **appServer-1**.

### Current Docker Images (local)
| Image | Size | Purpose |
|---|---|---|
| alpine-dhcp:latest | 13.8 MB | DHCP-Server |
| gns3/openvswitch:latest | 27.8 MB | OVS-1, OVS-2 |
| gns3/webterm:latest | 1.03 GB | webterm-1 |
| alpine:latest | 13 MB | Base for new containers |
| postgres:16-alpine | ~80 MB | PostgreSQL database |
| fortilab-appserver:latest | ~120 MB | Flask web app |

## Topology — 12 Nodes (2 FGT + 6 Docker + VPCS + Switch + NAT + Ubuntu)

### QEMU VMs (4)
| Node | RAM | vCPU | Adapters | Console | Image |
|---|---|---|---|---|---|
| FGT-Primary | 2 GB | 1 | 8 (3 usable) | Telnet | `fgt-v7.4.12.qcow2` |
| FGT-Secondary | 2 GB | 1 | 8 (3 usable) | Telnet | `fgt-v7.4.12.qcow2` |
| Ubuntu | 2 GB | 2 | 1 (e1000) | VNC | `ubuntu-24.04-minimal-cloudimg` (mod., pwd: `gns3`) |
| Switch1 | — | — | 3 | — | GNS3 ethernet switch |

### Docker Nodes (6 active)
| Node | Image | Adapters | Console | Purpose |
|---|---|---|---|---|
| DHCP-Server | `alpine-dhcp:latest` | 1 | Telnet | DHCP on LAN2 |
| OpenvSwitch-1 | `gns3/openvswitch:latest` | 16 | Telnet | L2 switching LAN1 |
| OpenvSwitch-2 | `gns3/openvswitch:latest` | 16 | Telnet | L2 switching LAN2 |
| webterm-1 | `gns3/webterm:latest` | 1 | VNC | Browser client (LAN1) |
| PostgreSQL-1 | `postgres:16-alpine` | 1 | Telnet | Database (LAN1) |
| appServer-1 | `fortilab-appserver:latest` | 1 | Telnet | Flask web app (LAN1) |

### Built-in (2)
| Node | Type | Purpose |
|---|---|---|
| PC1 | VPCS | CLI client (ping, traceroute) |
| NAT1 | GNS3 Cloud (virbr0) | WAN gateway |

### NOT YET ADDED (to be added one at a time)
| Node | Image | Purpose |
|---|---|---|
| Grafana-1 | `grafana/grafana:latest` | Dashboards |
| Prometheus-1 | `prom/prometheus:latest` | Metrics |
| Traffic-Gen-1 | `alpine:latest` | LAN1 traffic |
| Traffic-Gen-2 | `alpine:latest` | LAN2 traffic |

## Wiring (current)
```
NAT1:0 <-> Switch1:0
Switch1:1 <-> FGT-Primary:port1    (WAN)
Switch1:2 <-> FGT-Secondary:port1  (WAN)
FGT-Primary:port2   <-> OpenvSwitch-1:0  (LAN1)
FGT-Secondary:port2 <-> OpenvSwitch-2:0  (LAN2)
FGT-Primary:port3   <-> FGT-Secondary:port3  (transit OSPF)
OpenvSwitch-1:1 <-> PC1:0
OpenvSwitch-1:2 <-> webterm-1:0
OpenvSwitch-1:3 <-> PostgreSQL-1:0
OpenvSwitch-1:4 <-> appServer-1:0
OpenvSwitch-2:1 <-> DHCP-Server:0
OpenvSwitch-2:2 <-> Ubuntu:0
```

### OCI Instance (future)
| Node | Type | Purpose |
|---|---|---|
| OCI Instance | Real cloud VM | Threat simulator (nmap, EICAR, SQLi, phishing) |

## Port Maps (current — two independent FGTs)

| FGT | Port | Connected To | Address | Purpose |
|---|---|---|---|---|
| FGT-Primary | port1 (a0) | Switch1:1 | DHCP (`192.168.122.x`) | WAN |
| FGT-Primary | port2 (a1) | OpenvSwitch-1:0 | `192.168.10.1/24` | LAN1 |
| FGT-Primary | port3 (a2) | FGT-Secondary:port3 | `10.0.0.1/30` | Transit (OSPF) |
| FGT-Secondary | port1 (a0) | Switch1:2 | DHCP (`192.168.122.x`) | WAN |
| FGT-Secondary | port2 (a1) | OpenvSwitch-2:0 | `192.168.20.1/24` | LAN2 |
| FGT-Secondary | port3 (a2) | FGT-Primary:port3 | `10.0.0.2/30` | Transit (OSPF) |
| PostgreSQL-1 | eth0 | OpenvSwitch-1:3 | `192.168.10.11/24` (static) | Database |
| appServer-1 | eth0 | OpenvSwitch-1:4 | `192.168.10.103/24` (DHCP) | Flask web app |

> [!note] Approach — Two independent FGTs with OSPF
> - port1=WAN, port2=LAN, port3=transit. All 3 ports used per FGT.
> - No HA — each FGT is independent. OSPF on transit provides cross-LAN routing.
> - Both use `fortios.qcow2` (7.0.9 pre-licensed) — no FortiCloud registration needed.

## OCI Threat Simulator Endpoints
| Endpoint | Traffic | What It Proves |
|---|---|---|
| `GET /inspect` | HTTP | SNAT — shows source IP after NAT |
| `GET /api/status` | HTTP | API routing through FGT |
| `GET /eicar` | HTTP + HTTPS | AV block (EICAR hardcoded) |
| `GET /malware-sample` | HTTP + HTTPS | IPS block (factory signature) |
| `GET /attack?sql=payload` | HTTP | IPS — SQL injection |
| `GET /attack?xss=payload` | HTTP | IPS — XSS |
| `GET /phishing` | HTTPS | Web Filter — static URL block |
| `GET /hacking-tools` | HTTPS | Web Filter — static URL block |
| `GET /proxy` | HTTPS | Web Filter — static URL block |
| `POST /bruteforce` | HTTPS | Admin lockout |
| DNS `phish.test.lab` | DNS | DNS filter — static domain block |
| `nmap -sS FGT-WAN` | TCP SYN | IPS — port scan anomaly |

## Credentials

| Node | User | Password | Auth Method |
|---|---|---|---|
| FGT (both) | admin | (none — set at first boot) | Web UI / CLI |
| Ubuntu Desktop | ubuntu | gns3 | VNC login, SSH (NOPASSWD sudo) |
| Ubuntu root | root | gns3 | `su -` from ubuntu |
| VPCS | — | — | No auth required |
| Alpine Docker | root | (none — container default) | Telnet |
| webterm | — | — | VNC, no auth |

> [!tip] Ubuntu credentials are persistent
> The GNS3 base image (`ubuntu-24.04-minimal-cloudimg-amd64.img`) has been modified with the ubuntu user pre-created. Any new linked clone node inherits these credentials.

## Available Images
| Image | Version | Source | License | Location |
|---|---|---|---|---|
| `fortios.qcow2` | FortiOS 7.0.9 build 0444 (GA) | Pre-licensed | Valid eval, no registration needed | `~/GNS3/images/QEMU/` |
| `fgt-v7.4.12.qcow2` | FortiOS 7.4.12 build 2902 | Official Fortinet eval | Requires FortiCloud + free eval | `~/GNS3/images/QEMU/` |
| `fgt-v8.0.0.qcow2` | FortiOS 8.0.0.F build 0167 | Official Fortinet eval | Requires FortiCloud + free eval | `~/GNS3/images/QEMU/` |

> [!note] FGT images
> Both FGTs currently run `fgt-v7.4.12.qcow2` (requires FortiCloud eval account). The `fortios.qcow2` (7.0.9 pre-licensed) is also available in `~/GNS3/images/QEMU/` — no FortiCloud needed. Both images are OSPF-compatible and can be mixed in the same topology. See [[FortiGate-7.0.9-PreLicensed]] for setup guide.

## Key Constraints Summary
1. **7.0.9 pre-licensed** — no FortiCloud registration needed per VM
2. **2 FGTs** — no HA (port limitation), independent with OSPF
3. **3 policies each FGT** — forces efficient policy design
4. **3 routes each FGT** — default WAN + LAN + transit
5. **No FortiGuard** — UTM uses factory signatures + static lists
6. **Low encryption data plane** — IPsec uses AES128-SHA1, not AES256-GCM
