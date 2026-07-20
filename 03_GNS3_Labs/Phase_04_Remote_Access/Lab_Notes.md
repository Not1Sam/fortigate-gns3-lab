---
title: "Phase 04: VPN & Remote Access"
tags:
  - lab/vpn
  - lab/auth
status: "draft"
---

# Phase 04: Remote Access (IPsec VPN, SSL VPN)

> **Note:** Updated for current dual-FGT topology. LDAP/AD authentication removed per architecture decision (no LDAP server in current design). IPsec VPN is between FGT-Primary (LAN1) and FGT-Secondary (LAN2).

## IPsec Site-to-Site VPN — FGT-Primary

```fortinet
config vpn ipsec phase1-interface
    edit "to-LAN2"
        set interface port1
        set ike-version 2
        set peertype any
        set net-device enable
        set proposal aes128-sha1
        set remote-gw 192.168.122.x  (FGT-Secondary WAN IP)
        set psksecret changeme
    next
end

config vpn ipsec phase2-interface
    edit "to-LAN2-p2"
        set phase1name "to-LAN2"
        set proposal aes128-sha1
        set src-subnet 192.168.10.0 255.255.255.0
        set dst-subnet 192.168.20.0 255.255.255.0
    next
end
```

## IPsec Site-to-Site VPN — FGT-Secondary

```fortinet
config vpn ipsec phase1-interface
    edit "to-LAN1"
        set interface port1
        set ike-version 2
        set peertype any
        set net-device enable
        set proposal aes128-sha1
        set remote-gw 192.168.122.y  (FGT-Primary WAN IP)
        set psksecret changeme
    next
end

config vpn ipsec phase2-interface
    edit "to-LAN1-p2"
        set phase1name "to-LAN1"
        set proposal aes128-sha1
        set src-subnet 192.168.20.0 255.255.255.0
        set dst-subnet 192.168.10.0 255.255.255.0
    next
end
```

## IPsec Policy (slots 2 or 3)

This consumes a policy slot. After SNAT (slot 1) and IPsec (slot 2), one slot remains.

```fortinet
config firewall policy
    edit 2
        set name "IPsec_LAN1_to_LAN2"
        set srcintf "port2"
        set dstintf "port1"
        set action accept
        set srcaddr "all"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
    next
end
```

## Verification

```fortinet
diagnose vpn ike gateway list
diagnose vpn tunnel list
```
