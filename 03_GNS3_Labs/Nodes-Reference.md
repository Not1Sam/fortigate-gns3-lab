# Nodes Reference — GNS3 FortiGate Lab

## All 14 Nodes

| # | Node Name | Template | Type | RAM | Adapters | Console | Image |
|---|---|---|---|---|---|---|---|
| 1 | **FGT-Primary** | FortiGate-7.4.12 | QEMU | 2048 MB | 8 (virtio) | Telnet | `fgt-v7.4.12.qcow2` |
| 2 | **FGT-Secondary** | FortiGate-7.4.12 | QEMU | 2048 MB | 8 (virtio) | Telnet | linked clone (same image) |
| 3 | **Ubuntu-Client** | Ubuntu-Desktop-Client | QEMU | 2048 MB | 1 (e1000) | VNC | `ubuntu-24.04-minimal-cloudimg.img` (mod.) |
| 4 | **OVS-LAN1** | Open vSwitch | Docker | — | 16 | Telnet | `gns3/openvswitch:latest` |
| 5 | **OVS-LAN2** | Open vSwitch | Docker | — | 16 | Telnet | `gns3/openvswitch:latest` |
| 6 | **webterm-1** | webterm | Docker | — | 1 | VNC | `gns3/webterm:latest` |
| 7 | **Alpine DHCP** | Alpine Linux | Docker | — | 1 | Telnet | `alpine:latest` |
| 8 | **App Server** | *(to create)* | Docker | — | 1 | Telnet | `python:3.12-alpine` *(to pull)* |
| 9 | **PostgreSQL** | *(to create)* | Docker | — | 1 | Telnet | `postgres:16-alpine` |
| 10 | **Monitoring Stack** | *(to create)* | Docker | — | 1 | VNC/HTTP | `grafana/grafana` + `prom/prometheus` |
| 11 | **Traffic Gen + Syslog** | *(to create)* | Docker | — | 1 | Telnet | `alpine:latest` |
| 12 | **PC1** | VPCS | Built-in | — | — | Telnet | Built-in |
| 13 | **NAT1** | Cloud (NAT) | Built-in | — | — | — | `virbr0` host bridge |
| 14 | **OCI Instance** | *(outside GNS3)* | Cloud VM | 1–6 GB | — | SSH | Ubuntu 24.04 |

## Port Summary

| Node | Connected Ports |
|---|---|
| FGT-Primary | port1→NAT1, port2→OVS-LAN1, port3→FGT-Secondary port3 |
| FGT-Secondary | port1→NAT1, port2→OVS-LAN2, port3→FGT-Primary port3 |
| OVS-LAN1 | FGT1 port2, VPCS, webterm, App Server |
| OVS-LAN2 | FGT2 port2, Alpine DHCP, Ubuntu Client, Monitoring Stack |

## Docker Images to Pull

```bash
podman pull python:3.12-alpine
podman pull grafana/grafana:latest
podman pull prom/prometheus:latest
```

## Credentials

| Node | User | Password | Method |
|---|---|---|---|
| Ubuntu Client | `ubuntu` | `gns3` | SSH / VNC console |
| Alpine DHCP | `root` | *(none, set on first boot)* | Telnet console |
| App Server | `root` | *(none)* | Telnet console |
| PostgreSQL | `app` | `app-pass` | Internal DB auth |
| FGT Admin | `admin` | *(set during setup)* | Telnet / HTTPS |
| Grafana | `admin` | `admin` | VNC → HTTP 3000 |
