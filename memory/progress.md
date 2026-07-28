---
title: Progress
tags:
  - memory/progress
---

# Progress

## Done
- Pre-licensed FortiOS 7.0.9 image (`fortios.qcow2`) acquired, tested, documented
- Created [[FortiGate-7.0.9-PreLicensed]] with Windows import guide
- Wiring audit: 16 nodes, 16 links, zero conflicts
- Added Traffic-Gen-2 Docker node on LAN2
- PC1 wired to OVS-1 (LAN1)
- All Docker nodes set to `GNS3_USER=root` for container startup fix
- Created [[Docker-Services-Guide]] with full app deployment steps
- Updated Device-Setup-Guide with Traffic-Gen-2 and 7.0.9 image
- Added 7.0.9 pre-licensed image to Linux and Windows setup guides

## Next
- Start Docker nodes in GNS3 → configure IPs per [[Docker-Services-Guide]]
- Setup OSPF on transit link between FGTs
- Security profiles (AV, IPS, Web Filter, SSL Inspection)
- Add Kali (WAN) and OCI threat sim nodes
- FGT port3 IPs still set to old 169.254.0.x (HA) — update to 10.0.0.x for transit
