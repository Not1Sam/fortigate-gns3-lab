# Device Setup Guide (Modified Topology)

Per-node setup instructions for every device in the modified topology (4 FGTs, Docker Router, HA).

---

## 1. NAT1 (Cloud)

Provides WAN internet access for the lab.

**Linux:** Uses `virbr0` (libvirt default NAT bridge, 192.168.122.1).
1. Right-click NAT1 → Configure → check `virbr0`
2. Verify: `ip addr show virbr0` → should show `192.168.122.1/24`

**Windows (GNS3 VM):** Uses `VMnet8` (VMware NAT adapter).
1. Find the adapter: run `ipconfig` on Windows host, look for VMware VMnet8
2. Right-click NAT1 → Configure → check that adapter

---

## 2. Switch1 (Ethernet Switch)

No configuration needed — plug-and-play L2 switch.

**Wiring:**
| Switch Port | Connected To |
|---|---|
| a0p0 | NAT1 |
| a0p1 | FGT-Primary port1 |
| a0p2 | FGT-Secondary port1 |
| a0p3 | FGT-Primary-HA port1 |
| a0p4 | FGT-Secondary-HA port1 |

---

## 3. FGT-Primary

**Template:** QEMU, `fgt-v7.4.12.qcow2`, 2048 MB RAM, 1 vCPU, 8 adapters
**Console:** Telnet
**Login:** `admin` / no password

### 3.1 Interfaces
```
config system interface
    edit "port1"
        set mode dhcp
        set allowaccess ping https ssh
    next
    edit "port2"
        set ip 192.168.10.1 255.255.255.0
        set allowaccess ping https ssh
    next
    edit "port3"
        set ip 169.254.0.1 255.255.255.252
        set allowaccess ping
    next
end
```

### 3.2 Default Route
```
config router static
    edit 1
        set device "port1"
        set gateway 192.168.122.1
    next
end
```

### 3.3 DNS
```
config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end
```

### 3.4 DHCP Server (LAN1)
> **Important**: Add `set vci-match disable` if clients don't get IPs — FortiGate blocks non-FortiSwitch by default.

```
config system dhcp server
    edit 1
        set interface "port2"
        set default-gateway 192.168.10.1
        set netmask 255.255.255.0
        set dns-service default
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

### 3.5 Firewall Policy (LAN1 → WAN)
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

### 3.6 HA (Master, priority 200)
```
config system ha
    set group-name "FortiLab-HA"
    set mode a-p
    set hbdev "port3"
    set priority 200
    set session-sync-dev "port3"
end
```

### 3.7 Verify
```
get system interface physical
execute ping 8.8.8.8
get system ha status
```

---

## 4. FGT-Primary-HA

**Template:** QEMU, `fortios.qcow2` (7.0.9 pre-licensed), 2048 MB RAM, 1 vCPU, 8 adapters
**Console:** Telnet
**Login:** `admin` / no password

### 4.1 Interfaces
```
config system interface
    edit "port1"
        set mode dhcp
        set allowaccess ping https ssh
    next
    edit "port2"
        set ip 192.168.10.2 255.255.255.0
        set allowaccess ping
    next
    edit "port3"
        set ip 169.254.0.2 255.255.255.252
        set allowaccess ping
    next
end
```

### 4.2 Default Route
```
config router static
    edit 1
        set device "port1"
        set gateway 192.168.122.1
    next
end
```

### 4.3 DNS
```
config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end
```

### 4.4 HA (Backup, priority 100)
```
config system ha
    set group-name "FortiLab-HA"
    set mode a-p
    set hbdev "port3"
    set priority 100
    set session-sync-dev "port3"
end
```

### 4.5 Verify
```
get system ha status
# Should show BACKUP
```

---

## 5. FGT-Secondary

**Template:** QEMU, `fgt-v7.4.12.qcow2`, 2048 MB RAM, 1 vCPU, 8 adapters
**Console:** Telnet
**Login:** `admin` / no password

### 5.1 Interfaces
```
config system interface
    edit "port1"
        set mode dhcp
        set allowaccess ping https ssh
    next
    edit "port2"
        set ip 192.168.20.1 255.255.255.0
        set allowaccess ping https ssh
    next
    edit "port3"
        set ip 169.254.0.3 255.255.255.252
        set allowaccess ping
    next
end
```

### 5.2 Default Route
```
config router static
    edit 1
        set device "port1"
        set gateway 192.168.122.1
    next
end
```

### 5.3 DNS
```
config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end
```

### 5.4 Firewall Policy (LAN2 → WAN)
```
config firewall policy
    edit 1
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

### 5.5 HA (Master, priority 200)
```
config system ha
    set group-name "FortiLab-HA2"
    set mode a-p
    set hbdev "port3"
    set priority 200
    set session-sync-dev "port3"
end
```

### 5.6 Verify
```
get system interface physical
execute ping 8.8.8.8
get system ha status
```

---

## 6. FGT-Secondary-HA

**Template:** QEMU, `fortios.qcow2` (7.0.9 pre-licensed), 2048 MB RAM, 1 vCPU, 8 adapters
**Console:** Telnet
**Login:** `admin` / no password

### 6.1 Interfaces
```
config system interface
    edit "port1"
        set mode dhcp
        set allowaccess ping https ssh
    next
    edit "port2"
        set ip 192.168.20.2 255.255.255.0
        set allowaccess ping
    next
    edit "port3"
        set ip 169.254.0.4 255.255.255.252
        set allowaccess ping
    next
end
```

### 6.2 Default Route
```
config router static
    edit 1
        set device "port1"
        set gateway 192.168.122.1
    next
end
```

### 6.3 DNS
```
config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end
```

### 6.4 HA (Backup, priority 100)
```
config system ha
    set group-name "FortiLab-HA2"
    set mode a-p
    set hbdev "port3"
    set priority 100
    set session-sync-dev "port3"
end
```

### 6.5 Verify
```
get system ha status
# Should show BACKUP
```

---

## 7. OVS-LAN1

**Template:** Docker, `gns3/openvswitch:latest`, 16 adapters
**Console:** Telnet

### 7.1 Setup
```bash
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4 eth5 eth6; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone
```

### 7.2 Verify
```bash
ovs-vsctl show
```

### 7.3 Port Map
| OVS Port | Connected To |
|---|---|
| eth0 | FGT-Primary port2 |
| eth1 | FGT-Primary-HA port2 (passive) |
| eth2 | PC1 |
| eth3 | webterm-1 |
| eth4 | appServer-1 |
| eth5 | PostgreSQL-1 |
| eth6 | Docker Router eth0 |

---

## 8. OVS-LAN2

**Template:** Docker, `gns3/openvswitch:latest`, 16 adapters
**Console:** Telnet

### 8.1 Setup
```bash
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4 eth5 eth6 eth7; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone
```

### 8.2 Verify
```bash
ovs-vsctl show
```

### 8.3 Port Map
| OVS Port | Connected To |
|---|---|
| eth0 | FGT-Secondary port2 |
| eth1 | FGT-Secondary-HA port2 (passive) |
| eth2 | Alpine-DHCP |
| eth3 | Ubuntu Desktop |
| eth4 | Traffic-Gen-1 |
| eth5 | Grafana-1 |
| eth6 | Prometheus-1 |
| eth7 | Docker Router eth1 |

---

## 9. Docker Router

**Template:** Docker, `docker-router:latest` (custom build), 2 adapters
**Console:** Telnet

### 9.1 Build Image
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

### 9.2 Verify
```bash
ip addr show eth0   # 192.168.10.254/24
ip addr show eth1   # 192.168.20.254/24
cat /proc/sys/net/ipv4/ip_forward  # 1
ping 192.168.10.1    # FGT-Primary LAN
ping 192.168.20.1    # FGT-Secondary LAN
```

---

## 10. PC1 (VPCS)

**Template:** VPCS built-in
**Console:** Telnet

### 10.1 Setup
```
PC1> dhcp
PC1> show ip
```

### 10.2 Verify
```
PC1> ping 192.168.10.1
PC1> ping 8.8.8.8
PC1> ping 192.168.20.1
```

---

## 11. webterm-1

**Template:** Docker, `gns3/webterm:latest`, 1 adapter
**Console:** VNC

Gets IP via DHCP from FGT-Primary automatically. Open VNC console to use the browser.

---

## 12. Alpine-DHCP

**Template:** Docker, `alpine-dhcp:latest` (custom build), 1 adapter
**Console:** Telnet

### 12.1 Build Image
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

### 12.2 Verify
```bash
ps aux | grep dnsmasq
cat /var/lib/misc/dnsmasq.leases
```

---

## 13. Ubuntu Desktop

**Template:** QEMU, `ubuntu-24.04-minimal-cloudimg-amd64.img` (modified), 2048 MB RAM, 2 vCPU, 1 adapter
**Console:** VNC
**Credentials:** `ubuntu` / `gns3` (NOPASSWD sudo)

### 13.1 Network Setup
```bash
sudo dhcpcd enp2s0
ip addr show enp2s0
```

### 13.2 Verify
```bash
ping 8.8.8.8
```

---

## 14. appServer-1 (Flask)

**Template:** Docker, `python:3.12-alpine`, 1 adapter
**Console:** Telnet

### 14.1 IP Setup
```bash
ip addr add 192.168.10.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.10.1
echo nameserver 8.8.8.8 > /etc/resolv.conf
```

### 14.2 Deploy App
```bash
pip install flask psycopg2-binary requests --break-system-packages
python3 /opt/app.py &
```

### 14.3 Verify
```bash
curl http://127.0.0.1/
curl http://127.0.0.1/db-check
```

---

## 15. PostgreSQL-1

**Template:** Docker, `postgres:16-alpine`, 1 adapter
**Console:** Telnet
**Environment:** `POSTGRES_PASSWORD=gns3`

### 15.1 IP Setup
```bash
ip addr add 192.168.10.11/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.10.1
```

### 15.2 Create Database
```bash
su - postgres -c "pg_ctl -D /var/lib/postgresql/data start"
su - postgres -c "psql -c 'CREATE DATABASE labdb;'"
su - postgres -c "psql -c \"CREATE USER labuser WITH PASSWORD 'gns3lab';\""
su - postgres -c "psql -c 'GRANT ALL PRIVILEGES ON DATABASE labdb TO labuser;'"
```

### 15.3 Verify
```bash
su - postgres -c "psql -c '\\l'"
```

---

## 16. Grafana-1

**Template:** Docker, `grafana/grafana:latest`, 1 adapter
**Console:** VNC

### 16.1 IP Setup
```bash
ip addr add 192.168.20.11/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
```

### 16.2 Access
Open VNC or browse to `http://192.168.20.11:3000`
Default login: `admin` / `admin`

---

## 17. Prometheus-1

**Template:** Docker, `prom/prometheus:latest`, 1 adapter
**Console:** VNC

### 17.1 IP Setup
```bash
ip addr add 192.168.20.12/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
```

### 17.2 Access
Open VNC or browse to `http://192.168.20.12:9090`

---

## 18. Traffic-Gen-1

**Template:** Docker, `alpine:latest`, 1 adapter
**Console:** Telnet

### 18.1 IP Setup
```bash
ip addr add 192.168.20.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
```

### 18.2 Install Tools
```bash
apk add --no-cache curl busybox-extras
```

### 18.3 Syslog Receiver
```bash
nc -ulvp 514
```

### 18.4 Verify
```bash
curl ifconfig.me
```

---

## 16. Traffic-Gen-2

**Template:** Docker, `alpine:latest`, 1 adapter, `GNS3_USER=root`
**Console:** Telnet
**Network:** OVS-LAN2 eth6, LAN2

### 16.1 Set IP
```bash
ip addr add 192.168.20.13/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
echo nameserver 8.8.8.8 > /etc/resolv.conf
```

### 16.2 Traffic Script
```bash
while true; do
    curl -s http://192.168.10.10/ > /dev/null 2>&1
    curl -s http://192.168.20.11:3000/ > /dev/null 2>&1
    ping -c 1 192.168.10.1 > /dev/null 2>&1
    sleep 10
done &
```

---

## 17. OCI Cloud Instance (External)

**Deployment:** Ubuntu 24.04 on Oracle Cloud

### 19.1 Security Group Rules
| Protocol | Port | Source |
|---|---|---|
| UDP | 500 | FGT-Primary WAN IP |
| UDP | 4500 | FGT-Primary WAN IP |
| TCP | 80, 443 | Both FGT WAN IPs |
| ICMP | — | Both FGT WAN IPs |

### 19.2 Libreswan IPsec
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

### 19.3 Threat Simulator
```bash
sudo apt install -y python3-flask socat
sudo python3 /opt/threat-sim/app.py &
```

### 19.4 Endpoints
| Endpoint | What It Simulates |
|---|---|
| `GET /eicar` | AV signature match |
| `GET /attack?sql=payload` | SQL injection |
| `GET /attack?xss=payload` | XSS attack |
| `GET /phishing` | Phishing URL |
| `GET /hacking-tools` | Blocked tool URL |
| `GET /inspect` | Shows requester source IP |
| `nmap -sS <FGT-WAN>` | Port scan detection |
