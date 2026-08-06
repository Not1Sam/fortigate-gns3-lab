# Quick Node Reference (Modified Topology)

## QEMU VMs
| Name | Image | RAM | Console | Credentials |
|---|---|---|---|---|
| FGT-Primary | `fgt-v7.4.12.qcow2` | 2 GB | Telnet | `admin` / no password |
| FGT-Primary-HA | `fortios.qcow2` (7.0.9) | 2 GB | Telnet | `admin` / no password |
| FGT-Secondary | `fgt-v7.4.12.qcow2` | 2 GB | Telnet | `admin` / no password |
| FGT-Secondary-HA | `fortios.qcow2` (7.0.9) | 2 GB | Telnet | `admin` / no password |
| Ubuntu Desktop | `ubuntu-24.04-minimal-cloudimg` (mod) | 2 GB | VNC | `ubuntu` / `gns3` |

## Docker Nodes
| Name | Image | Adapters | Console | Purpose |
|---|---|---|---|---|
| OVS-LAN1 | `gns3/openvswitch:latest` | 16 | Telnet | LAN1 switch fabric |
| OVS-LAN2 | `gns3/openvswitch:latest` | 16 | Telnet | LAN2 switch fabric |
| Docker Router | `docker-router:latest` (custom) | 2 | Telnet | Cross-LAN routing |
| webterm-1 | `gns3/webterm:latest` | 1 | VNC | Browser on LAN1 |
| Alpine-DHCP | `alpine-dhcp:latest` (custom) | 1 | Telnet | DHCP on LAN2 |
| appServer-1 | `python:3.12-alpine` | 1 | Telnet | Flask app on LAN1 |
| PostgreSQL-1 | `postgres:16-alpine` | 1 | Telnet | DB backend |
| Grafana-1 | `grafana/grafana:latest` | 1 | VNC | Dashboards on LAN2 |
| Prometheus-1 | `prom/prometheus:latest` | 1 | VNC | Metrics on LAN2 |
| Traffic-Gen-1 | `alpine:latest` | 1 | Telnet | Syslog + threats on LAN2 |

## Built-in Nodes
| Name | Type | Purpose |
|---|---|---|
| PC1 | VPCS | Client CLI on LAN1 |
| NAT1 | Cloud (virbr0) | WAN gateway |
| Switch1 | Ethernet switch | WAN distribution |

## IP Assignments
| Node | IP | Subnet |
|---|---|---|
| FGT-Primary port1 | DHCP | WAN |
| FGT-Primary port2 | 192.168.10.1/24 | LAN1 |
| FGT-Primary port3 | 169.254.0.1/30 | HA Heartbeat |
| FGT-Primary-HA port1 | DHCP | WAN |
| FGT-Primary-HA port2 | 192.168.10.2/24 | LAN1 (passive) |
| FGT-Primary-HA port3 | 169.254.0.2/30 | HA Heartbeat |
| FGT-Secondary port1 | DHCP | WAN |
| FGT-Secondary port2 | 192.168.20.1/24 | LAN2 |
| FGT-Secondary port3 | 169.254.0.3/30 | HA Heartbeat |
| FGT-Secondary-HA port1 | DHCP | WAN |
| FGT-Secondary-HA port2 | 192.168.20.2/24 | LAN2 (passive) |
| FGT-Secondary-HA port3 | 169.254.0.4/30 | HA Heartbeat |
| Docker Router eth0 | 192.168.10.254/24 | LAN1 |
| Docker Router eth1 | 192.168.20.254/24 | LAN2 |
| Alpine DHCP | 192.168.20.2/24 | LAN2 |
| App-Server | 192.168.10.10/24 | LAN1 |
| PostgreSQL | 192.168.10.11/24 | LAN1 |
| Grafana | 192.168.20.11/24 | LAN2 |
| Prometheus | 192.168.20.12/24 | LAN2 |
| Traffic-Gen | 192.168.20.10/24 | LAN2 |

## FGT Web UI
| FGT | WAN IP | URL |
|---|---|---|
| FGT-Primary | 192.168.122.x (DHCP) | `https://<wan-ip>` |
| FGT-Secondary | 192.168.122.x (DHCP) | `https://<wan-ip>` |

## HA Clusters
| Cluster | Master | Backup | Group Name | Heartbeat |
|---|---|---|---|---|
| Cluster 1 | FGT-Primary (200) | FGT-Primary-HA (100) | FortiLab-HA | port3 (169.254.0.0/30) |
| Cluster 2 | FGT-Secondary (200) | FGT-Secondary-HA (100) | FortiLab-HA2 | port3 (169.254.0.0/30) |
