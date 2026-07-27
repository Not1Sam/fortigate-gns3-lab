# FortiGate Lab — Setup Guide (Linux)

> For Linux hosts running GNS3 natively with QEMU/KVM.

## Prerequisites

### Required Software
- GNS3 Server + GUI (install via package manager or gns3.com)
- QEMU/KVM with libvirt (`virt-manager`, `libvirtd`)
- Docker Engine (or Podman with Docker compat)
- Telnet client (`telnet` package)
- Git

### Host Environment
- Arch Linux, Ubuntu, Fedora, Debian, etc.
- At least 8 GB RAM free (~7.3 GB needed for the lab)
- CPU with virtualization extensions (VT-x/AMD-V)

### GNS3 Service Management
Some Linux distros use a `gns3-control` script; otherwise manage manually:
```bash
# Check GNS3 services
systemctl status gns3
systemctl status libvirtd
systemctl status docker

# Enable NAT forwarding for Cloud node internet access
sudo iptables -t nat -A POSTROUTING -s 192.168.122.0/24 ! -d 192.168.122.0/24 -j MASQUERADE
sudo iptables -A FORWARD -s 192.168.122.0/24 -j ACCEPT
```

## Architecture

```
NAT1 ──Switch1──┬──FGT-Primary (port1 WAN)──port2──OVS-LAN1──[PC1, webterm, App-Server, PostgreSQL]
                 │                         └port3──10.0.0.1/30──┐
                 │                                               ├──Transit (OSPF)
                 └──FGT-Secondary (port1 WAN)──port2──OVS-LAN2──[Alpine DHCP, Ubuntu, Grafana, Prometheus, Traffic-Gen]
                                              └port3──10.0.0.2/30──┘
```

## Phase 1: Base Setup

### 1.1 GNS3 Project
Open GNS3 → File → New Project → name it.

### 1.2 Nodes to Add
| Node | Type | Image | Adapters |
|---|---|---|---|
| FGT-Primary | QEMU | `fgt-v7.4.12.qcow2` | 8 |
| FGT-Secondary | QEMU | Linked clone | 8 |
| Ubuntu-Desktop-Client-1 | QEMU | `ubuntu-24.04-minimal-cloudimg` (mod) | 1 |
| OpenvSwitch-1 | Docker | `gns3/openvswitch:latest` | 16 |
| OpenvSwitch-2 | Docker | `gns3/openvswitch:latest` | 16 |
| webterm-1 | Docker | `gns3/webterm:latest` | 1 |
| Alpine-DHCP | Docker | `alpine:latest` | 1 |
| appServer-1 | Docker | `python:3.12-alpine` | 1 |
| PostgreSQL-1 | Docker | `postgres:16-alpine` | 1 |
| Grafana-1 | Docker | `grafana/grafana:latest` | 1 |
| Prometheus-1 | Docker | `prom/prometheus:latest` | 1 |
| Traffic-Gen-1 | Docker | `alpine:latest` | 1 |
| PC1 | VPCS | — | — |
| NAT1 | Cloud (virbr0) | — | — |
| Switch1 | Ethernet switch | — | 4 |

> FGT eval limits: 3 interfaces, 3 policies, 3 routes per FGT.

### 1.3 Cloud Node (NAT1)
On Linux, the Cloud node uses `virbr0` (libvirt default NAT network):
- Template: `NAT1` → type `Cloud`
- Interface: `virbr0` (the libvirt bridge, IP 192.168.122.1)
- No additional host network config needed

### 1.4 Wiring
**WAN Segment:**
- NAT1 a0p0 ↔ Switch1 a0p0
- Switch1 a0p1 ↔ FGT-Primary a0p0 (port1)
- Switch1 a0p2 ↔ FGT-Secondary a0p0 (port1)

**LAN1 (OVS-LAN1 = OpenvSwitch-1):**
- FGT-Primary a1p0 (port2) ↔ OpenvSwitch-1 a0p0 (eth0)
- OpenvSwitch-1 a1p0 (eth1) ↔ PC1 a0p0
- OpenvSwitch-1 a2p0 (eth2) ↔ webterm-1 a0p0
- OpenvSwitch-1 a3p0 (eth3) ↔ appServer-1 a0p0
- OpenvSwitch-1 a4p0 (eth4) ↔ PostgreSQL-1 a0p0

**LAN2 (OVS-LAN2 = OpenvSwitch-2):**
- FGT-Secondary a1p0 (port2) ↔ OpenvSwitch-2 a0p0 (eth0)
- OpenvSwitch-2 a1p0 (eth1) ↔ Alpine-DHCP a0p0
- OpenvSwitch-2 a2p0 (eth2) ↔ Ubuntu-Desktop-Client a0p0
- OpenvSwitch-2 a3p0 (eth3) ↔ Traffic-Gen-1 a0p0
- OpenvSwitch-2 a4p0 (eth4) ↔ Grafana-1 a0p0
- OpenvSwitch-2 a5p0 (eth5) ↔ Prometheus-1 a0p0

**Transit Link:**
- FGT-Primary a2p0 (port3) ↔ FGT-Secondary a2p0 (port3)

### 1.5 OVS Bridge Setup
After starting the OVS Docker nodes, connect via Telnet console and run:
```bash
# On OVS-LAN1 (OpenvSwitch-1)
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone

# On OVS-LAN2 (OpenvSwitch-2)
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4 eth5; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone
```

### 1.6 Ubuntu Base Image (one-time)
If using the modified Ubuntu image: user `ubuntu` / password `gns3`, NOPASSWD sudo.

### 1.7 Validate
- All 15 nodes visible and green in GNS3
- OVS bridges show all ports with `ovs-vsctl show`
- NAT1 has internet (confirm with `execute ping 8.8.8.8` from a FGT)

---

## Phase 2: FGT Configuration

### 2.1 FGT-Primary
**Port1 — WAN (DHCP from NAT1):**
```
config system interface
    edit port1
        set mode dhcp
        set allowaccess ping https ssh
    next
end
```

**Port2 — LAN1:**
```
config system interface
    edit port2
        set mode static
        set ip 192.168.10.1 255.255.255.0
        set allowaccess ping
    next
end
```

**Port3 — Transit:**
```
config system interface
    edit port3
        set mode static
        set ip 10.0.0.1 255.255.255.252
        set allowaccess ping
    next
end
```

**Default Route:**
```
config router static
    edit 1
        set device port1
        set gateway 192.168.122.1
    next
end
```

**DNS:**
```
config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end
```

**Verify:**
```
get system interface physical
execute ping 8.8.8.8
```

### 2.2 FGT-Secondary
**Port1 — WAN (DHCP from NAT1):**
```
config system interface
    edit port1
        set mode dhcp
        set allowaccess ping https ssh
    next
end
```

**Port2 — LAN2:**
```
config system interface
    edit port2
        set mode static
        set ip 192.168.20.1 255.255.255.0
        set allowaccess ping
    next
end
```

**Port3 — Transit:**
```
config system interface
    edit port3
        set mode static
        set ip 10.0.0.2 255.255.255.252
        set allowaccess ping
    next
end
```

**Default Route:**
```
config router static
    edit 1
        set device port1
        set gateway 192.168.122.1
    next
end
```

**DNS:**
```
config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end
```

**Verify:**
```
execute ping 8.8.8.8
```

---

## Phase 3: LAN Services

### 3.1 FGT-Primary DHCP (LAN1)
```
config system dhcp server
    edit 1
        set interface port2
        set netmask 255.255.255.0
        set default-gateway 192.168.10.1
        set dns-service default
        config ip-range
            edit 1
                set start-ip 192.168.10.100
                set end-ip 192.168.10.200
            next
        end
        set lease-time 86400
    next
end
```

### 3.2 Alpine DHCP Server (LAN2)
Build the custom Docker image:
```bash
docker build -t alpine-dhcp:latest - << 'EOF'
FROM alpine:latest
RUN apk add --no-cache dnsmasq
CMD ["sh", "-c", "echo 'interface=eth0' > /etc/dnsmasq.conf && echo 'dhcp-range=192.168.20.100,192.168.20.200,12h' >> /etc/dnsmasq.conf && echo 'dhcp-option=3,192.168.20.1' >> /etc/dnsmasq.conf && echo 'dhcp-option=6,8.8.8.8' >> /etc/dnsmasq.conf && ip addr add 192.168.20.2/24 dev eth0 && ip link set eth0 up && ip route add default via 192.168.20.1 && dnsmasq --no-daemon"]
EOF
```

In GNS3, create a Docker template with image `alpine-dhcp:latest`.

### 3.3 Verify Clients
- **PC1 (VPCS)**: enter `dhcp`, then `show ip` — should get 192.168.10.x
- **webterm-1**: auto DHCP from FGT-Primary
- **Ubuntu Desktop**: auto DHCP from Alpine — `ip addr show enp2s0`
- **Internet test**: `ping 8.8.8.8` from any client

---

## Phase 4: Policies & NAT

### 4.1 FGT-Primary — LAN1 to WAN
```
config firewall policy
    edit 1
        set name "LAN1-to-WAN"
        set srcintf port2
        set dstintf port1
        set srcaddr all
        set dstaddr all
        set action accept
        set schedule always
        set service ALL
        set nat enable
    next
end
```

### 4.2 FGT-Primary — Transit to WAN
```
config firewall policy
    edit 2
        set name "Transit-to-WAN"
        set srcintf port3
        set dstintf port1
        set srcaddr all
        set dstaddr all
        set action accept
        set schedule always
        set service ALL
        set nat enable
    next
end
```

### 4.3 FGT-Secondary — LAN2 to WAN
```
config firewall policy
    edit 1
        set name "LAN2-to-WAN"
        set srcintf port2
        set dstintf port1
        set srcaddr all
        set dstaddr all
        set action accept
        set schedule always
        set service ALL
        set nat enable
    next
end
```

### 4.4 FGT-Secondary — Transit to WAN
```
config firewall policy
    edit 2
        set name "Transit-to-WAN"
        set srcintf port3
        set dstintf port1
        set srcaddr all
        set dstaddr all
        set action accept
        set schedule always
        set service ALL
        set nat enable
    next
end
```

---

## Phase 5: Routing (OSPF over Transit)

### 5.1 FGT-Primary — OSPF
```
config router ospf
    set router-id 10.0.0.1
    config area
        edit 0.0.0.0
    next
    config network
        edit 1
            set prefix 192.168.10.0 255.255.255.0
            set area 0.0.0.0
        next
        edit 2
            set prefix 10.0.0.0 255.255.255.252
            set area 0.0.0.0
        next
    end
end
```

### 5.2 FGT-Secondary — OSPF
```
config router ospf
    set router-id 10.0.0.2
    config area
        edit 0.0.0.0
    next
    config network
        edit 1
            set prefix 192.168.20.0 255.255.255.0
            set area 0.0.0.0
        next
        edit 2
            set prefix 10.0.0.0 255.255.255.252
            set area 0.0.0.0
        next
    end
end
```

### 5.3 Inter-LAN Policies
On FGT-Primary:
```
config firewall policy
    edit 3
        set name "Transit-to-LAN1"
        set srcintf port3
        set dstintf port2
        set srcaddr all
        set dstaddr all
        set action accept
        set schedule always
        set service ALL
    next
end
```

On FGT-Secondary:
```
config firewall policy
    edit 3
        set name "Transit-to-LAN2"
        set srcintf port3
        set dstintf port2
        set srcaddr all
        set dstaddr all
        set action accept
        set schedule always
        set service ALL
    next
end
```

### 5.4 Verify OSPF
```
get router info ospf neighbor
get router info ospf route
execute ping 10.0.0.2
execute ping 192.168.20.1
```

---

## Phase 6: Docker Services

### 6.1 PostgreSQL
```
docker exec GNS3.PostgreSQL-1.* ip addr add 192.168.10.11/24 dev eth0
docker exec GNS3.PostgreSQL-1.* ip link set eth0 up
docker exec GNS3.PostgreSQL-1.* ip route add default via 192.168.10.1
docker exec GNS3.PostgreSQL-1.* psql -U postgres -c "CREATE DATABASE appdb;"
```

### 6.2 App-Server (Flask)
```
docker exec GNS3.appServer-1.* ip addr add 192.168.10.10/24 dev eth0
docker exec GNS3.appServer-1.* ip link set eth0 up
docker exec GNS3.appServer-1.* ip route add default via 192.168.10.1
docker exec GNS3.appServer-1.* sh -c "pip install flask psycopg2-binary"
```

Deploy app, then run:
```
docker exec -d GNS3.appServer-1.* python /app.py
```

### 6.3 Grafana & Prometheus
```
# Grafana
docker exec GNS3.Grafana-1.* ip addr add 192.168.20.10/24 dev eth0
docker exec GNS3.Grafana-1.* ip link set eth0 up
docker exec GNS3.Grafana-1.* ip route add default via 192.168.20.1

# Prometheus
docker exec GNS3.Prometheus-1.* ip addr add 192.168.20.11/24 dev eth0
docker exec GNS3.Prometheus-1.* ip link set eth0 up
docker exec GNS3.Prometheus-1.* ip route add default via 192.168.20.1
```

### 6.4 Traffic-Gen
```
docker exec GNS3.Traffic-Gen-1.* ip addr add 192.168.20.12/24 dev eth0
docker exec GNS3.Traffic-Gen-1.* ip link set eth0 up
docker exec GNS3.Traffic-Gen-1.* ip route add default via 192.168.20.1
docker exec GNS3.Traffic-Gen-1.* apk add --no-cache curl busybox-extras
```

---

## Phase 7: Security Profiles

Apply on both FGTs:
```
config antivirus profile
    edit "default-av"
        config http
            set options av
            set av-scan all
        end
    next
end

config ips sensor
    edit "default-ips"
        config entries
            edit 1
                set severity critical high medium
                set action block
            next
        end
    next
end

config webfilter profile
    edit "default-wf"
        config static-url-filter
            set status enable
            config entries
                edit 1
                    set url "phishing"
                    set action block
                next
                edit 2
                    set url "hacking-tools"
                    set action block
                next
            end
        end
    next
end

config application list
    edit "default-app"
        config entries
            edit 1
                set application 15892 15953 16294
                set action block
            next
        end
    next
end

config firewall ssl-ssh-profile
    edit "deep-inspection"
        set ssl-inspection deep-inspection
    next
end

config firewall policy
    edit 1
        set groups "default-av" "default-ips" "default-wf" "default-app"
        set ssl-ssh-profile "deep-inspection"
    next
end
```

---

## Phase 8: VPN

### 8.1 IPsec Site-to-Site (FGT-Primary → OCI)
```
config vpn ipsec phase1-interface
    edit "to-oci"
        set interface port1
        set remote-gw <OCI-public-IP>
        set proposal aes128-sha1
        set dhgrp 2
    next
end
config vpn ipsec phase2-interface
    edit "to-oci"
        set phase1name "to-oci"
        set proposal aes128-sha1
        set src-addr-type name
        set dst-addr-type name
        set src-name "LAN1"
        set dst-name "OCI-LAN"
    next
end
```

### 8.2 SSL VPN Portal
```
config vpn ssl settings
    set port 443
    set servercert self-signed
    config authentication-rule
        edit 1
            set groups "guest"
            set portal "full-access"
        next
    end
end
config vpn ssl web portal
    edit "full-access"
        set tunnel-mode enable
        set ip-pools "ssl-vpn-pool"
    next
end
config firewall address
    edit "ssl-vpn-pool"
        set type iprange
        set start-ip 10.0.2.10
        set end-ip 10.0.2.20
    next
end
```

---

## Phase 9: OCI Cloud

Deploy Ubuntu 24.04 instance with public IP. Security group rules:
- UDP 500, 4500 from FGT WAN IPs (IPsec)
- TCP 80, 443 from FGT WAN IPs (HTTP/S for threat sim)

```bash
sudo apt update && sudo apt install -y libreswan
# Configure /etc/ipsec.conf per Device-Setup-Guide.md
sudo ipsec restart
sudo apt install -y python3-flask socat
sudo python3 /opt/threat-sim/app.py &
```

---

## Phase 10: Logging & Demo

**Syslog forwarding** (on both FGTs):
```
config log syslogd setting
    set status enable
    set server 192.168.20.12
    set port 514
    set facility local7
end
```

**Demo scenarios:**
1. Internet access from any client
2. SNAT verification (curl ifconfig.me)
3. Inter-LAN ping across transit link
4. AV block (curling /eicar)
5. IPS block (SQLi / XSS payloads)
6. Web filter block (phishing URLs)
7. IPsec tunnel status
8. SSL VPN connection
9. App-Server + PostgreSQL integration
10. Grafana dashboards
