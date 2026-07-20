---
title: "Phase 03: Cryptography & Security Profiles"
tags:
  - lab/cryptography
  - lab/utm
status: "draft"
---

# Phase 03: Cryptography, Deep Inspection & UTM Profiles

> **Note:** Updated for current dual-FGT topology. The old version referenced 10.0.1.x IPs — LAN1 is now 192.168.10.0/24.

## UTM Profiles on Firewall Policy

```fortinet
config firewall policy
    edit 1
        set utm-status enable
        set ssl-ssh-profile "deep-inspection"
        set av-profile "default"
        set webfilter-profile "default"
        set ips-profile "default"
        set application-list "default"
    next
end
```

## Verification Commands

```fortinet
# Sniff traffic on port1
diagnose sniffer packet port1 'port 80 or port 443' 4 10

# Flow trace for a specific client
diagnose debug reset
diagnose debug flow filter daddr 192.168.10.10
diagnose debug flow show function-name enable
diagnose debug flow trace start 50
diagnose debug enable
```
