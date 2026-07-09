---
title: "Phase 05: Visibility & Hardening"
tags:
  - lab/analyzer
  - lab/hardening
status: "todo"
---

# 📊 Phase 05: Visibility, FortiAnalyzer & Hardening

## 🛠️ CLI Configuration Commands

### Connecting FortiGate to FortiAnalyzer
```fortinet
config log fortianalyzer setting
    set status enable
    set server "192.168.122.200"
    set enc-algorithm high
end
```

### Administrative Hardening (Trusted Hosts)
```fortinet
config system admin
    edit "admin"
        set trusthost1 10.0.1.0 255.255.255.0
        set trusthost2 192.168.122.0 255.255.255.0
    next
end
```

### Disabling HTTP/Telnet & Changing Ports
```fortinet
config system global
    set admin-sport 10443
    set admin-ssh-port 10022
end
```\n