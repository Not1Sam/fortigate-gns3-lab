# FortiGate Lab — Topology Setup Guide

## Overview

This guide walks through building the complete dual-FortiGate hybrid-cloud lab. Two independent FGTs with a transit link, separate LAN segments, Docker service nodes, and an OCI cloud threat simulator.

### Architecture
```
NAT1 ──Switch1──┬──FGT-Primary (port1 WAN)──port2──OVS-LAN1──[PC1, webterm, App-Server, PostgreSQL]
                 │                         └port3──10.0.0.1/30──┐
                 │                                               ├──Transit (OSPF)
                 └──FGT-Secondary (port1 WAN)──port2──OVS-LAN2──[Alpine DHCP, Ubuntu, Grafana, Prometheus, Traffic-Gen]
                                              └port3──10.0.0.2/30──┘
```

### Phase Summary
| Phase | What You'll Do |
|---|---|
| 1 — Base Setup | Create GNS3 project, place all nodes, wire them |
| 2 — FGT Config | Interfaces, IPs, routes, DNS on both FGTs |
| 3 — LAN Services | DHCP (FGT-P) + Alpine DHCP (LAN2), client verification |
| 4 — Policies & NAT | LAN→WAN SNAT, inter-LAN via transit |
| 5 — Routing | OSPF over transit link |
| 6 — Docker Services | PostgreSQL, App-Server, Grafana, Prometheus, Traffic-Gen |
| 7 — Security Profiles | AV, IPS, App Control, Web Filter, SSL Inspection |
| 8 — VPN | IPsec Site-to-Site to OCI, SSL VPN portal |
| 9 — OCI Cloud | Libreswan + threat simulator deployment |
| 10 — Logging & Demo | Syslog → Grafana, end-to-end scenarios |

---

## Phase 1: Base Setup

### 1.1 GNS3 Project
Create a new project in GNS3, or open an existing one.

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

### 1.3 Wiring
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

### 1.4 OVS Bridge Setup
After starting OpenvSwitch-1 and OpenvSwitch-2:
```bash
# On OVS-LAN1
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone

# On OVS-LAN2
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4 eth5; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone
```

### 1.5 Ubuntu Base Image (one-time)
If using the modified Ubuntu image: user `ubuntu` / password `gns3`, NOPASSWD sudo.

### 1.6 Validate
- All 15 nodes visible and green in GNS3
- OVS bridges show all ports with `ovs-vsctl show`
- NAT1 has internet (confirm with ping)

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
Build custom image:
```bash
docker build -t alpine-dhcp:latest - << 'EOF'
FROM alpine:latest
RUN apk add --no-cache dnsmasq
CMD ["sh", "-c", "echo 'interface=eth0' > /etc/dnsmasq.conf && echo 'dhcp-range=192.168.20.100,192.168.20.200,12h' >> /etc/dnsmasq.conf && echo 'dhcp-option=3,192.168.20.1' >> /etc/dnsmasq.conf && echo 'dhcp-option=6,8.8.8.8' >> /etc/dnsmasq.conf && ip addr add 192.168.20.2/24 dev eth0 && ip link set eth0 up && ip route add default via 192.168.20.1 && dnsmasq --no-daemon"]
EOF
```

Set in GNS3 Docker template:
- Image: `alpine-dhcp:latest`
- Start command: *(none — uses CMD from Dockerfile)*

### 3.3 Verify Clients
- **PC1 (VPCS)**: enter `dhcp`, then `show ip` — should get 192.168.10.x
- **webterm-1**: auto DHCP from FGT-Primary — check console
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

### 4.5 DNAT (WAN → App-Server) — optional
On FGT-Primary:
```
config firewall vip
    edit "web-server"
        set extip 192.168.122.x   # FGT-Primary WAN IP
        set mappedip 192.168.10.10
        set extintf port1
        set portforward enable
        set extport 8080
        set mappedport 80
    next
end
config firewall policy
    edit 3
        set name "WAN-to-AppServer"
        set srcintf port1
        set dstintf port2
        set srcaddr all
        set dstaddr web-server
        set action accept
        set schedule always
        set service HTTP
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
    end
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
    end
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

### 5.3 Verify OSPF
```
get router info ospf neighbor
get router info ospf route
execute ping 10.0.0.2 (from FGT-Primary)
execute ping 192.168.20.1 (from FGT-Primary — inter-LAN)
```

### 5.4 Inter-LAN Policy (FGT-Primary — allow transit→LAN1)
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

### 5.5 Inter-LAN Policy (FGT-Secondary — allow transit→LAN2)
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

---

## Phase 6: Docker Services

### 6.1 PostgreSQL
GNS3 template: `postgres:16-alpine`, 1 adapter.
Environment: `POSTGRES_PASSWORD=gns3`

After starting, verify:
```bash
docker exec GNS3.PostgreSQL-1.* ip addr add 192.168.10.11/24 dev eth0
docker exec GNS3.PostgreSQL-1.* ip link set eth0 up
docker exec GNS3.PostgreSQL-1.* ip route add default via 192.168.10.1
```

Create database for app:
```
docker exec GNS3.PostgreSQL-1.* psql -U postgres -c "CREATE DATABASE appdb;"
```

### 6.2 App-Server (Flask)
GNS3 template: `python:3.12-alpine`, 1 adapter.

After starting, configure IP:
```bash
docker exec GNS3.appServer-1.* ip addr add 192.168.10.10/24 dev eth0
docker exec GNS3.appServer-1.* ip link set eth0 up
docker exec GNS3.appServer-1.* ip route add default via 192.168.10.1
```

Deploy Flask app:
```bash
docker exec GNS3.appServer-1.* sh -c "pip install flask psycopg2-binary"
docker exec GNS3.appServer-1.* sh -c "cat > /app.py << 'EOF'
from flask import Flask
app = Flask(__name__)
@app.route('/')
def hello():
    return 'Hello from App-Server!'
@app.route('/db')
def db_check():
    import psycopg2
    try:
        conn = psycopg2.connect(host='192.168.10.11', dbname='appdb', user='postgres', password='gns3')
        return 'DB OK'
    except Exception as e:
        return f'DB Error: {e}'
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
EOF"
docker exec -d GNS3.appServer-1.* python /app.py
```

### 6.3 Grafana & Prometheus
GNS3 templates: `grafana/grafana:latest` and `prom/prometheus:latest`, 1 adapter each.

After starting, configure IPs:
```bash
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
GNS3 template: `alpine:latest`, 1 adapter.

```bash
docker exec GNS3.Traffic-Gen-1.* ip addr add 192.168.20.12/24 dev eth0
docker exec GNS3.Traffic-Gen-1.* ip link set eth0 up
docker exec GNS3.Traffic-Gen-1.* ip route add default via 192.168.20.1
```

Install tools:
```bash
docker exec GNS3.Traffic-Gen-1.* apk add --no-cache curl busybox-extras
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

### 7.4 Application Control Profile
```
config application list
    edit "default-app"
        config entries
            edit 1
                set application 15892 15953 16294  # Torrent apps
                set action block
            next
        end
    next
end
```

### 7.5 SSL Inspection Profile
```
config firewall ssl-ssh-profile
    edit "deep-inspection"
        set ssl-inspection deep-inspection
    next
end
```

### 7.6 Apply to Policy (FGT-Primary)
```
config firewall policy
    edit 1
        set groups "default-av" "default-ips" "default-wf" "default-app"
        set ssl-ssh-profile "deep-inspection"
    next
end
```
(Repeat on FGT-Secondary policy 1)

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

### 9.1 Deploy OCI Instance
- Ubuntu 24.04, public IP, security group allowing IPsec (UDP 500, 4500) + HTTP/HTTPS from FGT WAN IPs

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
    ike=aes128-sha1-modp1024
    phase2=aes128-sha1
    auto=start
EOF

sudo ipsec restart
```

### 9.3 Threat Simulator
```bash
sudo apt install -y python3-flask socat
# Deploy the threat simulator app (see OCI endpoints in facts.md)
sudo python3 /opt/threat-sim/app.py &
```

---

## Phase 10: Logging & Demo

### 10.1 Syslog Forwarding (FGTs → Traffic-Gen)
```
config log syslogd setting
    set status enable
    set server 192.168.20.12
    set port 514
    set facility local7
end
```

### 10.2 Verify Logs
```bash
docker exec GNS3.Traffic-Gen-1.* nc -ulvp 514
```

### 10.3 Demo Scenarios
1. **Internet access** — PC1 / Ubuntu ping 8.8.8.8
2. **SNAT** — traffic-gen `curl ifconfig.me` shows FGT WAN IP
3. **Inter-LAN** — PC1 pings Ubuntu across transit link
4. **AV block** — curl OCI `/eicar` shows block page
5. **IPS block** — curl OCI `/attack?sql=payload` shows block
6. **Web filter** — curl OCI `/phishing` shows block
7. **IPsec VPN** — verify tunnel status `get vpn ipsec tunnel details`
8. **SSL VPN** — connect from Ubuntu over SSL VPN portal
9. **App-Server** — curl `http://192.168.10.10/` from webterm
10. **Grafana** — `http://192.168.20.10:3000` dashboards
