---
title: Current Status
tags:
  - memory/progress
  - meta/active
---

# Current Status

## Phase: Wave 1 — Topology Build (In Progress)

### Completed
- [x] Project initialized (PROJECT.md, ROADMAP.md, REQUIREMENTS.md, STATE.md)
- [x] Topology designed — 15 nodes, dual FGTs, 2 OVS, WAN Switch, LAN clients, OCI threat sim
- [x] All technical documents created in vault (`03_GNS3_Labs/`)
  - `Full-Topology-Spec.md` — complete technical spec
  - `Topology.canvas` — visual topology diagram (updated with WAN Switch)
  - `Nodes-Reference.md` — quick node reference
  - `Topology-Setup-Guide.md` — step-by-step agent guide
- [x] `_INIT_.md` — agent init file for collaborators
- [x] FortiGate eval license limits researched and documented
- [x] Ubuntu base image modified — `ubuntu` user pre-created, password `gns3`, NOPASSWD sudo, cloud-init disabled. Any new linked clone inherits these credentials.
- [x] GNS3 project created — 10 nodes placed and wired (FGTs, OVS switches, WAN Switch, NAT1, VPCS, webterm, Alpine DHCP, Ubuntu Desktop)
- [x] NAT forwarding enabled (iptables MASQUERADE + FORWARD rules)
- [x] GNS3 project pushed to public GitHub: `https://github.com/Not1Sam/fortigate-gns3-lab`
- [x] Vault documents committed and pushed to same repo

### Phase A Wave 1 — Topology Build
- [x] FGT-Primary base config completed
  - port1: DHCP from NAT1 (`192.168.122.x`)
  - port2: `192.168.10.1/24` (LAN1) with HTTPS/SSH/ping access
  - port3: `169.254.0.1/30` (HA)
  - Static route: `0.0.0.0/0` via `192.168.122.1`
  - DNS: `8.8.8.8`, `1.1.1.1`
  - Internet: verified with `execute ping 8.8.8.8` ✅
- [x] FGT-Primary DHCP server configured for LAN1
  - Range: `192.168.10.100` — `192.168.10.200`
  - Gateway: `192.168.10.1`
  - DNS: default (8.8.8.8)
- [x] FGT-Secondary base config completed
  - port1: static `192.168.122.3`, WAN access verified ✅
  - port2: `192.168.20.1/24` (LAN2) with HTTPS/SSH/ping access
  - port3: `169.254.0.2/30` (HA)
  - Static route, DNS, internet verified ✅
  - Firewall policy LAN2-to-WAN (NAT enabled) ✅
- [x] Alpine DHCP server on LAN2 (dnsmasq)
  - Static IP: `192.168.20.2/24` on eth0, gateway `192.168.20.1`
  - dnsmasq: range `192.168.20.100` — `192.168.20.200`, 12h lease
  - DNS option: `8.8.8.8`
- [ ] FGT-Primary LAN1-to-WAN firewall policy
- [ ] Ubuntu Desktop console login verified (DHCP from LAN1)
- [ ] webterm console verified
- [ ] Verify OVS bridges forward traffic

### Phase A Wave 2-6
(Not yet started)

### Phase B: HA Cluster
(Not yet started)

