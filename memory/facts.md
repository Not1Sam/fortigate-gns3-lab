# Technical Facts & Topology Specs

## Environment Baseline
- **Hypervisor**: GNS3 on Linux (QEMU/KVM via libvirt + Docker)
- **Firewall OS**: FortiOS v7.4.12 KVM (stable, works within 2 GB RAM limit)
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

## Network Addressing

| Segment | Subnet | Gateway | DHCP |
|---|---|---|---|
| WAN (NAT1) | `192.168.122.0/24` | `192.168.122.1` (virbr0) | virbr0/libvirt |
| LAN1 (FGT-Primary) | `192.168.10.0/24` | `192.168.10.1` (FGT-P port2) | FGT-P built-in DHCP |
| LAN2 (FGT-Secondary) | `192.168.20.0/24` | `192.168.20.1` (FGT-S port2) | Alpine dnsmasq |
| Transit | `10.0.0.0/30` | — | Static |
| IPsec Tunnel | `10.0.1.0/24` | — | — |

## Topology — 14 Nodes

### QEMU VMs (3)
| Node | RAM | vCPU | Adapters | Console | Image |
|---|---|---|---|---|---|
| FGT-Primary | 2 GB | 1 | 8 (3 usable) | Telnet | `fgt-v7.4.12.qcow2` (108 MB) |
| FGT-Secondary | 2 GB | 1 | 8 (3 usable) | Telnet | Linked clone of above |
| Ubuntu-Desktop-Client | 2 GB | 2 | 1 (e1000) | VNC | `ubuntu-24.04-minimal-cloudimg` (mod., pwd: `gns3`) |

### Docker Nodes (8)
| Node | Image | Adapters | Console | Purpose |
|---|---|---|---|---|
| OVS-LAN1 / OVS-LAN2 | `gns3/openvswitch:latest` | 16 | Telnet | L2 switching |
| webterm-1 | `gns3/webterm:latest` | 1 | VNC | Browser client |
| Alpine DHCP | `alpine:latest` (8.7 MB) | 1 | Telnet | dnsmasq on LAN2 |
| App Server | `python:3.12-alpine` (~50 MB) | 1 | Telnet | Flask+Nginx on port 80/443 |
| PostgreSQL | `postgres:16-alpine` (297 MB) | 1 | Telnet | DB for App Server on 5432 |
| Monitoring Stack | `grafana/grafana` + `prom/prometheus` | 1 | VNC (HTTP) | Grafana 3000 + Prometheus 9090 |
| Traffic Gen + Syslog | `alpine:latest` | 1 | Telnet | Auto-requests + syslog UDP 514 |

### Built-in / External (3)
| Node | Type | Purpose |
|---|---|---|
| PC1 | VPCS | CLI client (ping, traceroute) |
| NAT1 | GNS3 Cloud (virbr0) | WAN gateway, `192.168.122.1` |
| OCI Instance | Real cloud VM | Threat simulator (nmap, EICAR, SQLi, phishing) |

## Port Maps

### FGT-Primary
| Port | Connected To | Address |
|---|---|---|
| port1 | Switch1 (NAT1) | DHCP (`192.168.122.x`) |
| port2 | OVS-LAN1 | `192.168.10.1/24` |
| port3 | FGT-Secondary port3 | `10.0.0.1/30` (transit) |

### FGT-Secondary
| Port | Connected To | Address |
|---|---|---|
| port1 | Switch1 (NAT1) | DHCP (`192.168.122.x`) |
| port2 | OVS-LAN2 | `192.168.20.1/24` |
| port3 | FGT-Primary port3 | `10.0.0.2/30` (transit) |

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
| FGT (both) | `admin` | (none — set at first boot) | Web UI / CLI |
| Ubuntu Desktop | `ubuntu` | `gns3` | VNC login, SSH (NOPASSWD sudo) |
| Ubuntu root | `root` | `gns3` | `su -` from ubuntu |
| VPCS | — | — | No auth required |
| Alpine Docker | `root` | (none — container default) | Telnet |
| webterm | — | — | VNC, no auth |

> [!tip] Ubuntu credentials are persistent
> The GNS3 base image (`ubuntu-24.04-minimal-cloudimg-amd64.img`) has been modified with the ubuntu user pre-created. Any new linked clone node inherits these credentials.

## Key Constraints Summary
1. **2 FortiCloud accounts needed** — one eval license per account
2. **3 policies each FGT** — forces efficient policy design
3. **3 routes each FGT** — transit + default + LAN = 3
4. **No FortiGuard** — UTM uses factory signatures + static lists
5. **Low encryption data plane** — IPsec uses AES128-SHA1, not AES256-GCM
