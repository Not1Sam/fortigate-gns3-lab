# Design & Architectural Decisions

## Approach 1: Two Independent FGTs (no HA)
- **Problem**: HA with eval license (3-interface limit) makes VLAN subinterfaces impossible. HA forces both FGTs into same LAN config, losing LAN1/LAN2 separation.
- **Decision**: Run two independent FortiGates. Each has its own WAN, LAN, and a transit link between them (port3). No HA cluster. OSPF over transit for dynamic routing demo.
- **Outcome**: Covers all objectives except live HA failover — HA is studied on paper and documented.
- **Alternatives considered**:
  - *HA Active-Passive + VLANs*: 3-interface limit exceeded (port1 + port2 + VLAN10 + VLAN20 + port3 = 5)
  - *HA Active-Passive + flat LAN*: Works, but loses LAN isolation and cross-FGT features

## Transit Link Between FGTs (10.0.0.0/30)
- **Purpose**: Route traffic between LAN1 and LAN2, demonstrate OSPF dynamic routing
- **Design**: port3 on each FGT, directly connected, OSPF over the link
- **Outcome**: Both LANs can communicate through the transit while keeping separate firewall policies

## OSPF Over Static Routing
- **Decision**: Use OSPF on the transit link despite the 3-route limit. Connected routes are automatic; the two static routes (default + one for the other LAN) stay within 3.
- **Note**: The route limit may not be strictly enforced (community reports), but we design within it.

## Port3 for Transit (not HA)
- **Decision**: Eval license limits to ports 1-3. Port3 is the only available port for the inter-FGT link after WAN and LAN.
- **Impact**: No HA heartbeat link needed — FGTs operate independently.

## Two Separate OVS Switches (not one)
- **Decision**: OVS-LAN1 behind FGT-Primary port2, OVS-LAN2 behind FGT-Secondary port2
- **Outcome**: Independent LAN segments with separate firewalls, DHCP servers, and policies.

## OCI as External Threat Actor (not passive web server)
- **Decision**: Instead of a simple web server on OCI, deploy a multi-endpoint threat simulator that actively attacks the FGTs (nmap, SQLi, EICAR, phishing, brute-force)
- **Outcome**: Each endpoint proves a different security feature in a single live demo.

## No FortiAnalyzer / FortiManager
- **Decision**: Skip both — no eval licenses available. Built-in FGT logging + syslog covers this lab.
- **Workaround**: Traffic-Gen container acts as syslog receiver (UDP 514) + Grafana for dashboards.

## No FortiGuard UTM Updates
- **Decision**: All UTM features (AV, IPS, Web Filter, DNS Filter) configured with factory signatures + static lists
- **Workaround**: EICAR is hardcoded in AV engine. IPS SQLi/XSS signatures are factory-built. Web/DNS filtering uses static URL/domain block lists instead of dynamic category lookup.

## Ubuntu Disk Modified Directly (not cloud-init ISO)
- **Decision**: Modified base image via fuse2fs in a privileged container
- **Outcome**: `ubuntu` user with password `gns3`, NOPASSWD sudo, fully persistent across linked clones.

## HA Study Approach
- **Decision**: We study HA conceptually, build the config on paper, document it in the setup guide, then disable it
- **Rationale**: Live HA would consume the 3-port limit and block all other lab features. The supervisor's objective is to "study" HA, which is satisfied by learning and documenting it.
