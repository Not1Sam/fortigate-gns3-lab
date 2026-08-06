# Full Topology Specification (Modified)

## Architecture Overview

Four FortiGate VMs organized in 2 HA pairs, a Docker Router for cross-LAN routing, OVS switch fabric, Docker service containers, and an OCI cloud threat simulator.

### Key Design Principles
- **HA Active-Passive** — each FGT has a backup on the same LAN segment
- **Docker Router** — replaces OSPF transit, frees port3 for HA heartbeat
- **Physical LAN isolation** — OVS-LAN1 and OVS-LAN2 are completely separate
- **SSL VPN** — uses ssl.root virtual interface (no extra port)
- **All 18 nodes fit in 1 GNS3 project**

## Node Inventory

### QEMU VMs (5)
| # | Name | Type | Image | RAM | vCPU | Adapters | Console |
|---|---|---|---|---|---|---|---|
| 1 | FGT-Primary | QEMU | `fgt-v7.4.12.qcow2` | 2048 MB | 1 | 8 | Telnet |
| 2 | FGT-Primary-HA | QEMU | `fortios.qcow2` (7.0.9) | 2048 MB | 1 | 8 | Telnet |
| 3 | FGT-Secondary | QEMU | `fgt-v7.4.12.qcow2` | 2048 MB | 1 | 8 | Telnet |
| 4 | FGT-Secondary-HA | QEMU | `fortios.qcow2` (7.0.9) | 2048 MB | 1 | 8 | Telnet |
| 5 | Ubuntu Desktop | QEMU | `ubuntu-24.04-minimal-cloudimg` (mod) | 2048 MB | 2 | 1 | VNC |

### Docker Nodes (10)
| # | Name | Image | Adapters | Console | Role |
|---|---|---|---|---|---|
| 6 | OVS-LAN1 | `gns3/openvswitch:latest` | 16 | Telnet | OVS-LAN1 switch |
| 7 | OVS-LAN2 | `gns3/openvswitch:latest` | 16 | Telnet | OVS-LAN2 switch |
| 8 | Docker Router | `docker-router:latest` (custom) | 2 | Telnet | Cross-LAN routing |
| 9 | webterm-1 | `gns3/webterm:latest` | 1 | VNC | Client browser on LAN1 |
| 10 | Alpine-DHCP | `alpine-dhcp:latest` (custom) | 1 | Telnet | DHCP server for LAN2 |
| 11 | appServer-1 | `python:3.12-alpine` | 1 | Telnet | Flask app on LAN1 |
| 12 | PostgreSQL-1 | `postgres:16-alpine` | 1 | Telnet | DB backend on LAN1 |
| 13 | Grafana-1 | `grafana/grafana:latest` | 1 | VNC | Dashboards on LAN2 |
| 14 | Prometheus-1 | `prom/prometheus:latest` | 1 | VNC | Metrics on LAN2 |
| 15 | Traffic-Gen-1 | `alpine:latest` | 1 | Telnet | Syslog + threats on LAN2 |

### Built-in Nodes (3)
| # | Name | Type | Role |
|---|---|---|---|
| 16 | PC1 | VPCS | CLI client on LAN1 |
| 17 | NAT1 | Cloud (virbr0) | WAN gateway 192.168.122.1 |
| 18 | Switch1 | Ethernet switch | WAN distribution |

## Network Addressing

| Segment | Subnet | Gateway | DHCP Server |
|---|---|---|---|
| WAN | `192.168.122.0/24` | `192.168.122.1` (NAT1) | virbr0/libvirt |
| LAN1 | `192.168.10.0/24` | `192.168.10.1` (FGT-P port2) | FGT-Primary (100-200) |
| LAN2 | `192.168.20.0/24` | `192.168.20.1` (FGT-S port2) | Alpine DHCP (100-200) |
| HA Heartbeat | `169.254.0.0/30` | N/A (p2p) | Static |
| IPsec Tunnel | `10.0.1.0/24` | — | — |

### Static Assignments
| Node | IP | Segment |
|---|---|---|
| FGT-Primary port1 | DHCP (192.168.122.x) | WAN |
| FGT-Primary port2 | `192.168.10.1/24` | LAN1 |
| FGT-Primary port3 | `169.254.0.1/30` | HA Heartbeat |
| FGT-Primary-HA port1 | DHCP (192.168.122.x) | WAN |
| FGT-Primary-HA port2 | `192.168.10.2/24` | LAN1 (passive) |
| FGT-Primary-HA port3 | `169.254.0.2/30` | HA Heartbeat |
| FGT-Secondary port1 | DHCP (192.168.122.x) | WAN |
| FGT-Secondary port2 | `192.168.20.1/24` | LAN2 |
| FGT-Secondary port3 | `169.254.0.3/30` | HA Heartbeat |
| FGT-Secondary-HA port1 | DHCP (192.168.122.x) | WAN |
| FGT-Secondary-HA port2 | `192.168.20.2/24` | LAN2 (passive) |
| FGT-Secondary-HA port3 | `169.254.0.4/30` | HA Heartbeat |
| Docker Router eth0 | `192.168.10.254/24` | LAN1 |
| Docker Router eth1 | `192.168.20.254/24` | LAN2 |
| Alpine DHCP | `192.168.20.2/24` | LAN2 |
| Ubuntu Desktop | DHCP (192.168.20.x) | LAN2 |
| PC1 | DHCP (192.168.10.x) | LAN1 |
| webterm-1 | DHCP (192.168.10.x) | LAN1 |
| App-Server | Static `192.168.10.10/24` | LAN1 |
| PostgreSQL | Static `192.168.10.11/24` | LAN1 |
| Grafana | Static `192.168.20.11/24` | LAN2 |
| Prometheus | Static `192.168.20.12/24` | LAN2 |
| Traffic-Gen | Static `192.168.20.10/24` | LAN2 |

## Port Maps & Wiring

### FGT-Primary
| FGT Port | Adapter | Connected To | IP |
|---|---|---|---|
| port1 | a0p0 | Switch1 a0p1 | DHCP |
| port2 | a1p0 | OVS-LAN1 a0p0 | 192.168.10.1/24 |
| port3 | a2p0 | FGT-Primary-HA a2p0 | 169.254.0.1/30 |

### FGT-Primary-HA
| FGT Port | Adapter | Connected To | IP |
|---|---|---|---|
| port1 | a0p0 | Switch1 a0p3 | DHCP |
| port2 | a1p0 | OVS-LAN1 a1p0 | 192.168.10.2/24 (passive) |
| port3 | a2p0 | FGT-Primary a2p0 | 169.254.0.2/30 |

### FGT-Secondary
| FGT Port | Adapter | Connected To | IP |
|---|---|---|---|
| port1 | a0p0 | Switch1 a0p2 | DHCP |
| port2 | a1p0 | OVS-LAN2 a0p0 | 192.168.20.1/24 |
| port3 | a2p0 | FGT-Secondary-HA a2p0 | 169.254.0.3/30 |

### FGT-Secondary-HA
| FGT Port | Adapter | Connected To | IP |
|---|---|---|---|
| port1 | a0p0 | Switch1 a0p4 | DHCP |
| port2 | a1p0 | OVS-LAN2 a1p0 | 192.168.20.2/24 (passive) |
| port3 | a2p0 | FGT-Secondary a2p0 | 169.254.0.4/30 |

### OVS-LAN1
| OVS Port | Adapter | Connected To |
|---|---|---|
| eth0 | a0p0 | FGT-Primary a1p0 |
| eth1 | a1p0 | FGT-Primary-HA a1p0 |
| eth2 | a2p0 | PC1 a0p0 |
| eth3 | a3p0 | webterm-1 a0p0 |
| eth4 | a4p0 | App-Server a0p0 |
| eth5 | a5p0 | PostgreSQL a0p0 |
| eth6 | a6p0 | Docker Router a0p0 |

### OVS-LAN2
| OVS Port | Adapter | Connected To |
|---|---|---|
| eth0 | a0p0 | FGT-Secondary a1p0 |
| eth1 | a1p0 | FGT-Secondary-HA a1p0 |
| eth2 | a2p0 | Alpine-DHCP a0p0 |
| eth3 | a3p0 | Ubuntu Desktop a0p0 |
| eth4 | a4p0 | Traffic-Gen a0p0 |
| eth5 | a5p0 | Grafana a0p0 |
| eth6 | a6p0 | Prometheus a0p0 |
| eth7 | a7p0 | Docker Router a1p0 |

### Others
| Node | Adapter | Connected To |
|---|---|---|
| NAT1 | a0p0 | Switch1 a0p0 |
| Switch1 | a0p1 | FGT-Primary a0p0 |
| Switch1 | a0p2 | FGT-Secondary a0p0 |
| Switch1 | a0p3 | FGT-Primary-HA a0p0 |
| Switch1 | a0p4 | FGT-Secondary-HA a0p0 |

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
| FGT-Primary-HA | 2048 MB | 1 | 2048 MB |
| FGT-Secondary | 2048 MB | 1 | 2048 MB |
| FGT-Secondary-HA | 2048 MB | 1 | 2048 MB |
| Ubuntu Desktop | 2048 MB | 1 | 2048 MB |
| Docker containers | ~64 MB avg | 10 | ~640 MB |
| GNS3 server + overhead | — | — | ~512 MB |
| **Total** | | | **~9.3 GB** |

## Eval License Budget (per FGT)
| Resource | Used | Limit |
|---|---|---|
| Interfaces | 3 | 3 |
| Firewall Policies | 2-3 | 3 |
| Routes | 1 | 3 |
| vCPU | 1 | 1 |
| RAM | 2048 MB | 2048 MB |
