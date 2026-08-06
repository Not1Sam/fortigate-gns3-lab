# Progress — bingus

> Last updated: 2026-08-05

## Actual Lab Status

| Item | Status | Details |
|---|---|---|
| GNS3 Project | ✅ | `d311a72f` — 16 nodes, 19 links |
| Nodes placed | ✅ | 4 FGTs, 3 OVS, NAT1, PC1, Ubuntu, webterm-1, appServer-1, PostgreSQL-1, DHCP-Server, DockerRouter-1, Prometheus-1, Grafana-1 |
| Docker Router | ✅ | Built, placed, wired to OVS-LAN1 (eth0) + OVS-LAN2 (eth1) |
| FGT-Primary | ✅ | port1=192.168.122.2, port2=192.168.10.1, port3=10.0.0.1 |
| FGT-Primary-HA | ✅ | port1=192.168.122.4, port2=192.168.10.2, port3=10.0.0.2 |
| FGT-Secondary | ✅ | port1=192.168.122.3, port2=192.168.20.1, port3=10.0.0.5 |
| FGT-Secondary-HA | ✅ | port1=192.168.122.5, port2=192.168.20.2, port3=10.0.0.6 |
| HA Cluster 1 | ✅ | "FortiLab-HA" — FGT-Primary (master, pri 200) + FGT-Primary-HA (backup, pri 100) |
| HA Cluster 2 | ✅ | "FortiLab-HA2" — FGT-Secondary (master, pri 200) + FGT-Secondary-HA (backup, pri 100) |
| Internet (LAN1) | ✅ | PC1 gets DHCP, pings 8.8.8.8 |
| Internet (LAN2) | ✅ | Ubuntu gets DHCP (192.168.20.141), pings 8.8.8.8 |
| Firewall policies | ⚠️ | Basic "ALL" service, logging enabled, no security profiles |
| DHCP LAN1 | ✅ | FGT-Primary (192.168.10.100-200) |
| DHCP LAN2 | ✅ | External Alpine DHCP server |
| DNS | ✅ | All 4 FGTs: 8.8.8.8, 1.1.1.1 |
| NAT forwarding | ✅ | `gns3-control forward-enable` on wlan0 |
| Security profiles | ❌ | **BLOCKED** — VM has no UTM license |
| Cross-LAN routing | ✅ | PC1 (192.168.10.101) ↔ Ubuntu (192.168.20.141) — fixed by setting different HA group-id |
| VPN IPsec | ❌ | Not configured yet |
| Docker containers | ✅ | appServer-1 (Flask), PostgreSQL-1, Prometheus-1, Grafana-1 all running |
| Syslog/logging | ✅ | Memory + syslog enabled on all 4 FGTs |
| Screenshots | ❌ | None taken yet |

## RESOLVED — Switch1 → OVS-WAN

Switch1 was replaced with OpenvSwitch-3 (OVS-WAN) for proper L2 switching.
All 4 FGTs + NAT1 connected via individual ports on OVS-WAN.

## RESOLVED — HA MAC Flapping

Both HA clusters had the same virtual MAC (00:09:0f:09:00:00), causing MAC flapping on OVS-WAN.
Fixed by setting different group-id:
- FGT-Primary: `config system ha → set group-id 1`
- FGT-Secondary: `config system ha → set group-id 2`

## RESOLVED — Docker Router ARP Issue

Docker Router ARP was a side effect of the Switch1 wiring issue.
Cross-LAN routing now goes through OVS-WAN and both FGTs' static routes + policies.

## Port Addressing

| Segment | Subnet |
|---|---|
| WAN | 192.168.122.0/24 (virbr0) |
| LAN1 | 192.168.10.0/24 |
| LAN2 | 192.168.20.0/24 |
| HA Cluster 1 | 10.0.0.0/30 (10.0.0.1 + 10.0.0.2) |
| HA Cluster 2 | 10.0.0.4/30 (10.0.0.5 + 10.0.0.6) |

## Key Notes

- HA uses port3 on each FGT for heartbeat
- FGT-Primary-HA lost SSH after HA formed (normal, use telnet)
- `gns3-control forward-enable` must be run for internet access
- Ubuntu minimal has no `ping` — install with `apt install iputils-ping`
- appServer-1 telnet console doesn't accept input (Docker console issue) — use docker exec
- LAN2 uses external Alpine DHCP (not FGT-Secondary)
- **UTM NOT LICENSED** — FortiGate VM v7.4.1 has no UTM license. Cannot apply security profiles.
- Prometheus/Grafana built from alpine:latest base (Docker Hub pull broken due to DNS)
- All Docker containers configured via docker exec (GNS3 Docker console doesn't accept input)
