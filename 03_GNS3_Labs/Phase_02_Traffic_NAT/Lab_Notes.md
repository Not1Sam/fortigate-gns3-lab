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
```

## 🔍 Lab Verification Protocols

### Testing Outbound Source NAT (IP Pool)

**From a LAN client VM (Alpine Webterm on port2):**

1. Generate outbound traffic to verify SNAT is working:
   ```bash
   ping 8.8.8.8
   wget -q -O- http://example.com
   ```

2. On the FortiGate, verify the session is translated using the IP Pool:
   ```bash
   diagnose firewall session list
   ```
   Expected output should show `src=192.168.122.200-210` (from the IP Pool range) instead of the LAN client's original IP.

3. Alternatively, capture traffic on the WAN interface to confirm source IP translation:
   ```bash
   diagnose sniffer packet port1 'host 8.8.8.8' 4
   ```

### Testing Inbound Destination NAT (VIP)

**From a WAN client node:**

1. Query the external VIP address from a WAN-connected device:
   ```bash
   curl http://192.168.122.150
   ```
   Expected: Response from the DMZ web server at 10.0.2.100.

2. Verify the session on the FortiGate shows proper DNAT translation:
   ```bash
   diagnose firewall session list | grep 192.168.122.150
   ```
   Expected: Session shows destination translated to `10.0.2.100:80`.

3. Test HTTP connectivity end-to-end:
   ```bash
   curl -v http://192.168.122.150
   ```
   Verify the response headers and body from the DMZ web server.\n