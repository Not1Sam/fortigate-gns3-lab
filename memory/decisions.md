# Design & Architectural Decisions

## Dual-FortiGate HA Topology (replaces single-firewall VLAN-on-a-stick)
- **Problem**: Original design used one FGT with VLAN sub-interfaces, but couldn't demo HA failover or IPsec VPN between peers
- **Decision**: Two FortiGates, each with 2 FortiCloud accounts for eval licenses. FGT1 = Active, FGT2 = Passive in FGCP cluster. HA heartbeat on port3 (not port7 — eval license limits to ports 1-3)
- **Outcome**: Can demo active-passive failover, session pickup, IPsec VPN, and cross-FGT routing

## Port3 for HA Heartbeat (not port7)
- **Decision**: Eval license limits to ports 1-3. Port3 used for HA instead of the conventional port7
- **Impact**: No functional difference — FGCP HA works on any interface

## Two Separate OVS Switches (not one)
- **Problem**: Single OVS couldn't demonstrate DHCP relay or independent LAN segments
- **Decision**: OVS-LAN1 behind FGT1 port2, OVS-LAN2 behind FGT2 port2
- **Outcome**: Can test DHCP relay across FGTs, IPsec between isolated LANs, and independent policy enforcement

## OCI as External Threat Actor (not passive web server)
- **Decision**: Instead of a simple web server on OCI, deploy a multi-endpoint threat simulator that actively attacks the FGTs (nmap, SQLi, EICAR, phishing, brute-force)
- **Outcome**: Each endpoint proves a different FortiGuard feature in a single live demo

## No FortiAnalyzer / FortiManager
- **Decision**: Skip both — no eval licenses available. Built-in FGT logging covers this lab
- **Workaround**: Alpine Docker container with socat acts as syslog receiver (UDP 514)

## No Dynamic Routing (OSPF/BGP)
- **Decision**: Eval license limits to 3 routes per FGT. Static routing only
- **Impact**: 3 routes per FGT is exactly enough for default route + LAN + IPsec tunnel

## No FortiGuard UTM Updates
- **Decision**: All UTM features (AV, IPS, Web Filter, DNS Filter) configured with factory signatures + static lists
- **Workaround**: EICAR is hardcoded in AV engine. IPS SQLi/XSS signatures are factory-built. Web/DNS filtering uses static URL/domain block lists instead of dynamic category lookup

## Ubuntu Disk Modified Directly (not cloud-init ISO)
- **Problem**: Cloud-init ISO attachment failed via GNS3 QEMU properties. No sudo on host for libguestfs/qemu-nbd
- **Solution**: Flattened qcow2 backing chain → extracted root partition to raw → mounted via fuse2fs in privileged Podman container → modified `/etc/shadow` and seeded cloud-init → wrote back to standalone qcow2
- **Outcome**: `ubuntu` user with password `gns3`, NOPASSWD sudo, fully persistent

## Deployment Phases
- **Phase A**: Standalone FGTs — routing, NAT, IPsec VPN, UTM profiles, OCI threat sim integration
- **Phase B**: HA cluster — FGCP active-passive, failover testing, attack resilience during failover
