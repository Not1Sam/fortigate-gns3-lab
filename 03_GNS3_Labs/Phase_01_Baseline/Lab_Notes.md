---
title: "Phase 01: Setup Baseline & HA"
tags:
  - lab/baseline
  - lab/ha
status: "draft"
---

# Phase 01: Lab Setup & High Availability

> **Note:** This file has been updated for the current dual-FGT HA architecture (July 20 redesign). The previous version referenced a single-FGT topology with port4 HA and 10.0.x.x IPs.

## Current Topology IP Plan

| Node | Port | Subnet | Gateway | Role |
|---|---|---|---|---|
| FGT-Primary | port1 | DHCP (192.168.122.x) | 192.168.122.1 | WAN — NAT1 virbr0 |
| FGT-Primary | port2 | 192.168.10.1/24 | — | LAN1 — OVS-LAN1 |
| FGT-Primary | port3 | 169.254.0.1/30 | — | HA heartbeat |
| FGT-Secondary | port1 | DHCP (192.168.122.x) | 192.168.122.1 | WAN — NAT1 virbr0 |
| FGT-Secondary | port2 | 192.168.20.1/24 | — | LAN2 — OVS-LAN2 |
| FGT-Secondary | port3 | 169.254.0.2/30 | — | HA heartbeat |

## Phase A Wave 1 — Build

1. Create GNS3 project
2. Add all 14 nodes
3. Wire per topology (see Full-Topology-Spec.md or Topology.canvas)
4. Start all nodes
5. Verify console access to each

## Phase A Wave 2 — Alpine DHCP

```bash
# Console into Alpine DHCP container
apk add dnsmasq
vi /etc/dnsmasq.conf
```

DHCP config:
```
interface=eth0
dhcp-range=192.168.20.100,192.168.20.200,255.255.255.0,24h
dhcp-option=3,192.168.20.1
dhcp-option=6,192.168.20.1
```

```bash
rc-update add dnsmasq default
service dnsmasq start
```

## Phase A Wave 3 — FGT Baseline Config

### FGT-Primary

```fortinet
config system interface
    edit port1
        set mode dhcp
        set allowaccess ping
    next
    edit port2
        set ip 192.168.10.1 255.255.255.0
        set allowaccess ping https ssh
    next
    edit port3
        set ip 169.254.0.1 255.255.255.252
        set allowaccess ping
    next
end

config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 192.168.122.1
        set device port1
    next
end
```

### FGT-Secondary

```fortinet
config system interface
    edit port1
        set mode dhcp
        set allowaccess ping
    next
    edit port2
        set ip 192.168.20.1 255.255.255.0
        set allowaccess ping https ssh
    next
    edit port3
        set ip 169.254.0.2 255.255.255.252
        set allowaccess ping
    next
end

config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 192.168.122.1
        set device port1
    next
end
```

## Phase B — HA Cluster

> **Note:** HA config comes *after* IPsec and UTM profiles are tested in standalone mode (Phase A). This avoids config sync complications during initial setup.

```fortinet
# On FGT-Primary (priority 200)
config system ha
    set group-id 10
    set group-name "FGT-HA-Cluster"
    set mode a-p
    set hbdev port3 50
    set priority 200
    set monitor port1 port2
end

# On FGT-Secondary (priority 100)
config system ha
    set group-id 10
    set group-name "FGT-HA-Cluster"
    set mode a-p
    set hbdev port3 50
    set priority 100
    set monitor port1 port2
end
```

## Verification

```fortinet
get system ha status
diagnose sys ha status
execute ha synchronize start
```
