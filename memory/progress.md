# Current Status

> Last updated: 2026-07-27

## Re-Design: HA → Transit Link
- **Old design**: Two FGTs in FGCP HA cluster, port3 = HA heartbeat (169.254.0.0/30)
- **New design**: Two independent FGTs, port3 = transit link (10.0.0.0/30), OSPF routing
- **Reason**: HA required 5 logical interfaces (port1 + port2 + VLAN10 + VLAN20 + port3) but eval caps at 3 physical ports. Dropped HA, kept transit for inter-LAN routing.

### Done
- [x] GNS3 project created — 15 nodes placed, all images verified
- [x] Wiring complete except transit link (FGT-Primary port3 ↔ FGT-Secondary port3 missing)
- [x] FGT-Primary base config: port1 DHCP, port2 192.168.10.1/24, LAN1 DHCP, LAN1→WAN NAT
- [x] FGT-Secondary base config: port1 DHCP, port2 192.168.20.1/24, LAN2→WAN NAT
- [x] Alpine DHCP custom image built (alpine-dhcp:latest), LAN2 DHCP range 192.168.20.100-200
- [x] Ubuntu Desktop netplan DHCP configured
- [x] OVS bridges (br0) configured on both OVS nodes
- [x] Docker images pulled (python:3.12-alpine, postgres:16-alpine, grafana/grafana, prom/prometheus)
- [x] GNS3 init.sh patched — su fallback for Docker containers with missing/invalid shell users
- [x] Ubuntu disk modified — user ubuntu, password gns3, NOPASSWD sudo
- [x] Setup guides created: Setup-Guide-Linux.md, Setup-Guide-Windows.md
- [x] Device-Setup-Guide.md (per-node instructions for all 16 devices)
- [x] Vault files synced: Full-Topology-Spec.md, Topology.canvas (colored), Nodes-Reference.md
- [x] memory/init.md updated with OS detection + guide selection

### Missing / Next
- [ ] Wire transit link: FGT-Primary port3 ↔ FGT-Secondary port3
- [ ] Enable NAT forwarding (gns3-control forward-enable — needs fingerprint sudo)
- [ ] Docker service nodes: PostgreSQL, App-Server, Grafana, Prometheus, Traffic-Gen need IP config + app deployment
- [ ] FGT OSPF config on both FGTs
- [ ] Security profiles (AV, IPS, Web Filter, App Control, SSL Inspection)
- [ ] IPsec VPN to OCI (blocked on OCI deployment)
- [ ] OCI threat simulator (blocked on OCI deployment)
- [ ] Syslog → Grafana demo setup
