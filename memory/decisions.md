# Design & Architectural Decisions

## Approach 2: HA Active-Passive Cluster (current)
- **Context**: Originally designed as two independent FGTs with transit link and OSPF. Later overridden by user decision to implement HA for live cluster demo.
- **Decision**: Active-Passive HA cluster on port3 (heartbeat). FGT-Primary = priority 200 (active), FGT-Secondary = priority 100 (passive). Group-name: FortiLab-HA.
- **Trade-off**: HA forces both FGTs into identical config. LAN1 and LAN2 merge into one flat LAN segment behind the virtual MAC. LAN isolation is lost.
- **Impact on 3-interface eval limit**: Port1=WAN, port2=LAN, port3=HA heartbeat (all 3 used). No transit/OSPF possible.
- **Alternatives considered**:
  - *HA Active-Passive + VLANs*: Exceeds 3-interface limit
  - *Two independent FGTs + transit*: Original design, preserves LAN isolation but no live HA demo

## LAN Subnet (Post-HA)
- **Decision**: Collapse both LAN segments into a single subnet 192.168.10.0/24 behind the HA cluster's active node port2.
- **DHCP**: FGT-Primary handles DHCP for the merged LAN. Alpine DHCP on old LAN2 side is reconfigured to serve the same subnet.
- **Docker Services**: All containers deployed on a single bridge network (`fortigate-lab`, 192.168.10.0/24) with static IPs.

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
