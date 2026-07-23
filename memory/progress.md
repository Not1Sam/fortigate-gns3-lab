---
title: Current Status
tags:
  - memory/progress
  - meta/active
---

# Current Status

## Phase: Wave 1 — Topology Build (LAN Services Verified)

### Completed
- [x] Project initialized (PROJECT.md, ROADMAP.md, REQUIREMENTS.md, STATE.md)
- [x] Topology designed — 15 nodes, dual FGTs, 2 OVS, WAN Switch, LAN clients, OCI threat sim
- [x] All technical documents created in vault (`03_GNS3_Labs/`)
- [x] `_INIT_.md` — agent init file for collaborators
- [x] FortiGate eval license limits researched and documented
- [x] Ubuntu base image modified — `ubuntu` / `gns3`, NOPASSWD sudo, cloud-init disabled
- [x] GNS3 project created — 10 nodes placed and wired
- [x] NAT forwarding enabled (iptables MASQUERADE + FORWARD rules)
- [x] Repo pushed to public GitHub: `https://github.com/Not1Sam/fortigate-gns3-lab`

### Layer 3 — Internet Access
- [x] FGT-Primary base config (interfaces, static route, DNS, internet) ✅
- [x] FGT-Secondary base config (interfaces, static route, DNS, internet) ✅
- [x] FGT-Primary LAN1-to-WAN firewall policy (NAT enabled) ✅
- [x] FGT-Secondary LAN2-to-WAN firewall policy (NAT enabled) ✅

### DHCP Services
- [x] FGT-Primary DHCP server for LAN1 (`192.168.10.100` — `192.168.10.200`) ✅
- [x] Alpine DHCP server on LAN2 (`192.168.20.100` — `192.168.20.200`) ✅
  - Custom Docker image built (`alpine-dhcp:latest`) with dnsmasq pre-installed
  - start_command writes dnsmasq.conf and configures IP automatically on every start
  - Container restart: fully automated
- [x] Ubuntu Desktop netplan configured for DHCP on enp2s0 (persistent across reboots) ✅

### Clients Verified
- [x] PC1 (VPCS) on LAN1 — DHCP + internet ✅
- [x] webterm-1 on LAN1 — DHCP + internet ✅
- [x] Alpine DHCP on LAN2 — static IP `192.168.20.2` + dnsmasq running ✅
- [x] Ubuntu Desktop on LAN2 — auto DHCP from Alpine ✅

### Pending
- [ ] Docker service nodes (App Server, PostgreSQL, Monitoring, Traffic Gen)
- [ ] OCI cloud instance — Libreswan + threat simulator
- [ ] IPsec VPN between FGTs and OCI
- [ ] HA cluster between FGTs
- [ ] UTM security profiles
- [ ] SD-WAN configuration

