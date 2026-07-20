---
title: "Phase 02: Traffic Control & NAT"
tags:
  - lab/policy
  - lab/nat
status: "draft"
---

# Phase 02: Traffic Control & NAT Policies

> **Note:** Updated for current dual-FGT topology (July 20 redesign). Old version referenced DMZ subnet (10.0.2.0/24) and port3→port3 DNAT — no DMZ exists in current design; App Server is on LAN1.

## Outbound Source NAT (IP Pool)

```fortinet
config firewall ippool
    edit "WAN_IP_Pool"
        set startip 192.168.122.200
        set endip 192.168.122.210
        set type overload
    next
end
```

### Policy 1: LAN1 → WAN (on FGT-Primary)

```fortinet
config firewall policy
    edit 1
        set name "LAN1_to_WAN"
        set srcintf "port2"
        set dstintf "port1"
        set action accept
        set srcaddr "all"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
        set ippool enable
        set poolname "WAN_IP_Pool"
        set nat enable
    next
end
```

### Policy 1: LAN2 → WAN (on FGT-Secondary)

Same config substituting `port2` LAN2 IPs scope.

## Inbound DNAT (Virtual IP)

App Server is at `192.168.10.10:80` on OVS-LAN1 behind FGT-Primary:

```fortinet
config firewall vip
    edit "App_Server_HTTP"
        set extip 192.168.122.150
        set extport 80
        set mappedip 192.168.10.10
        set mappedport 80
        set portforward enable
    next
end
```

### Inbound Policy (port1 → port2)

```fortinet
config firewall policy
    edit 2
        set name "WAN_to_App_Server"
        set srcintf "port1"
        set dstintf "port2"
        set action accept
        set srcaddr "all"
        set dstaddr "App_Server_HTTP"
        set schedule "always"
        set service "HTTP"
    next
end
```

> **Note:** This consumes policy slot 2 of 3. Slot 3 is reserved for IPsec (LAN1↔LAN2 cross-FGT) or management.

## Verification

```bash
# On FortiGate
diagnose firewall session list | grep 192.168.122.150
diagnose sniffer packet port1 'port 80' 4

# From WAN side (NAT1 or OCI)
curl http://192.168.122.150

# From LAN client
ping 8.8.8.8
diagnose firewall session list | grep 8.8.8.8
```
