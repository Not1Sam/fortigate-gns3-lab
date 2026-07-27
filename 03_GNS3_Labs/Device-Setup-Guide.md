# Device Setup Guide

Per-node setup instructions for every device in the topology.

---

## 1. NAT1 (Cloud)

Provides WAN internet access for the lab.

**Linux:** Uses `virbr0` (libvirt default NAT bridge, 192.168.122.1).
1. Right-click NAT1 → Configure → check `virbr0`
2. Verify: `ip addr show virbr0` → should show `192.168.122.1/24`

**Windows (GNS3 VM):** Uses `VMnet8` (VMware NAT adapter) or VirtualBox host-only adapter.
1. Find the adapter: run `ipconfig` on Windows host, look for VMware VMnet8
2. Right-click NAT1 → Configure → check that adapter
3. Create a Cloud template pointing to that interface

---

## 2. Switch1 (Ethernet Switch)

No configuration needed — plug-and-play L2 switch.

**Wiring:**
| Switch Port | Connected To |
|---|---|
| a0p0 | NAT1 |
| a0p1 | FGT-Primary port1 |
| a0p2 | FGT-Secondary port1 |

---

## 3. FGT-Primary

**Template:** QEMU, `fgt-v7.4.12.qcow2`, 2048 MB RAM, 1 vCPU, 8 adapters
**Console:** Telnet
**Login:** `admin` / no password

### 3.1 Interfaces
```
config system interface
    edit port1
        set mode dhcp
        set allowaccess ping https ssh
    next
    edit port2
        set mode static
        set ip 192.168.10.1 255.255.255.0
        set allowaccess ping
    next
    edit port3
        set mode static
        set ip 10.0.0.1 255.255.255.252
        set allowaccess ping
    next
end
```

### 3.2 Default Route
```
config router static
    edit 1
        set device port1
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
        set interface port2
        set netmask 255.255.255.0
        set default-gateway 192.168.10.1
        set dns-service default
        set vci-match disable
        config ip-range
            edit 1
                set start-ip 192.168.10.100
                set end-ip 192.168.10.200
            next
        end
    next
end
```

### 3.5 Firewall Policies
```
# LAN1 -> WAN (SNAT)
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

# Transit -> WAN (SNAT)
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

# Transit -> LAN1 (inter-LAN)
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

### 3.6 OSPF
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

### 3.7 Verify
```
execute ping 8.8.8.8
execute ping 10.0.0.2
get router info ospf neighbor
```

---

## 4. FGT-Secondary

**Template:** Linked clone of FGT-Primary, 2048 MB RAM, 1 vCPU, 8 adapters
**Console:** Telnet
**Login:** `admin` / no password

### 4.1 Interfaces
```
config system interface
    edit port1
        set mode dhcp
        set allowaccess ping https ssh
    next
    edit port2
        set mode static
        set ip 192.168.20.1 255.255.255.0
        set allowaccess ping
    next
    edit port3
        set mode static
        set ip 10.0.0.2 255.255.255.252
        set allowaccess ping
    next
end
```

### 4.2 Default Route
```
config router static
    edit 1
        set device port1
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

### 4.4 Firewall Policies
```
# LAN2 -> WAN (SNAT)
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

# Transit -> WAN (SNAT)
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

# Transit -> LAN2 (inter-LAN)
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

### 4.5 OSPF
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

### 4.6 Verify
```
execute ping 8.8.8.8
execute ping 10.0.0.1
get router info ospf neighbor
```

---

## 5. OVS-LAN1 (OpenvSwitch-1)

**Template:** Docker, `gns3/openvswitch:latest`, 16 adapters
**Console:** Telnet

### 5.1 Setup
```bash
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone
```

### 5.2 Verify
```bash
ovs-vsctl show
# Should show br0 with ports eth0-eth4
```

### 5.3 Port Map
| OVS Port | Connected To |
|---|---|
| eth0 | FGT-Primary port2 |
| eth1 | PC1 |
| eth2 | webterm-1 |
| eth3 | appServer-1 |
| eth4 | PostgreSQL-1 |

---

## 6. OVS-LAN2 (OpenvSwitch-2)

**Template:** Docker, `gns3/openvswitch:latest`, 16 adapters
**Console:** Telnet

### 6.1 Setup
```bash
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4 eth5; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone
```

### 6.2 Verify
```bash
ovs-vsctl show
# Should show br0 with ports eth0-eth5
```

### 6.3 Port Map
| OVS Port | Connected To |
|---|---|
| eth0 | FGT-Secondary port2 |
| eth1 | Alpine-DHCP |
| eth2 | Ubuntu Desktop |
| eth3 | Traffic-Gen-1 |
| eth4 | Grafana-1 |
| eth5 | Prometheus-1 |

---

## 7. PC1 (VPCS)

**Template:** VPCS built-in
**Console:** Telnet

### 7.1 Setup
```
PC1> dhcp
PC1> show ip
```

### 7.2 Verify
```
PC1> ping 192.168.10.1
PC1> ping 8.8.8.8
```

---

## 8. webterm-1

**Template:** Docker, `gns3/webterm:latest`, 1 adapter
**Console:** VNC

Gets IP via DHCP from FGT-Primary automatically. Open VNC console to use the browser.

---

## 9. Alpine-DHCP

**Template:** Docker, `alpine-dhcp:latest` (custom build), 1 adapter
**Console:** Telnet

### 9.1 Build Image
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

### 9.2 Verify
```bash
ps aux | grep dnsmasq
# Or from console: check DHCP leases
cat /var/lib/misc/dnsmasq.leases
```

---

## 10. Ubuntu Desktop

**Template:** QEMU, `ubuntu-24.04-minimal-cloudimg-amd64.img` (modified), 2048 MB RAM, 2 vCPU, 1 adapter
**Console:** VNC
**Credentials:** `ubuntu` / `gns3` (NOPASSWD sudo)

### 10.1 Network Setup (DHCP)
```bash
# Automatic via Alpine DHCP server, but if needed:
sudo dhcpcd enp2s0
ip addr show enp2s0
```

### 10.2 Verify Internet
```bash
ping 8.8.8.8
curl ifconfig.me
```

---

## 11. appServer-1 (Flask)

**Template:** Docker, `python:3.12-alpine`, 1 adapter
**Console:** Telnet

### 11.1 IP Setup
```bash
ip addr add 192.168.10.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.10.1
```

### 11.2 Deploy App
```bash
pip install flask psycopg2-binary
```

```python
# /app.py
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
```

### 11.3 Run
```bash
python /app.py &
```

### 11.4 Verify
```bash
curl http://127.0.0.1:80/
```

---

## 12. PostgreSQL-1

**Template:** Docker, `postgres:16-alpine`, 1 adapter
**Console:** Telnet
**Environment:** `POSTGRES_PASSWORD=gns3`

### 12.1 IP Setup
```bash
ip addr add 192.168.10.11/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.10.1
```

### 12.2 Create Database
```bash
psql -U postgres -c "CREATE DATABASE appdb;"
```

### 12.3 Verify
```bash
psql -U postgres -c "\l"
```

---

## 13. Grafana-1

**Template:** Docker, `grafana/grafana:latest`, 1 adapter
**Console:** VNC

### 13.1 IP Setup
```bash
ip addr add 192.168.20.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
```

### 13.2 Access
Open VNC or browse to `http://192.168.20.10:3000`
Default login: `admin` / `admin`

---

## 14. Prometheus-1

**Template:** Docker, `prom/prometheus:latest`, 1 adapter
**Console:** VNC

### 14.1 IP Setup
```bash
ip addr add 192.168.20.11/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
```

### 14.2 Access
Open VNC or browse to `http://192.168.20.11:9090`

---

## 15. Traffic-Gen-1

**Template:** Docker, `alpine:latest`, 1 adapter
**Console:** Telnet

### 15.1 IP Setup
```bash
ip addr add 192.168.20.12/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
```

### 15.2 Install Tools
```bash
apk add --no-cache curl busybox-extras
```

### 15.3 Syslog Receiver
```bash
nc -ulvp 514
```

### 15.4 Verify Internet
```bash
curl ifconfig.me
# Should show FGT-Secondary WAN IP
```

---

## 16. OCI Cloud Instance (External)

**Deployment:** Ubuntu 24.04 on Oracle Cloud (or any public cloud)

### 16.1 Security Group Rules
| Protocol | Port | Source |
|---|---|---|
| UDP | 500 | FGT-Primary WAN IP |
| UDP | 4500 | FGT-Primary WAN IP |
| TCP | 80, 443 | Both FGT WAN IPs |
| ICMP | — | Both FGT WAN IPs |

### 16.2 Libreswan IPsec
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

### 16.3 Threat Simulator
```bash
sudo apt install -y python3-flask socat
sudo python3 /opt/threat-sim/app.py &
```

### 16.4 Endpoints
| Endpoint | What It Simulates |
|---|---|
| `GET /eicar` | AV signature match |
| `GET /attack?sql=payload` | SQL injection |
| `GET /attack?xss=payload` | XSS attack |
| `GET /phishing` | Phishing URL |
| `GET /hacking-tools` | Blocked tool URL |
| `GET /inspect` | Shows requester source IP |
| `nmap -sS <FGT-WAN>` | Port scan detection |
