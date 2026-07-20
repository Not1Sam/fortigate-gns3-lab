# GNS3 FortiGate Security Lab — Full Topology Specification

## 1. Architecture Overview

A GNS3-based network security lab with 14 nodes demonstrating FortiGate routing, NAT, IPsec VPN, HA clustering, and full FortiGuard UTM stack. An OCI free-tier VM acts as an external threat simulation platform for live security demos.

### Design Principles

- **Two separate LAN segments** (OVS-LAN1 behind FGT1, OVS-LAN2 behind FGT2) enable DHCP relay testing and cross-FGT IPsec scenarios
- **All Docker nodes use Podman** (no sudo needed)
- **Zero extra licenses** — everything uses built-in FortiGate features
- **OCI instance is a real internet host** — attack traffic comes from outside the lab, proving the security stack works against genuine external threats

---

## 2. Network Addressing

| Segment | Subnet | Gateway | DHCP Server | DHCP Range |
|---|---|---|---|---|
| WAN (NAT1) | `192.168.122.0/24` | `192.168.122.1` (virbr0) | virbr0 (libvirt) | `192.168.122.2-254` |
| LAN1 (FGT1) | `192.168.10.0/24` | `192.168.10.1` (FGT1 port2) | FGT1 built-in DHCP | `192.168.10.100-200` |
| LAN2 (FGT2) | `192.168.20.0/24` | `192.168.20.1` (FGT2 port2) | Alpine dnsmasq | `192.168.20.100-200` |
| HA Link | `169.254.0.0/30` | — | Static | — |
| IPsec Tunnel | `10.0.0.0/8` | — | — | — |
| Management | `192.168.10.0/24` via FGT1 | — | — | — |

### VLAN Strategy (Future Expansion)

| VLAN ID | Purpose | Subnet | FGT Interface |
|---|---|---|---|
| 10 | LAN1 — Clients | `192.168.10.0/24` | FGT1 port2 |
| 20 | LAN2 — Clients | `192.168.20.0/24` | FGT2 port2 |
| — | Management | In-band via LAN1 | FGT1 port2 |

---

## 3. Node Specifications

### 3.1 QEMU Nodes

#### FortiGate-7.4.12 (x2)

| Property | Value |
|---|---|
| Template | FortiGate-7.4.12 |
| Platform | `x86_64` |
| RAM | 2048 MB |
| vCPU | 1 |
| Adapters | 8 × `virtio-net-pci` |
| Console | Telnet |
| Disk Image | `fgt-v7.4.12.qcow2` (108 MB) |
| Clone Mode | Linked clone |
| Boot Priority | `c` (disk) |
| Port Naming | `Ethernet{0}` |
| On Close | `power_off` |

**FGT-Primary Port Map**

| Port | Connected To | Address | Purpose |
|---|---|---|---|
| port1 | NAT1 | DHCP (`192.168.122.x`) | WAN — internet access |
| port2 | OVS-LAN1 | `192.168.10.1/24` | LAN1 — internal network |
| port3 | FGT-Secondary port3 | `169.254.0.1/30` | HA heartbeat |

**FGT-Secondary Port Map**

| Port | Connected To | Address | Purpose |
|---|---|---|---|
| port1 | NAT1 | DHCP (`192.168.122.x`) | WAN — internet access |
| port2 | OVS-LAN2 | `192.168.20.1/24` | LAN2 — internal network |
| port3 | FGT-Primary port3 | `169.254.0.2/30` | HA heartbeat |

#### Ubuntu Desktop Client

| Property | Value |
|---|---|
| Template | Ubuntu-Desktop-Client |
| Platform | `x86_64` |
| RAM | 2048 MB |
| vCPU | 2 |
| Adapters | 1 × `e1000` |
| Console | VNC |
| Disk Image | `ubuntu-24.04-minimal-cloudimg-amd64.img` (modified) |
| Boot Priority | `c` (disk) |
| Extra QEMU Options | `-cpu host` |
| Credentials | `ubuntu` / `gns3` |

---

### 3.2 Docker Nodes (Podman)

All Docker nodes run via GNS3's Podman integration. No additional configuration needed beyond pulling the specified image.

#### Open vSwitch (x2)

| Property | Value |
|---|---|
| Image | `gns3/openvswitch:latest` |
| Adapters | 16 |
| Console | Telnet |
| Memory | Not allocated (kernel network stack only) |

#### webterm

| Property | Value |
|---|---|
| Image | `gns3/webterm:latest` |
| Adapters | 1 |
| Console | VNC (HTTP browser on port 80) |
| Notes | Full browser environment — used for HTTP/HTTPS testing, web filter demos |

#### Alpine Linux — DHCP Server

| Property | Value |
|---|---|
| Image | `alpine:latest` (8.7 MB) |
| Adapters | 1 |
| Console | Telnet |
| First Boot | `apk add dnsmasq && dnsmasq -d` |
| DHCP Config | Range `192.168.20.100-200`, gateway `192.168.20.1`, DNS `208.67.222.222` (OpenDNS) |

#### App Server (Flask + Nginx)

| Property | Value |
|---|---|
| Image | `python:3.12-alpine` (~50 MB, to pull) |
| Adapters | 1 |
| Console | Telnet |
| Internal Ports | 80 (HTTP), 443 (HTTPS via Nginx proxy), 5000 (Flask) |
| Services | Nginx reverse proxy → Flask app |
| App Endpoints | `/` (landing page), `/api/status` (JSON health), `/inspect` (request inspector), `/login` (simulated auth) |
| Database Backend | PostgreSQL on `192.168.10.10:5432` |

#### PostgreSQL 16

| Property | Value |
|---|---|
| Image | `postgres:16-alpine` (297 MB, already cached) |
| Adapters | 1 |
| Console | Telnet |
| Internal Port | 5432 |
| Credentials | `app` / `app-pass` / `appdb` |
| Storage | Ephemeral (no persistence needed) |

#### Monitoring Stack (Grafana + Prometheus)

| Property | Value |
|---|---|
| Image(s) | `grafana/grafana:latest` + `prom/prometheus:latest` (to pull) |
| Adapters | 1 |
| Console | VNC (HTTP) for Grafana |
| Grafana Port | 3000 |
| Prometheus Port | 9090 |
| Data Sources | FGT SNMP metrics, syslog events |
| Dashboards | Live session count, throughput graphs, blocked attacks, VPN status |

#### Traffic Generator + Syslog Collector

| Property | Value |
|---|---|
| Image | `alpine:latest` (8.7 MB, already cached) |
| Adapters | 1 |
| Console | Telnet |
| Services | socat (syslog UDP 514), cron-based curl/ping/dig to App Server and OCI |

---

### 3.3 Built-in / Other Nodes

#### VPCS — PC1

| Property | Value |
|---|---|
| Type | Built-in VPCS |
| Console | Telnet |
| Purpose | Lightweight CLI — ping, traceroute, basic TCP tests |

#### NAT1 — GNS3 Cloud Node

| Property | Value |
|---|---|
| Type | GNS3 Cloud (NAT) |
| Host Interface | `virbr0` (libvirt/KVM bridge) |
| Subnet | `192.168.122.0/24` |
| Gateway | `192.168.122.1` |
| DHCP Pool | `192.168.122.2–254` |
| Internet Access | Host IP forwarding + iptables MASQUERADE on physical interface |

---

### 3.4 OCI Threat Simulation Platform

| Property | Value |
|---|---|
| Shape | VM.Standard.A1.Flex (ARM) or VM.Standard.E2.1.Micro (AMD free-tier) |
| OS | Ubuntu 24.04 LTS |
| Public IP | Ephemeral or reserved |
| Security List | Ingress: TCP 80/443, UDP 500/4500 (IPsec), ICMP |
| Software | Python (Flask/Gunicorn), nmap, socat, Unbound DNS, OpenSSL |

**Threat Simulator Endpoints**

| Endpoint | Traffic | Demo |
|---|---|---|
| `GET /inspect` | HTTP | Request inspection — shows source IP after SNAT |
| `GET /api/status` | HTTP | JSON API response — proves API routing works |
| `GET /eicar` | HTTP + HTTPS | EICAR test file — Antivirus blocks at FGT |
| `GET /malware-sample` | HTTP + HTTPS | Known IPS test pattern — IPS blocks at FGT |
| `GET /attack?sql=payload` | HTTP | SQL injection in URL — IPS signature blocks |
| `GET /attack?xss=payload` | HTTP | XSS in URL — IPS signature blocks |
| `GET /phishing` | HTTPS | Fake login page — Web Filter blocks by category |
| `GET /hacking-tools` | HTTPS | Hacking tool page — Web Filter blocks by category |
| `GET /proxy` | HTTPS | Web proxy site — Web Filter blocks by category |
| `POST /bruteforce` | HTTPS | Login brute force — admin lockout triggers |
| DNS `phish.test.lab` | DNS | Malicious domain — DNS Filter blocks |

---

## 4. Connection Map

```
[OCI Threat Simulator]
    │  (internet)
[PUBLIC INTERNET]
    │
[NAT1 — virbr0 — 192.168.122.1/24]
    │
    ├──── WAN (port1) ──── FGT-Primary ──── HA (port3) ──── FGT-Secondary ──── WAN (port1) ────┘
    │                         │ (port2)                      │ (port2)
    │                     [OVS-LAN1]                    [OVS-LAN2]
    │                    ╱    │    ╲                   ╱    │    ╲
    │               [VPCS] [webterm] [App Server]  [DHCP] [Ubuntu] [Monitoring]
    │                                          │                    │
    │                                    [PostgreSQL]       [Traffic Gen + Syslog]
    │
    └───── WAN (port1) ──── (attack traffic from OCI directly to FGT WAN IPs)
```

### Edge Details

| # | From | To | Label | Type |
|---|---|---|---|---|
| 1 | OCI | PUBLIC INTERNET | — | Internet |
| 2 | PUBLIC INTERNET | NAT1 | — | Internet |
| 3 | NAT1 | FGT-Primary port1 | WAN | Ethernet |
| 4 | NAT1 | FGT-Secondary port1 | WAN | Ethernet |
| 5 | FGT-Primary port3 | FGT-Secondary port3 | HA Heartbeat | Ethernet |
| 6 | FGT-Primary port2 | OVS-LAN1 | LAN | Ethernet |
| 7 | FGT-Secondary port2 | OVS-LAN2 | LAN | Ethernet |
| 8 | OVS-LAN1 | VPCS | — | Ethernet |
| 9 | OVS-LAN1 | webterm-1 | — | Ethernet |
| 10 | OVS-LAN1 | App Server | — | Ethernet |
| 11 | App Server | PostgreSQL | DB (5432) | Internal (container link, not a GNS3 cable) |
| 12 | OVS-LAN2 | Alpine DHCP | — | Ethernet |
| 13 | OVS-LAN2 | Ubuntu Client | — | Ethernet |
| 14 | OVS-LAN2 | Monitoring Stack | — | Ethernet |
| 15 | Monitoring Stack | Traffic Gen + Syslog | — | Internal |
| 16 | OCI | FGT-Primary (attack) | Attack traffic | Internet |
| 17 | OCI | FGT-Secondary (attack) | Attack traffic | Internet |

---

## 5. Host Resource Budget

| Component | RAM Estimate | Type | Running Count |
|---|---|---|---|
| FGT-Primary | 2048 MB | QEMU | 1 |
| FGT-Secondary | 2048 MB | QEMU | 1 |
| Ubuntu Desktop | 2048 MB | QEMU | 1 |
| OVS-LAN1 | ~100 MB | Docker | 1 |
| OVS-LAN2 | ~100 MB | Docker | 1 |
| webterm | ~200 MB | Docker | 1 |
| App Server | ~150 MB | Docker | 1 |
| PostgreSQL | ~200 MB | Docker | 1 |
| Alpine DHCP | ~20 MB | Docker | 1 |
| Monitoring Stack | ~400 MB | Docker | 1 (combined) |
| Traffic Gen + Syslog | ~30 MB | Docker | 1 |
| **Total Estimated** | **~7.3 GB** | — | **11 running** |

**Host System:** 32 GB RAM, 8+ cores — well within limits.

---

## 6. Docker Image Status

| Image | Status | Size |
|---|---|---|
| `alpine:latest` | ✅ Cached | 8.7 MB |
| `postgres:16-alpine` | ✅ Cached | 297 MB |
| `gns3/openvswitch:latest` | ✅ In GNS3 registry | — |
| `gns3/webterm:latest` | ✅ In GNS3 registry | — |
| `python:3.12-alpine` | ❌ Needs pull | ~50 MB |
| `grafana/grafana:latest` | ❌ Needs pull | ~200 MB |
| `prom/prometheus:latest` | ❌ Needs pull | ~200 MB |

---

## 7. Execution Phases

### Phase A — Core Security & Connectivity (Standalone)

| Wave | Tasks |
|---|---|
| **1 — Build** | Create GNS3 project, add all 14 nodes, wire per topology, start all nodes, verify console access |
| **2 — Alpine DHCP** | Console into Alpine, `apk add dnsmasq`, configure and start DHCP |
| **3 — FGT Config** | FGT1 + FGT2: WAN DHCP, LAN static IPs, static routes, SNAT policies |
| **4 — IPsec VPN** | Site-to-site IKEv2 IPsec between FGT1↔FGT2 |
| **5 — UTM Profiles** | AV, IPS, Web Filter, SSL Inspection, App Control policies |
| **6 — OCI App Deployment** | Deploy threat simulator, configure endpoints, test each |

### Phase B — High Availability (Cluster)

| Wave | Tasks |
|---|---|
| **1 — HA Config** | Clear FGT2 IPs, configure FGCP A-P with port3 heartbeat |
| **2 — Failover Testing** | Pull FGT1 cables, verify session pickup, automated recovery |
| **3 — Security Testing vs Cluster** | All Phase A security tests against active unit, verify logs merge |

---

## 8. Demo Scenarios Matrix

| # | Scenario | Trigger | Detection | Visible In |
|---|---|---|---|---|
| 1 | App availability | Ubuntu browser → App Server `192.168.10.10` | App loads | Browser + Grafana |
| 2 | EICAR blocked | webterm → `https://OCI-IP/eicar` | AV block page | Browser + Grafana alert |
| 3 | SQLi detected | OCI → `http://FGT1-WAN/?id=1'OR'1'=1` | IPS blocks | FGT log + Grafana alert |
| 4 | Phishing blocked | Ubuntu → `https://OCI-IP/phishing` | Web Filter block | Block page + Grafana |
| 5 | IPsec encrypting | VPCS pings `192.168.20.x` | Ping succeeds | Console + VPN monitor |
| 6 | HA failover | Pull FGT1 WAN cable | Traffic continues | Ping + Grafana shows failover |
| 7 | Attack during failover | OCI scans during switchover | No gap in coverage | FGT logs + Grafana |
| 8 | Live auto-traffic | Traffic Gen sends requests | All policies active | Real-time Grafana graphs |
| 9 | Syslog streaming | Every event sent | Logs in tail | Syslog console + Grafana |

---

## 9. FortiGate Eval License — Constraints & Workarounds

### Hard Limits
| Resource | Cap | Lab Allocation |
|---|---|---|
| Interfaces | **3** (port1, port2, port3) | All 3 used per FGT |
| Firewall Policies | **3** | LAN→WAN, IPsec, Mgmt — fits exactly |
| Routes | **3** | Default, LAN (auto), IPsec — fits exactly |
| vCPU | **1** | Template-configured |
| RAM | **2 GB** | Template-configured |
| FortiGuard | **Not included** | See workarounds below |
| Encryption | Low only (data plane) | AES128-SHA1 for IPsec |

### UTM Workarounds (No FortiGuard)
| Feature | Workaround | Demo Impact |
|---|---|---|
| **Antivirus** | EICAR test file is hardcoded in AV engine | ✅ Block page shown |
| **IPS** | SQLi/XSS/port-scan signatures are factory-built | ✅ Full detection |
| **Web Filter** | Static URL filter instead of dynamic categories | ✅ Same block page |
| **DNS Filter** | Static domain block list instead of rating | ✅ Same block effect |
| **App Control** | Factory signatures (browsers, common protocols) | ⚠️ Limited |
| **SSL Inspection** | Self-signed CA, no FortiGuard needed | ✅ Full functionality |

### Activation
Both FGTs already have active eval licenses via separate FortiCloud accounts.

---

## 10. Security Hardening Baseline

Applied during Phase A config:

| Area | Measure |
|---|---|
| Admin access | Whitelisted to LAN subnets only (`192.168.10.0/24`, `192.168.20.0/24`) |
| Admin port | Non-default (`10443`) |
| Admin TLS | TLS 1.2+ only |
| SSH ciphers | ChaCha20-Poly1305, AES256-GCM only |
| Idle timeout | 10 minutes |
| Unused ports | N/A — all 3 available ports are in use |
| WAN interface | Ping only — no HTTPS/SSH exposed |
| HA authentication | PSK on heartbeat link |
| Logging | All internet-bound traffic logged |
| Password policy | Min length 8, complexity enabled |

---

## 11. Reference

| Resource | Location |
|---|---|
| Topology Diagram | `03_GNS3_Labs/Topology.canvas` |
| Node Reference | `03_GNS3_Labs/Nodes-Reference.md` |
| GNS3 Project | `~/GNS3/projects/d311a72f-2416-4426-9138-96ccd23fe8fd/` |
| Images Directory | `~/GNS3/images/QEMU/` |
| Memory — Facts | `memory/facts.md` |
| Memory — Decisions | `memory/decisions.md` |
| Memory — Progress | `memory/progress.md` |
