---
title: Progress Template
tags:
  - memory/progress
  - meta/active
---

# Progress — Phase A Wave 2 — HA & Docker Services

## Phase: Wave 1-2 — HA Cluster & Docker Services

### Completed
- [x] Pull all Docker images (alpine, postgres:16-alpine, grafana/grafana, prom/prometheus, python:3.12-alpine, gns3/webterm)
- [x] Build custom alpine-dhcp image with dnsmasq
- [x] Create Docker bridge network `fortigate-lab` (192.168.10.0/24)
- [x] Deploy PostgreSQL-1 (192.168.10.11) — DB `appdb` created
- [x] Deploy App-Server (192.168.10.10) — Flask app on port 80, DB connected
- [x] Deploy Grafana-1 (192.168.10.20) — port 3000
- [x] Deploy Prometheus-1 (192.168.10.21) — port 9090
- [x] Deploy Alpine DHCP (192.168.10.2) — dnsmasq running
- [x] Deploy Traffic-Gen (192.168.10.22) — curl + busybox
- [x] Apply Firewall NAT Policies (LAN→WAN) on both FGTs
- [x] Configure HA Cluster (Active/Passive, port3 heartbeat)
- [x] Architecture updated: independent FGTs → HA cluster

### Pending
- [ ] Verify HA cluster sync via `get system ha status`
- [ ] Start OVS bridges within GNS3
- [ ] Start Ubuntu Desktop node
- [ ] webterm-1 in GNS3 (needs display server)

---
> [!info] How to use
> Copy this file to `progress/<your-name>.md` and track your own progress.
> The AI agent will read and update your progress file.
