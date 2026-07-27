# Progress — bingus

> Last updated: 2026-07-27

## Phase Progress
| Phase | Status | Notes |
|---|---|---|
| 1 — Base Setup | ✅ | 14-node topology created, wired, OVS bridges configured |
| 2 — FGT Config | ✅ | Interfaces, IPs, routes, DNS on both FGTs |
| 3 — LAN Services | ✅ | FGT-Primary DHCP + Alpine DHCP (LAN2), Ubuntu netplan fixed |
| 4 — Policies & NAT | ✅ | LAN→WAN SNAT policies on both FGTs |
| 5 — Routing (OSPF) | ❌ | |
| 6 — Docker Services | ✅ | All 6 containers deployed: PostgreSQL, App-Server (Flask), Grafana, Prometheus, Alpine DHCP, Traffic-Gen |
| 7 — Security Profiles | ❌ | |
| 8 — VPN | ❌ | |
| 9 — OCI Cloud | ❌ | |
| 10 — Logging & Demo | ❌ | |

## FGT Console Info
| FGT | WAN IP (DHCP) | Console |
|---|---|---|
| FGT-Primary | 192.168.122.x | DHCP — check via `get system interface physical` |
| FGT-Secondary | 192.168.122.x | DHCP — check via `get system interface physical` |

## Notes
- HA cluster configured: Active/Passive on port3 (FortiLab-HA, priority 200/100)
- Architecture switched from independent FGTs with transit/OSPF to HA cluster
- Post-HA: LAN1 and LAN2 merged into single subnet 192.168.10.0/24
- Docker containers deployed via Windows Docker engine on `fortigate-lab` bridge network
- webterm-1 container not running standalone (needs GNS3 display server)

## OS & Setup
- OS: Windows (Docker Desktop)
- GNS3 version: unknown
- Docker available: yes (7 images pulled, 6 containers running)
