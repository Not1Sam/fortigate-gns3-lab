# FortiGate Lab — Topology Setup Guide (Modified)

## Overview

This guide walks through building the **modified** dual-FortiGate hybrid-cloud lab. Four FortiGate VMs organized in 2 HA pairs, a Docker Router for cross-LAN routing, OVS switch fabric, and an OCI cloud threat simulator.

### Architecture
```
NAT1 ──Switch1──┬──FGT-Primary (ACTIVE)────port2──OVS-LAN1──[PC1, webterm, App-Server, PostgreSQL, Docker Router eth0]
                │         └port3──HA heartbeat──┘
                │
                ├──FGT-Primary-HA (STANDBY)──port2──OVS-LAN1──(passive)
                │         └port3──HA heartbeat──┘
                │
                ├──FGT-Secondary (ACTIVE)────port2──OVS-LAN2──[Alpine DHCP, Ubuntu, Grafana, Prometheus, Traffic-Gen, Docker Router eth1]
                │         └port3──HA heartbeat──┘
                │
                └──FGT-Secondary-HA (STANDBY)──port2──OVS-LAN2──(passive)
                          └port3──HA heartbeat──┘

Docker Router: eth0=192.168.10.254/24 (LAN1) ── eth1=192.168.20.254/24 (LAN2)
```

### Key Design Principles
- **No OSPF transit** — Docker Router handles cross-LAN routing
- **port3 = HA heartbeat** — freed by removing OSPF transit
- **HA Active-Passive** — each FGT has a backup on the same LAN segment
- **SSL VPN** — uses ssl.root virtual interface (no extra port needed)
- **Eval license** — 3 interfaces, 3 policies, 3 routes per FGT

### Phase Summary
| Phase | What You'll Do |
|---|---|
| 1 — Base Setup | Create GNS3 project, place all nodes, wire them |
| 2 — FGT Config | Interfaces, IPs, routes, DNS on all 4 FGTs |
| 3 — HA Setup | Configure HA clusters on both pairs |
| 4 — Docker Router | Cross-LAN routing between LAN1 and LAN2 |
| 5 — LAN Services | DHCP, Docker services, clients |
| 6 — Policies & NAT | LAN→WAN SNAT, inter-LAN via Docker Router |
| 7 — Security Profiles | AV, IPS, Web Filter, SSL Inspection |
| 8 — VPN | IPsec Site-to-Site to OCI, SSL VPN portal |
| 9 — OCI Cloud | Libreswan + threat simulator deployment |
| 10 — Logging & Demo | Syslog → Traffic-Gen, end-to-end scenarios |

---

## Phase 1: Base Setup

### 1.1 GNS3 Project
Create a new project in GNS3 called `fortigate-lab`.

### 1.2 Nodes to Add

**QEMU VMs (5):**
| Node | Type | Image | RAM | vCPU | Adapters | Console |
|---|---|---|---|---|---|---|
| FGT-Primary | QEMU | `fgt-v7.4.12.qcow2` | 2048 MB | 1 | 8 | Telnet |
| FGT-Primary-HA | QEMU | `fortios.qcow2` (7.0.9 pre-licensed) | 2048 MB | 1 | 8 | Telnet |
| FGT-Secondary | QEMU | `fgt-v7.4.12.qcow2` | 2048 MB | 1 | 8 | Telnet |
| FGT-Secondary-HA | QEMU | `fortios.qcow2` (7.0.9 pre-licensed) | 2048 MB | 1 | 8 | Telnet |
| Ubuntu Desktop | QEMU | `ubuntu-24.04-minimal-cloudimg` (mod) | 2048 MB | 2 | 1 | VNC |

**Docker Nodes (9):**
| Node | Image | Adapters | Console |
|---|---|---|---|
| OVS-LAN1 | `gns3/openvswitch:latest` | 16 | Telnet |
| OVS-LAN2 | `gns3/openvswitch:latest` | 16 | Telnet |
| Docker Router | `frrouting/frr:latest` | 2 | Telnet |
| webterm-1 | `gns3/webterm:latest` | 1 | VNC |
| Alpine-DHCP | `alpine-dhcp:latest` (custom) | 1 | Telnet |
| appServer-1 | `python:3.12-alpine` | 1 | Telnet |
| PostgreSQL-1 | `postgres:16-alpine` | 1 | Telnet |
| Grafana-1 | `grafana/grafana:latest` | 1 | VNC |
| Prometheus-1 | `prom/prometheus:latest` | 1 | VNC |
| Traffic-Gen-1 | `alpine:latest` | 1 | Telnet |

**Built-in Nodes (3):**
| Node | Type | Role |
|---|---|---|
| PC1 | VPCS | Client CLI on LAN1 |
| NAT1 | Cloud (virbr0) | WAN gateway |
| Switch1 | Ethernet switch | WAN distribution |

> **Total: 18 nodes** (5 QEMU + 9 Docker + 3 Built-in + 1 Docker Router = 14 unique + 2 HA + 1 Router)

### 1.3 Wiring

**WAN Segment:**
```
NAT1 a0p0 ── Switch1 a0p0
Switch1 a0p1 ── FGT-Primary a0p0 (port1)
Switch1 a0p2 ── FGT-Secondary a0p0 (port1)
Switch1 a0p3 ── FGT-Primary-HA a0p0 (port1)
Switch1 a0p4 ── FGT-Secondary-HA a0p0 (port1)
```

**HA Heartbeat (port3 to port3):**
```
FGT-Primary a2p0 (port3) ── FGT-Primary-HA a2p0 (port3)
FGT-Secondary a2p0 (port3) ── FGT-Secondary-HA a2p0 (port3)
```

**LAN1 (OVS-LAN1):**
```
FGT-Primary a1p0 (port2) ── OVS-LAN1 a0p0 (eth0)
FGT-Primary-HA a1p0 (port2) ── OVS-LAN1 a1p0 (eth1)   [passive]
OVS-LAN1 eth2 ── PC1 a0p0
OVS-LAN1 eth3 ── webterm-1 a0p0
OVS-LAN1 eth4 ── appServer-1 a0p0
OVS-LAN1 eth5 ── PostgreSQL-1 a0p0
OVS-LAN1 eth6 ── Docker Router a0p0 (eth0)
```

**LAN2 (OVS-LAN2):**
```
FGT-Secondary a1p0 (port2) ── OVS-LAN2 a0p0 (eth0)
FGT-Secondary-HA a1p0 (port2) ── OVS-LAN2 a1p0 (eth1)  [passive]
OVS-LAN2 eth2 ── Alpine-DHCP a0p0
OVS-LAN2 eth3 ── Ubuntu Desktop a0p0
OVS-LAN2 eth4 ── Traffic-Gen-1 a0p0
OVS-LAN2 eth5 ── Grafana-1 a0p0
OVS-LAN2 eth6 ── Prometheus-1 a0p0
OVS-LAN2 eth7 ── Docker Router a1p0 (eth1)
```

### 1.4 OVS Bridge Setup
After starting OVS-LAN1 and OVS-LAN2:
```bash
# On OVS-LAN1
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4 eth5 eth6; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone

# On OVS-LAN2
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4 eth5 eth6 eth7; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone
```

### 1.5 Validate
- All nodes visible in GNS3
- OVS bridges show all ports with `ovs-vsctl show`
- NAT1 has internet (confirm with ping from host)

---

## Phase 2: FGT Configuration (All 4 FGTs)

### 2.1 FGT-Primary

**Port1 — WAN (DHCP from NAT1):**
```
config system interface
    edit "port1"
        set mode dhcp
        set allowaccess ping https ssh
    next
end
```

**Port2 — LAN1:**
```
config system interface
    edit "port2"
        set ip 192.168.10.1 255.255.255.0
        set allowaccess ping https ssh
    next
end
```

**Port3 — HA Heartbeat:**
```
config system interface
    edit "port3"
        set ip 169.254.0.1 255.255.255.252
        set allowaccess ping
    next
end
```

**Default Route:**
```
config router static
    edit 1
        set device "port1"
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

### 2.2 FGT-Primary-HA

**Port1 — WAN (DHCP from NAT1):**
```
config system interface
    edit "port1"
        set mode dhcp
        set allowaccess ping https ssh
    next
end
```

**Port2 — LAN1 (passive):**
```
config system interface
    edit "port2"
        set ip 192.168.10.2 255.255.255.0
        set allowaccess ping
    next
end
```

**Port3 — HA Heartbeat:**
```
config system interface
    edit "port3"
        set ip 169.254.0.2 255.255.255.252
        set allowaccess ping
    next
end
```

**Default Route:**
```
config router static
    edit 1
        set device "port1"
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

### 2.3 FGT-Secondary

**Port1 — WAN (DHCP from NAT1):**
```
config system interface
    edit "port1"
        set mode dhcp
        set allowaccess ping https ssh
    next
end
```

**Port2 — LAN2:**
```
config system interface
    edit "port2"
        set ip 192.168.20.1 255.255.255.0
        set allowaccess ping https ssh
    next
end
```

**Port3 — HA Heartbeat:**
```
config system interface
    edit "port3"
        set ip 169.254.0.3 255.255.255.252
        set allowaccess ping
    next
end
```

**Default Route:**
```
config router static
    edit 1
        set device "port1"
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

### 2.4 FGT-Secondary-HA

**Port1 — WAN (DHCP from NAT1):**
```
config system interface
    edit "port1"
        set mode dhcp
        set allowaccess ping https ssh
    next
end
```

**Port2 — LAN2 (passive):**
```
config system interface
    edit "port2"
        set ip 192.168.20.2 255.255.255.0
        set allowaccess ping
    next
end
```

**Port3 — HA Heartbeat:**
```
config system interface
    edit "port3"
        set ip 169.254.0.4 255.255.255.252
        set allowaccess ping
    next
end
```

**Default Route:**
```
config router static
    edit 1
        set device "port1"
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

---

## Phase 3: HA Setup

### 3.1 Cluster 1 — FGT-Primary + FGT-Primary-HA

**FGT-Primary (priority 200 = master):**
```
config system ha
    set group-name "FortiLab-HA"
    set mode a-p
    set hbdev "port3"
    set priority 200
    set session-sync-dev "port3"
end
```

**FGT-Primary-HA (priority 100 = backup):**
```
config system ha
    set group-name "FortiLab-HA"
    set mode a-p
    set hbdev "port3"
    set priority 100
    set session-sync-dev "port3"
end
```

**Verify:**
```
get system ha status
```
Expected: FGT-Primary = MASTER, FGT-Primary-HA = BACKUP

### 3.2 Cluster 2 — FGT-Secondary + FGT-Secondary-HA

**FGT-Secondary (priority 200 = master):**
```
config system ha
    set group-name "FortiLab-HA2"
    set mode a-p
    set hbdev "port3"
    set priority 200
    set session-sync-dev "port3"
end
```

**FGT-Secondary-HA (priority 100 = backup):**
```
config system ha
    set group-name "FortiLab-HA2"
    set mode a-p
    set hbdev "port3"
    set priority 100
    set session-sync-dev "port3"
end
```

**Verify:**
```
get system ha status
```
Expected: FGT-Secondary = MASTER, FGT-Secondary-HA = BACKUP

---

## Phase 4: Docker Router (Cross-LAN Routing)

### 4.1 Build the Docker Router Image

```bash
docker build -t docker-router:latest - << 'EOF'
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y iproute2 iptables && rm -rf /var/lib/apt/lists/*
CMD ["sh", "-c", "\
    ip addr add 192.168.10.254/24 dev eth0 && \
    ip addr add 192.168.20.254/24 dev eth1 && \
    ip link set eth0 up && \
    ip link set eth1 up && \
    echo 1 > /proc/sys/net/ipv4/ip_forward && \
    echo 'nameserver 8.8.8.8' > /etc/resolv.conf && \
    tail -f /dev/null"]
EOF
```

### 4.2 GNS3 Docker Template
- Image: `docker-router:latest`
- Adapters: 2
- Console: Telnet

### 4.3 Verify
```bash
# Console into Docker Router
ip addr show eth0   # Should show 192.168.10.254/24
ip addr show eth1   # Should show 192.168.20.254/24
cat /proc/sys/net/ipv4/ip_forward  # Should show 1

# Test routing
ping 192.168.10.1   # Should reach FGT-Primary LAN
ping 192.168.20.1   # Should reach FGT-Secondary LAN
```

---

## Phase 5: LAN Services

### 5.1 FGT-Primary DHCP (LAN1)
```
config system dhcp server
    edit 1
        set interface "port2"
        set default-gateway 192.168.10.1
        set netmask 255.255.255.0
        config ip-range
            edit 1
                set start-ip 192.168.10.100
                set end-ip 192.168.10.200
            next
        end
        set dns-service default
    next
end
```

### 5.2 Alpine DHCP Server (LAN2)
Build custom image:
```bash
docker build -t alpine-dhcp:latest - << 'EOF'
FROM alpine:latest
RUN apk add --no-cache dnsmasq
CMD ["sh", "-c", "\
    echo 'interface=eth0' > /etc/dnsmasq.conf && \
    echo 'dhcp-range=192.168.20.100,192.168.20.200,12h' >> /etc/dnsmasq.conf && \
    echo 'dhcp-option=3,192.168.20.1' >> /etc/dnsmasq.conf && \
    echo 'dhcp-option=6,8.8.8.8' >> /etc/dnsmasq.conf && \
    ip addr add 192.168.20.2/24 dev eth0 && \
    ip link set eth0 up && \
    ip route add default via 192.168.20.1 && \
    dnsmasq --no-daemon"]
EOF
```

### 5.3 PostgreSQL
```
# IP setup
ip addr add 192.168.10.11/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.10.1

# Create database
su - postgres -c "pg_ctl -D /var/lib/postgresql/data start"
su - postgres -c "psql -c 'CREATE DATABASE labdb;'"
su - postgres -c "psql -c \"CREATE USER labuser WITH PASSWORD 'gns3lab';\""
su - postgres -c "psql -c 'GRANT ALL PRIVILEGES ON DATABASE labdb TO labuser;'"
```

### 5.4 App Server (Flask)
```
# IP setup
ip addr add 192.168.10.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.10.1
echo nameserver 8.8.8.8 > /etc/resolv.conf

# Install Flask
pip install flask psycopg2-binary requests --break-system-packages

# Deploy app (see Docker-Services-Guide.md for full code)
python3 /opt/app.py &
```

### 5.5 Grafana & Prometheus
```
# Grafana (192.168.20.11:3000)
ip addr add 192.168.20.11/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1

# Prometheus (192.168.20.12:9090)
ip addr add 192.168.20.12/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
```

### 5.6 Traffic-Gen
```
ip addr add 192.168.20.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
apk add --no-cache curl busybox-extras
```

### 5.7 Verify Clients
- **PC1**: `dhcp` then `show ip` — should get 192.168.10.x
- **Ubuntu**: `dhcpcd enp2s0` — should get 192.168.20.x
- **Internet**: `ping 8.8.8.8` from any client

---

## Phase 6: Policies & NAT

### 6.1 FGT-Primary — LAN1 to WAN
```
config firewall policy
    edit 1
        set name "LAN1-to-WAN"
        set srcintf "port2"
        set dstintf "port1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "HTTP" "HTTPS" "DNS" "PING"
        set logtraffic all
        set nat enable
    next
end
```

### 6.2 FGT-Secondary — LAN2 to WAN
```
config firewall policy
    edit 2
        set name "LAN2-to-WAN"
        set srcintf "port2"
        set dstintf "port1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "HTTP" "HTTPS" "DNS" "PING"
        set logtraffic all
        set nat enable
    next
end
```

### 6.3 Verify
```
# From PC1: ping 8.8.8.8
# From Ubuntu: ping 8.8.8.8
# Both should work through their respective FGTs
```

---

## Phase 7: Security Profiles

### 7.1 Antivirus Profile
```
config antivirus profile
    edit "default-av"
        config http
            set options av
            set av-scan all
        end
    next
end
```

### 7.2 IPS Profile
```
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
```

### 7.3 Web Filter Profile
```
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
```

### 7.4 Apply to Policy (FGT-Primary)
```
config firewall policy
    edit 1
        set groups "default-av" "default-ips" "default-wf"
        set ssl-ssh-profile "deep-inspection"
    next
end
```
(Repeat on FGT-Secondary policy 2)

---

## Phase 8: VPN

### 8.1 IPsec Site-to-Site (FGT-Primary → OCI)
```
config vpn ipsec phase1-interface
    edit "phase1-to-oci"
        set interface "port1"
        set ike-version 2
        set remote-gw <OCI-public-IP>
        set proposal aes256-sha256
        set dhgrp 14
        set psksecret "SharedSecret123"
    next
end

config vpn ipsec phase2-interface
    edit "phase2-to-oci"
        set phase1name "phase1-to-oci"
        set proposal aes256-sha256
        set dhgrp 14
        set src-addr-type subnet
        set src-subnet 192.168.10.0 255.255.255.0
        set dst-addr-type subnet
        set dst-subnet 10.0.1.0 255.255.255.0
    next
end
```

### 8.2 SSL VPN Portal
```
config vpn ssl settings
    set servercert "cert_vpn"
    set tunnel-ip-pools "sslvpn_tunnel_pool"
    set port 10443
    set source-interface "port1"
    set source-address "all"
    set default-portal "full-access"
end

config vpn ssl web portal
    edit "full-access"
        set tunnel-mode enable
        set ip-pools "sslvpn_tunnel_pool"
    next
end
```

---

## Phase 9: OCI Cloud

### 9.1 Deploy OCI Instance
- Ubuntu 24.04, A1.Flex (1 OCPU, 6 GB RAM)
- Public IP required
- Security group: allow UDP 500, 4500 (IPsec) + TCP 80, 443 from FGT WAN IPs

### 9.2 Libreswan IPsec
```bash
sudo apt update && sudo apt install -y libreswan

sudo tee /etc/ipsec.conf << 'EOF'
conn fgt-primary
    left=%defaultroute
    leftid=<OCI-public-IP>
    leftsubnet=<OCI-VPC-subnet>
    right=<FGT-Primary-WAN-IP>
    rightsubnet=192.168.10.0/24
    ikelifetime=24h
    lifetime=8h
    ike=aes256-sha2_256;modp2048
    phase2=aes256-sha2_256
    auto=start
EOF

sudo ipsec restart
```

### 9.3 Threat Simulator
```bash
sudo apt install -y python3-flask socat
# Deploy threat simulator app (see endpoints in Full-Topology-Spec.md)
sudo python3 /opt/threat-sim/app.py &
```

---

## Phase 10: Logging & Demo

### 10.1 Syslog Forwarding
```
config log syslogd setting
    set status enable
    set server 192.168.20.10
    set port 514
    set facility local7
end
```

### 10.2 Demo Scenarios
1. **Internet access** — PC1 / Ubuntu ping 8.8.8.8
2. **SNAT** — `curl ifconfig.me` shows FGT WAN IP
3. **Inter-LAN** — PC1 pings Ubuntu via Docker Router
4. **HA failover** — disconnect port3, verify master switches
5. **AV block** — curl OCI `/eicar` shows block page
6. **IPS block** — curl OCI `/attack?sql=payload` shows block
7. **Web filter** — curl OCI `/phishing` shows block
8. **IPsec VPN** — verify tunnel `get vpn ipsec tunnel details`
9. **SSL VPN** — connect from Ubuntu over SSL VPN portal
10. **App-Server** — `curl http://192.168.10.10/` from webterm
11. **Grafana** — `http://192.168.20.11:3000` dashboards
