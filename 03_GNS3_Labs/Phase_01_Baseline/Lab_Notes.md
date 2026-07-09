---
title: "Phase 01: Setup Baseline & HA"
tags:
  - lab/baseline
  - lab/ha
status: "in-progress"
---

# 📐 Phase 01: Lab Setup & High Availability

## 🔌 IP Addressing Plan
| Interface | Connection | IP / Method | Role |
| :--- | :--- | :--- | :--- |
| **port1** | NAT Cloud | DHCP (`192.168.122.X`) | WAN (Internet Access) |
| **port2** | HQ LAN | `10.0.1.1/24` (Static) | Local LAN Gateway |
| **port3** | HQ DMZ | `10.0.2.1/24` (Static) | DMZ Gateway |
| **port4** | HA Link | `10.254.254.1/30` | HA Heartbeat Link |

---

## 🛠️ CLI Configuration Commands

### Initializing WAN, LAN, and Static Route
```fortinet
config system interface
    edit port1
        set mode dhcp
        set allowaccess ping
    next
    edit port2
        set mode static
        set ip 10.0.1.1 255.255.255.0
        set allowaccess ping https ssh
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

### HA Clustering Setup (Active-Passive)
```fortinet
config system ha
    set group-id 10
    set group-name "HQ-Cluster"
    set mode a-p
    set hbdev port4 50
    set priority 200  # Set to 100 on the second/slave box
    set monitor port1 port2
end
```

---

## 🔍 Verification Commands
*   Verify HA cluster synchronization status:
    `get system ha status`\n