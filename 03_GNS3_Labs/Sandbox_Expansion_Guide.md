---
title: "Lab Expansion Guide"
tags:
  - gns3/labs
  - network/expansion
status: "active"
---

# Lab Expansion Guide

This guide describes how to extend the current dual-FGT HA topology with additional segments, services, or testing capabilities.

> **Warning:** This replaces an earlier multi-firewall-with-WAN-router guide that was based on an obsolete single-FGT + VyOS/Branch topology (archived July 20, 2026). The current topology uses dual-FGT HA with 2 OVS switches and no separate WAN router. See `Full-Topology-Spec.md` for the base architecture.

## Current Architecture Recap

- **FGT-Primary**: port1=WAN (DHCP via NAT1), port2=LAN1 (192.168.10.1/24 → OVS-LAN1), port3=HA (169.254.0.1/30)
- **FGT-Secondary**: port1=WAN (DHCP via NAT1), port2=LAN2 (192.168.20.1/24 → OVS-LAN2), port3=HA (169.254.0.2/30)
- **3-policy limit per FGT**: LAN→WAN (SNAT), IPsec (LAN1↔LAN2), Mgmt
- **3-route limit per FGT**: Default, LAN subnet, IPsec tunnel

## Expansion Options

### Option 1: Add a Third LAN Segment

**Problem:** All 3 ports are used (WAN, LAN, HA).

**Workaround:** Use VLAN sub-interfaces on port2 if you can spare a policy/route, or add a routed container behind the existing LAN:

1. Deploy a new Docker Alpine container on OVS-LAN1
2. Configure it as a router (`iptables` + IP forwarding)
3. It creates a new subnet behind the existing LAN segment
4. Traffic goes: Client → Alpine router → FGT → WAN

No extra interfaces, policies, or routes consumed on the FGT.

### Option 2: Add External Threat Sources

The OCI threat simulator covers 12 endpoints. To extend:

- **Second OCI instance** in a different region for geo-IP testing
- **Local Kali VM** on a separate GNS3 cloud bridge for LAN-side attacks
- **Botnet simulation script** on Traffic Gen node (cron-based randomized attacks)

### Option 3: Replace Syslog with Full SIEM

The current syslog receiver (Alpine + socat on UDP 514) forwards all FGT logs. To upgrade:

1. Deploy a Loki + Promtail stack via Podman in the monitoring container
2. Configure FGT log settings to send structured logs
3. Add Loki datasource to existing Grafana

No extra licenses needed. All log shipping uses built-in FGT syslog.

### Option 4: Extend to Physical Hardware

When staging on physical FortiGate:

1. Replace NAT1 with real ISP modem (DHCP on WAN)
2. Replace OVS switches with physical managed switches
3. Add dedicated HA heartbeat cable between physical FGTs
4. Deploy the same configs from this lab with adjusted interface names

---

## Document History

| Date | Change |
|---|---|
| 2026-07-07 | Original guide created (multi-FGT with VyOS WAN router, OSPF, 4 subnets, LDAP) |
| 2026-07-20 | Rewritten for dual-FGT HA architecture (no WAN router, 2 OVS, static routing, no LDAP/FAZ) |
