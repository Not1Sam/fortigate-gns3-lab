---
title: "Coursera: Enterprise Firewall Administrator"
course: "Enterprise Firewall Administrator"
tags:
  - coursera/enterprise-admin
  - theory/advanced-firewall
status: "in-progress"
---

# 🎓 Enterprise Firewall Administrator Notes

## 📋 Module Tracker
- [ ] **Module 1**: Enterprise Network Topologies & WAN Architectures
- [ ] **Module 2**: Dynamic Routing Protocols (OSPF & BGP)
- [ ] **Module 3**: High Availability (HA) Clustering & FGCP
- [ ] **Module 4**: Enterprise IPSec VPNs & SD-WAN
- [ ] **Module 5**: Diagnostics, Sniffers & Troubleshooting

---

## 📝 Core Concepts

### 🚀 High Availability (HA) FGCP
> [!important] Master Election Criteria (Strict Order)
> 1.  **Monitored Ports**: Box with more active monitored interfaces wins.
> 2.  **Uptime**: Box running >5 minutes longer wins.
> 3.  **Configured Priority**: Highest value (0-255) wins.
> 4.  **Serial Number**: Highest alphabetical/numerical serial number wins.

### 🗺️ Dynamic Routing (OSPF)
*   **Administrative Distance (AD)**: Statically Connected = `0`, Static Route = `1`, OSPF = `110`, BGP = `200`.
*   **Routing Decisions**: Longest Prefix Match > AD > Metric/Priority.\n