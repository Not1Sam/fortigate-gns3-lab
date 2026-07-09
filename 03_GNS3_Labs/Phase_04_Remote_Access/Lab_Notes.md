---
title: "Phase 04: VPN & Authentication"
tags:
  - lab/vpn
  - lab/auth
status: "todo"
---

# 🌐 Phase 04: Remote Access (IPSec, SSL VPN, LDAP)

## 🛠️ CLI Configuration Commands

### LDAP Directory Integration
```fortinet
config user ldap
    edit "AD_Directory"
        set server "10.0.1.250"
        set cnid "sAMAccountName"
        set dn "dc=enterprise,dc=local"
        set type regular
        set username "cn=fgtbind,cn=users,dc=enterprise,dc=local"
        set password "SecretPassword123"
    next
end
```

### SSL VPN Split Tunneling Config
```fortinet
config vpn ssl web portal
    edit "tunnel-access"
        set tunnel-mode enable
        set split-tunneling enable
        set split-tunneling-routing-address "HQ-Subnets"
    next
end
```\n