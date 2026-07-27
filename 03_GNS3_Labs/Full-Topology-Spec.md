# Full Topology Specification

## Architecture Overview

Two independent FortiGate firewalls, each serving a separate LAN segment with its own WAN connection, OVS switch fabric, and clients. A transit link between the two FGTs provides inter-LAN routing and OSPF dynamic routing demonstration.

### Key Design Principles
- **No HA** — two independent gateways, each with its own eval license
- **Physical LAN isolation** — OVS-LAN1 and OVS-LAN2 are completely separate
- **Transit link** — /30 subnet between FGTs for inter-LAN traffic
- **OSPF** — dynamic routing over the transit link
- **All 14 nodes fit in 1 GNS3 project**

## Node Inventory

### QEMU VMs (3)
| # | Name | Type | Image | RAM | vCPU | Adapters | Console |
|---|---|---|---|---|---|---|---|
| 1 | FGT-Primary | QEMU | `fgt-v7.4.12.qcow2` | 2048 MB | 1 | 8 | Telnet |
| 2 | FGT-Secondary | QEMU | Linked clone | 2048 MB | 1 | 8 | Telnet |
| 3 | Ubuntu-Desktop-Client-1 | QEMU | `ubuntu-24.04-minimal-cloudimg` (mod) | 2048 MB | 2 | 1 | VNC |

### Docker Nodes (8)
| # | Name | Image | Adapters | Console | Role |
|---|---|---|---|---|---|
| 4 | OpenvSwitch-1 | `gns3/openvswitch:latest` | 16 | Telnet | OVS-LAN1 switch |
| 5 | OpenvSwitch-2 | `gns3/openvswitch:latest` | 16 | Telnet | OVS-LAN2 switch |
| 6 | webterm-1 | `gns3/webterm:latest` | 1 | VNC | Client browser on LAN1 |
| 7 | Alpine-DHCP | `alpine-dhcp:latest` | 1 | Telnet | DHCP server for LAN2 |
| 8 | appServer-1 | `python:3.12-alpine` | 1 | Telnet | Flask app on LAN1 |
| 9 | PostgreSQL-1 | `postgres:16-alpine` | 1 | Telnet | DB backend on LAN1 |
| 10 | Grafana-1 | `grafana/grafana:latest` | 1 | VNC | Dashboards on LAN2 |
| 11 | Prometheus-1 | `prom/prometheus:latest` | 1 | VNC | Metrics on LAN2 |
| 12 | Traffic-Gen-1 | `alpine:latest` | 1 | Telnet | Syslog + threats on LAN2 |

### Built-in Nodes (3)
| # | Name | Type | Role |
|---|---|---|---|
| 13 | PC1 | VPCS | CLI client on LAN1 |
| 14 | NAT1 | Cloud (virbr0) | WAN gateway 192.168.122.1 |
| 15 | Switch1 | Ethernet switch | WAN distribution |

## Network Addressing

| Segment | Subnet | Gateway | DHCP Server |
|---|---|---|---|
| WAN | `192.168.122.0/24` | `192.168.122.1` (NAT1) | virbr0/libvirt |
| LAN1 | `192.168.10.0/24` | `192.168.10.1` (FGT-P port2) | FGT-Primary (100-200) |
| LAN2 | `192.168.20.0/24` | `192.168.20.1` (FGT-S port2) | Alpine DHCP (100-200) |
| Transit | `10.0.0.0/30` | N/A (p2p) | Static |
| IPsec | `10.0.1.0/24` | — | — |

### Static Assignments
| Node | IP | Segment |
|---|---|---|
| FGT-Primary port1 | DHCP (192.168.122.x) | WAN |
| FGT-Primary port2 | `192.168.10.1/24` | LAN1 |
| FGT-Primary port3 | `10.0.0.1/30` | Transit |
| FGT-Secondary port1 | DHCP (192.168.122.x) | WAN |
| FGT-Secondary port2 | `192.168.20.1/24` | LAN2 |
| FGT-Secondary port3 | `10.0.0.2/30` | Transit |
| Alpine DHCP | `192.168.20.2/24` | LAN2 |
| Ubuntu Desktop | DHCP (192.168.20.x) | LAN2 |
| PC1 | DHCP (192.168.10.x) | LAN1 |
| webterm-1 | DHCP (192.168.10.x) | LAN1 |
| App-Server | Static `192.168.10.10/24` | LAN1 |
| PostgreSQL | Static `192.168.10.11/24` | LAN1 |
| Grafana | Static `192.168.20.10/24` | LAN2 |
| Prometheus | Static `192.168.20.11/24` | LAN2 |
| Traffic-Gen | Static `192.168.20.12/24` | LAN2 |

## Port Maps & Wiring

### FGT-Primary
| FGT Port | Adapter | Connected To | IP |
|---|---|---|---|
| port1 | a0p0 | Switch1 a1p1 | DHCP |
| port2 | a1p0 | OVS-LAN1 a0p0 | 192.168.10.1/24 |
| port3 | a2p0 | FGT-Secondary a2p0 | 10.0.0.1/30 |

### FGT-Secondary
| FGT Port | Adapter | Connected To | IP |
|---|---|---|---|
| port1 | a0p0 | Switch1 a1p2 | DHCP |
| port2 | a1p0 | OVS-LAN2 a0p0 | 192.168.20.1/24 |
| port3 | a2p0 | FGT-Primary a2p0 | 10.0.0.2/30 |

### OVS-LAN1 (OpenvSwitch-1)
| OVS Port | Adapter | Connected To |
|---|---|---|
| eth0 | a0p0 | FGT-Primary a1p0 |
| eth1 | a1p0 | PC1 a0p0 |
| eth2 | a2p0 | webterm-1 a0p0 |
| eth3 | a3p0 | App-Server a0p0 |
| eth4 | a4p0 | PostgreSQL a0p0 |

### OVS-LAN2 (OpenvSwitch-2)
| OVS Port | Adapter | Connected To |
|---|---|---|
| eth0 | a0p0 | FGT-Secondary a1p0 |
| eth1 | a1p0 | Alpine-DHCP a0p0 |
| eth2 | a2p0 | Ubuntu-Desktop-Client a0p0 |
| eth3 | a3p0 | Traffic-Gen a0p0 |
| eth4 | a4p0 | Grafana a0p0 |
| eth5 | a5p0 | Prometheus a0p0 |

### Others
| Node | Adapter | Connected To |
|---|---|---|
| NAT1 | a0p0 | Switch1 a0p0 |
| Switch1 | a0p1 | FGT-Primary a0p0 |
| Switch1 | a0p2 | FGT-Secondary a0p0 |

## OCI Threat Simulator Endpoints (External)
| Endpoint | Traffic | What It Proves |
|---|---|---|
| `GET /inspect` | HTTP | SNAT |
| `GET /api/status` | HTTP | API routing |
| `GET /eicar` | HTTP+HTTPS | AV block |
| `GET /malware-sample` | HTTP+HTTPS | IPS block |
| `GET /attack?sql=payload` | HTTP | IPS SQLi |
| `GET /attack?xss=payload` | HTTP | IPS XSS |
| `GET /phishing` | HTTPS | Web Filter |
| `GET /hacking-tools` | HTTPS | Web Filter |
| `GET /proxy` | HTTPS | Web Filter |
| `POST /bruteforce` | HTTPS | Admin lockout |
| DNS `phish.test.lab` | DNS | DNS filter |
| `nmap -sS FGT-WAN` | TCP SYN | IPS scan detect |

## Host Resource Budget
| Component | RAM | Count | Total |
|---|---|---|---|
| FGT-Primary | 2048 MB | 1 | 2048 MB |
| FGT-Secondary | 2048 MB | 1 | 2048 MB |
| Ubuntu Desktop | 2048 MB | 1 | 2048 MB |
| Docker containers | ~64 MB avg | 8 | ~512 MB |
| GNS3 server + overhead | — | — | ~512 MB |
| **Total** | | | **~7.3 GB** |

## Eval License Budget (per FGT)
| Resource | Used | Limit |
|---|---|---|
| Interfaces | 3 | 3 |
| Firewall Policies | 3 | 3 |
| Routes | 3 | 3 |
| vCPU | 1 | 1 |
| RAM | 2048 MB | 2048 MB |
