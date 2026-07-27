# Quick Node Reference

## QEMU VMs
| Name | Image | RAM | Console | Credentials |
|---|---|---|---|---|
| FGT-Primary | `fgt-v7.4.12.qcow2` | 2 GB | Telnet | `admin` / no password |
| FGT-Secondary | Linked clone | 2 GB | Telnet | `admin` / no password |
| Ubuntu Desktop | `ubuntu-24.04-minimal-cloudimg` (mod) | 2 GB | VNC | `ubuntu` / `gns3` |

## Docker Nodes
| Name | Image | Adapters | Console | Purpose |
|---|---|---|---|---|
| OpenvSwitch-1 | `gns3/openvswitch:latest` | 16 | Telnet | LAN1 switch fabric |
| OpenvSwitch-2 | `gns3/openvswitch:latest` | 16 | Telnet | LAN2 switch fabric |
| webterm-1 | `gns3/webterm:latest` | 1 | VNC | Browser on LAN1 |
| Alpine-DHCP | `alpine-dhcp:latest` | 1 | Telnet | DHCP on LAN2 |
| appServer-1 | `python:3.12-alpine` | 1 | Telnet | Flask app on LAN1 |
| PostgreSQL-1 | `postgres:16-alpine` | 1 | Telnet | DB backend |
| Grafana-1 | `grafana/grafana:latest` | 1 | VNC | Dashboards on LAN2 |
| Prometheus-1 | `prom/prometheus:latest` | 1 | VNC | Metrics on LAN2 |
| Traffic-Gen-1 | `alpine:latest` | 1 | Telnet | Syslog + threats on LAN2 |

## Built-in Nodes
| Name | Type | Purpose |
|---|---|---|
| PC1 | VPCS | CLI client on LAN1 |
| NAT1 | Cloud (virbr0) | WAN gateway |
| Switch1 | Ethernet switch | WAN distribution |

## IP Assignments
| Node | IP | Subnet |
|---|---|---|
| FGT-Primary port1 | DHCP | WAN |
| FGT-Primary port2 | 192.168.10.1/24 | LAN1 |
| FGT-Primary port3 | 10.0.0.1/30 | Transit |
| FGT-Secondary port1 | DHCP | WAN |
| FGT-Secondary port2 | 192.168.20.1/24 | LAN2 |
| FGT-Secondary port3 | 10.0.0.2/30 | Transit |
| Alpine DHCP | 192.168.20.2/24 | LAN2 |
| App-Server | 192.168.10.10/24 | LAN1 |
| PostgreSQL | 192.168.10.11/24 | LAN1 |
| Grafana | 192.168.20.10/24 | LAN2 |
| Prometheus | 192.168.20.11/24 | LAN2 |
| Traffic-Gen | 192.168.20.12/24 | LAN2 |

## FGT Web UI
| FGT | WAN IP | URL |
|---|---|---|
| FGT-Primary | 192.168.122.x (DHCP) | `https://<wan-ip>` |
| FGT-Secondary | 192.168.122.x (DHCP) | `https://<wan-ip>` |
