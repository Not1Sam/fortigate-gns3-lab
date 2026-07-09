---
title: "Phase 03: Cryptography & Security Profiles"
tags:
  - lab/cryptography
  - lab/utm
status: "todo"
---

# 🔬 Phase 03: Cryptography, Deep Inspection, & UTM Profiles

## 🛠️ CLI Configuration Commands

### Active Profiles on Firewall Policy
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

---

## 🔍 Verification & Debugging Commands

### Sniffing Packets on specific port (e.g. port1)
`diagnose sniffer packet port1 'port 80 or port 443' 4 10`

### Checking flow trace for traffic diagnostics
```fortinet
diagnose debug reset
diagnose debug flow filter daddr 10.0.1.10
diagnose debug flow show function-name enable
diagnose debug flow trace start 50
diagnose debug enable
```\n