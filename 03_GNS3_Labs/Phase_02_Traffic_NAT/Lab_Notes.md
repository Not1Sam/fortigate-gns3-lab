---
title: "Phase 02: Traffic Control & NAT"
tags:
  - lab/policy
  - lab/nat
status: "todo"
---

# 🛡️ Phase 02: Traffic Control & NAT Policies

## 🛠️ CLI Configuration Commands

### Outbound Source NAT (IP Pool)

Define a dynamic overload IP Pool for outbound LAN traffic. This allows multiple LAN clients to share a range of public IPs when accessing the Internet.

```fortinet
config firewall ippool
    edit "HQ_WAN_IP_Pool"
        set startip 192.168.122.200
        set endip 192.168.122.210
        set type overload
        set comment "Dynamic overload pool for LAN-to-Internet SNAT"
    next
end
```

### Outbound LAN to WAN Access Policy (with IP Pool SNAT)
```fortinet
config firewall policy
    edit 1
        set name "LAN_to_Internet"
        set srcintf "port2"
        set dstintf "port1"
        set action accept
        set srcaddr "all"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
        set ippool enable
        set poolname "HQ_WAN_IP_Pool"
        set nat enable
    next
end
```

### Creating Virtual IP (VIP) for Destination NAT
```fortinet
config firewall vip
    edit "DMZ_Web_Server"
        set extip 192.168.122.150
        set extport 80
        set mappedip 10.0.2.100
        set mappedport 80
        set portforward enable
    next
end
```

### Directing Incoming Traffic to DMZ Server
```fortinet
config firewall policy
    edit 2
        set name "Publish_DMZ_Web"
        set srcintf "port1"
        set dstintf "port3"
        set action accept
        set srcaddr "all"
        set dstaddr "DMZ_Web_Server"
        set schedule "always"
        set service "HTTP"
    next
end
```\n