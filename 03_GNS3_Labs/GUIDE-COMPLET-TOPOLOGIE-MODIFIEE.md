# Guide Complet — Topologie Modifiee (A a Z)

> Guide complet pour construire, configurer et tester le laboratoire FortiGate modifie dans GNS3.
> Suit exactement la topologie du rapport : 4 FortiGate HA + Docker Router + OCI Cloud.

---

## Table des matieres

1. [Prerequis](#1-prerequis)
2. [Creation des images Docker](#2-creation-des-images-docker)
3. [Enregistrement des templates GNS3](#3-enregistrement-des-templates-gns3)
4. [Construction de la topologie](#4-construction-de-la-topologie)
5. [Cablage](#5-cablage)
6. [Configuration des switches OVS](#6-configuration-des-switches-ovs)
7. [Configuration des FortiGate (tous les 4)](#7-configuration-des-fortigate)
8. [Configuration HA](#8-configuration-ha)
9. [Configuration du Docker Router](#9-configuration-du-docker-router)
10. [Configuration des services Docker](#10-configuration-des-services-docker)
11. [Test de connectivite de base](#11-test-de-connectivite-de-base)
12. [Profils de securite](#12-profils-de-securite)
13. [VPN IPsec et SSL](#13-vpn-ipsec-et-ssl)
14. [OCI Cloud](#14-oci-cloud)
15. [Journalisation](#15-journalisation)
16. [Scenarios de test](#16-scenarios-de-test)
17. [Captures d'ecran](#17-captures-dcran)
18. [Depannage](#18-depannage)

---

## 1. Prerequis

### 1.1 Logiciel requis

| Logiciel | Usage |
|---|---|
| GNS3 (GUI + Server) | Simulation reseau |
| QEMU/KVM | Emulation des FortiGate et VMs |
| Docker | Conteneurs de services |
| Telnet | Console des FortiGate et OVS |
| libvirt | Reseau NAT (virbr0) |

### 1.2 Images requises

| Image | Source | Pour qui |
|---|---|---|
| `fgt-v7.4.12.qcow2` | Fortinet support site | FGT-Primary, FGT-Secondary |
| `fortios.qcow2` (7.0.9 pre-licensed) | Fortinet (image pre-licencee) | FGT-Primary-HA, FGT-Secondary-HA |
| `ubuntu-24.04-minimal-cloudimg-amd64.img` | cloud-images.ubuntu.com | Ubuntu Desktop |

### 1.3 Verifier l'hote

```bash
# Virtualisation
egrep -c '(vmx|svm)' /proc/cpuinfo   # Doit etre > 0

# Services
systemctl status libvirtd --no-pager | head -3
systemctl status docker --no-pager | head -3

# Reseau NAT
ip addr show virbr0   # Doit afficher 192.168.122.1/24

# Si virbr0 manque :
sudo virsh net-start default
sudo virsh net-autostart default
```

### 1.4 Activer le NAT Internet

```bash
sudo iptables -t nat -A POSTROUTING -s 192.168.122.0/24 ! -d 192.168.122.0/24 -j MASQUERADE
sudo iptables -A FORWARD -s 192.168.122.0/24 -j ACCEPT
```

---

## 2. Creation des images Docker

### 2.1 Docker Router (Debian + IP routing)

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

### 2.2 Alpine DHCP Server

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

### 2.3 App Server (Flask)

```bash
docker build -t fortilab-appserver:latest - << 'EOF'
FROM python:3.12-alpine
RUN pip install --no-cache-dir flask psycopg2-binary requests
COPY app.py /opt/app.py
CMD ["python3", "/opt/app.py"]
EOF
```

Creez le fichier `app.py` au meme niveau que le Dockerfile :

```python
from flask import Flask, request
import psycopg2, os

app = Flask(__name__)

DB_HOST = os.getenv("DB_HOST", "192.168.10.11")
DB_NAME = os.getenv("DB_NAME", "labdb")
DB_USER = os.getenv("DB_USER", "labuser")
DB_PASS = os.getenv("DB_PASS", "gns3lab")

@app.route("/")
def home():
    return "<h1>App Server</h1><p>Connected to PostgreSQL</p>"

@app.route("/health")
def health():
    return {"status": "ok", "service": "app-server"}

@app.route("/db-check")
def db_check():
    try:
        conn = psycopg2.connect(host=DB_HOST, dbname=DB_NAME, user=DB_USER, password=DB_PASS)
        cur = conn.cursor()
        cur.execute("SELECT 1")
        return {"database": "connected", "result": cur.fetchone()[0]}
    except Exception as e:
        return {"database": "error", "message": str(e)}, 500

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=80)
```

---

## 3. Enregistrement des templates GNS3

### 3.1 Templates QEMU

Dans GNS3 GUI : `Edit` → `Preferences` → `QEMU VMs`

| Template | Image | RAM | vCPU | Adapters | Console |
|---|---|---|---|---|---|
| FGT-Primary | `fgt-v7.4.12.qcow2` | 2048 MB | 1 | 8 | Telnet |
| FGT-Secondary | Clone lie de FGT-Primary | 2048 MB | 1 | 8 | Telnet |
| FGT-7.0.9-HA | `fortios.qcow2` | 2048 MB | 1 | 8 | Telnet |
| Ubuntu-Desktop | `ubuntu-24.04-minimal-cloudimg` | 2048 MB | 2 | 1 | VNC |

**Parametres communs QEMU :**
- Adapter type: `virtio-net-pci`
- Disk interface: `virtio`
- Console type: `telnet` (sauf Ubuntu → `vnc`)

### 3.2 Templates Docker

Dans GNS3 GUI : `Edit` → `Preferences` → `Docker containers`

| Template | Image | Adapters | Console |
|---|---|---|---|
| Docker Router | `docker-router:latest` | 2 | Telnet |
| OpenvSwitch-1 | `gns3/openvswitch:latest` | 16 | Telnet |
| OpenvSwitch-2 | `gns3/openvswitch:latest` | 16 | Telnet |
| webterm-1 | `gns3/webterm:latest` | 1 | VNC |
| Alpine-DHCP | `alpine-dhcp:latest` | 1 | Telnet |
| appServer-1 | `fortilab-appserver:latest` | 1 | Telnet |
| PostgreSQL-1 | `postgres:16-alpine` | 1 | Telnet |
| Grafana-1 | `grafana/grafana:latest` | 1 | VNC |
| Prometheus-1 | `prom/prometheus:latest` | 1 | VNC |
| Traffic-Gen-1 | `alpine:latest` | 1 | Telnet |

### 3.3 Nodes integres GNS3

| Node | Type |
|---|---|
| PC1 | VPCS |
| NAT1 | Cloud (virbr0) |
| Switch1 | Ethernet switch (8 ports) |

---

## 4. Construction de la topologie

### 4.1 Node inventory

```
QEMU VMs (5)        : FGT-Primary, FGT-Secondary, FGT-Primary-HA, FGT-Secondary-HA, Ubuntu
Docker nodes (10)    : OVS-LAN1, OVS-LAN2, Docker Router, webterm-1, Alpine-DHCP,
                       appServer-1, PostgreSQL-1, Grafana-1, Prometheus-1, Traffic-Gen-1
Built-in (3)         : PC1, NAT1, Switch1
                       TOTAL = 18 nodes
```

### 4.2 Positions sur le canvas

Placez les nodes comme suit (approximativement) :

```
                        NAT1
                         |
                      Switch1
                    /    |    \
          FGT-Primary  FGT-Secondary  FGT-Primary-HA  FGT-Secondary-HA
               |              |              |              |
            OVS-LAN1       OVS-LAN2       OVS-LAN1       OVS-LAN2
           /  |  \         / |  \         (passif)       (passif)
       PC1 webterm  AppSrv  Alpine Ubuntu Traffic Grafana Prometheus
         PostgreSQL         DHCP                  Docker Router
                                \                /
                               eth0           eth1
```

---

## 5. Cablage

### 5.1 WAN (Switch1)

| Switch1 port | Connecte a | Port FGT |
|---|---|---|
| eth0 | NAT1 | — |
| eth1 | FGT-Primary | port1 (a0p0) |
| eth2 | FGT-Secondary | port1 (a0p0) |
| eth3 | FGT-Primary-HA | port1 (a0p0) |
| eth4 | FGT-Secondary-HA | port1 (a0p0) |

### 5.2 HA Heartbeat (port3 ↔ port3)

| FGT-Primary port3 (a2p0) | ↔ | FGT-Primary-HA port3 (a2p0) |
|---|---|---|
| FGT-Secondary port3 (a2p0) | ↔ | FGT-Secondary-HA port3 (a2p0) |

### 5.3 LAN1 (OVS-LAN1)

| OVS-LAN1 eth | Connecte a |
|---|---|
| eth0 | FGT-Primary port2 (a1p0) |
| eth1 | FGT-Primary-HA port2 (a1p0) — passif |
| eth2 | PC1 |
| eth3 | webterm-1 |
| eth4 | appServer-1 |
| eth5 | PostgreSQL-1 |
| eth6 | Docker Router eth0 |

### 5.4 LAN2 (OVS-LAN2)

| OVS-LAN2 eth | Connecte a |
|---|---|
| eth0 | FGT-Secondary port2 (a1p0) |
| eth1 | FGT-Secondary-HA port2 (a1p0) — passif |
| eth2 | Alpine-DHCP |
| eth3 | Ubuntu Desktop |
| eth4 | Traffic-Gen-1 |
| eth5 | Grafana-1 |
| eth6 | Prometheus-1 |
| eth7 | Docker Router eth1 |

---

## 6. Configuration des switches OVS

### 6.1 OVS-LAN1

Console Telnet dans OVS-LAN1 :
```bash
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4 eth5 eth6; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone
ovs-vsctl show
```

### 6.2 OVS-LAN2

Console Telnet dans OVS-LAN2 :
```bash
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4 eth5 eth6 eth7; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone
ovs-vsctl show
```

---

## 7. Configuration des FortiGate

> **Ordre important :** Configurez d'abord les 4 FGTs, puis activez le HA.

### 7.1 FGT-Primary (Master, Cluster 1)

Console Telnet dans FGT-Primary (login: `admin`, pas de mot de passe) :

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

config router static
    edit 1
        set device "port1"
        set gateway 192.168.122.1
    next
end

config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end

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

**Verification :**
```
get system interface physical
execute ping 8.8.8.8
```

### 7.2 FGT-Primary-HA (Backup, Cluster 1)

Console Telnet dans FGT-Primary-HA :

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

config router static
    edit 1
        set device "port1"
        set gateway 192.168.122.1
    next
end

config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end
```

### 7.3 FGT-Secondary (Master, Cluster 2)

Console Telnet dans FGT-Secondary :

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

config router static
    edit 1
        set device "port1"
        set gateway 192.168.122.1
    next
end

config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end

config system dhcp server
    edit 1
        set interface "port2"
        set default-gateway 192.168.20.1
        set netmask 255.255.255.0
        config ip-range
            edit 1
                set start-ip 192.168.20.100
                set end-ip 192.168.20.200
            next
        end
        set dns-service default
    next
end

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

**Verification :**
```
get system interface physical
execute ping 8.8.8.8
```

### 7.4 FGT-Secondary-HA (Backup, Cluster 2)

Console Telnet dans FGT-Secondary-HA :

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

config router static
    edit 1
        set device "port1"
        set gateway 192.168.122.1
    next
end

config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end
```

---

## 8. Configuration HA

> Activez le HA **apres** avoir configure les interfaces sur les 4 FGTs.

### 8.1 Cluster 1 : FGT-Primary + FGT-Primary-HA

**Sur FGT-Primary (priority 200 = master) :**
```
config system ha
    set group-name "FortiLab-HA"
    set mode a-p
    set hbdev "port3"
    set priority 200
    set session-sync-dev "port3"
end
```

**Sur FGT-Primary-HA (priority 100 = backup) :**
```
config system ha
    set group-name "FortiLab-HA"
    set mode a-p
    set hbdev "port3"
    set priority 100
    set session-sync-dev "port3"
end
```

**Verification :**
```
get system ha status
```
Attendu : FGT-Primary = `MASTER`, FGT-Primary-HA = `BACKUP`

### 8.2 Cluster 2 : FGT-Secondary + FGT-Secondary-HA

**Sur FGT-Secondary (priority 200 = master) :**
```
config system ha
    set group-name "FortiLab-HA2"
    set mode a-p
    set hbdev "port3"
    set priority 200
    set session-sync-dev "port3"
end
```

**Sur FGT-Secondary-HA (priority 100 = backup) :**
```
config system ha
    set group-name "FortiLab-HA2"
    set mode a-p
    set hbdev "port3"
    set priority 100
    set session-sync-dev "port3"
end
```

**Verification :**
```
get system ha status
```
Attendu : FGT-Secondary = `MASTER`, FGT-Secondary-HA = `BACKUP`

---

## 9. Configuration du Docker Router

Le Docker Router est deja demarre avec les bonnes IPs (via le CMD du Dockerfile). Verifiez :

Console Telnet dans Docker Router :
```bash
ip addr show eth0   # 192.168.10.254/24
ip addr show eth1   # 192.168.20.254/24
cat /proc/sys/net/ipv4/ip_forward   # 1
```

**Test de routage :**
```bash
ping 192.168.10.1    # FGT-Primary LAN
ping 192.168.20.1    # FGT-Secondary LAN
```

Si le ping ne fonctionne pas, verifiez que les politiques firewall autorisent le trafic LAN-to-LAN via le Docker Router.

---

## 10. Configuration des services Docker

### 10.1 PostgreSQL

Console Telnet dans PostgreSQL-1 :
```bash
ip addr add 192.168.10.11/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.10.1

su - postgres -c "pg_ctl -D /var/lib/postgresql/data start"
su - postgres -c "psql -c 'CREATE DATABASE labdb;'"
su - postgres -c "psql -c \"CREATE USER labuser WITH PASSWORD 'gns3lab';\""
su - postgres -c "psql -c 'GRANT ALL PRIVILEGES ON DATABASE labdb TO labuser;'"

# Verifier
su - postgres -c "psql -c '\\l'"
```

### 10.2 App Server (Flask)

Console Telnet dans appServer-1 :
```bash
ip addr add 192.168.10.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.10.1
echo nameserver 8.8.8.8 > /etc/resolv.conf

python3 /opt/app.py &
```

**Verification :**
```bash
curl http://127.0.0.1/
curl http://127.0.0.1/db-check
```

### 10.3 Grafana

Console Telnet dans Grafana-1 :
```bash
ip addr add 192.168.20.11/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
```

Acces : `http://192.168.20.11:3000` (admin/admin)

### 10.4 Prometheus

Console Telnet dans Prometheus-1 :
```bash
ip addr add 192.168.20.12/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
```

Acces : `http://192.168.20.12:9090`

### 10.5 Traffic-Gen

Console Telnet dans Traffic-Gen-1 :
```bash
ip addr add 192.168.20.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
apk add --no-cache curl busybox-extras
```

### 10.6 Alpine DHCP (LAN2)

Demarre automatiquement via son CMD. Verifiez :
```bash
ps aux | grep dnsmasq
```

### 10.7 webterm-1

Obtient son IP via DHCP de FGT-Primary automatiquement.

### 10.8 PC1 (VPCS)

```
PC1> dhcp
PC1> show ip
```

### 10.9 Ubuntu Desktop

```bash
sudo dhcpcd enp2s0
ip addr show enp2s0
```

---

## 11. Test de connectivite de base

Executez ces tests dans l'ordre :

| # | Test | Depuis | Commande | Attendu |
|---|---|---|---|---|
| 1 | WAN FGT-Primary | FGT-Primary | `execute ping 8.8.8.8` | Reponse |
| 2 | WAN FGT-Secondary | FGT-Secondary | `execute ping 8.8.8.8` | Reponse |
| 3 | LAN1 local | PC1 | `ping 192.168.10.1` | Reponse |
| 4 | LAN2 local | Ubuntu | `ping 192.168.20.1` | Reponse |
| 5 | DHCP LAN1 | PC1 | `dhcp` puis `show ip` | 192.168.10.x |
| 6 | DHCP LAN2 | Ubuntu | `dhcpcd enp2s0` | 192.168.20.x |
| 7 | Internet LAN1 | PC1 | `ping 8.8.8.8` | Reponse via FGT-Primary |
| 8 | Internet LAN2 | Ubuntu | `ping 8.8.8.8` | Reponse via FGT-Secondary |
| 9 | Cross-LAN | PC1 | `ping 192.168.20.10` | Reponse via Docker Router |
| 10 | App Server | PC1 | `ping 192.168.10.10` | Reponse |
| 11 | PostgreSQL | AppServer | `curl http://192.168.10.10/db-check` | `{"database":"connected"}` |
| 12 | Grafana | Ubuntu | `curl http://192.168.20.11:3000` | Login page |

---

## 12. Profils de securite

### 12.1 Sur FGT-Primary

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

config firewall ssl-ssh-profile
    edit "deep-inspection"
        set ssl-inspection deep-inspection
    next
end

config firewall policy
    edit 1
        set groups "default-av" "default-ips" "default-wf"
        set ssl-ssh-profile "deep-inspection"
    next
end
```

### 12.2 Sur FGT-Secondary

Meme commandes que 12.1 (remplacez juste le nom de la politique si necessaire).

---

## 13. VPN IPsec et SSL

### 13.1 IPsec Site-to-Site (FGT-Primary → OCI)

**Sur FGT-Primary :**

```
config vpn ipsec phase1-interface
    edit "phase1-to-oci"
        set interface "port1"
        set ike-version 2
        set remote-gw <IP-PUBLIQUE-OCI>
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

### 13.2 SSL VPN

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

config firewall address
    edit "ssl-vpn-pool"
        set type iprange
        set start-ip 10.212.134.200
        set end-ip 10.212.134.210
    next
end
```

---

## 14. OCI Cloud

### 14.1 Instance OCI

- Ubuntu 24.04, A1.Flex (1 OCPU, 6 GB RAM)
- IP publique requis
- Security group : UDP 500, 4500 + TCP 80, 443 depuis les IPs WAN des FGTs

### 14.2 Libreswan (sur l'instance OCI)

```bash
sudo apt update && sudo apt install -y libreswan

sudo tee /etc/ipsec.conf << 'EOF'
conn fgt-primary
    left=%defaultroute
    leftid=<IP-PUBLIQUE-OCI>
    leftsubnet=<SOUS-RESEAU-VPC-OCI>
    right=<IP-WAN-FGT-PRIMARY>
    rightsubnet=192.168.10.0/24
    ikelifetime=24h
    lifetime=8h
    ike=aes256-sha2_256;modp2048
    phase2=aes256-sha2_256
    auto=start
EOF

sudo ipsec restart
```

### 14.3 Simulateur de menaces

```bash
sudo apt install -y python3-flask socat
# Deployer l'application (voir Full-Topology-Spec.md pour les endpoints)
sudo python3 /opt/threat-sim/app.py &
```

---

## 15. Journalisation

### 15.1 Syslog (sur les 4 FGTs)

```
config log syslogd setting
    set status enable
    set server 192.168.20.10
    set port 514
    set facility local7
end
```

### 15.2 Verification

Sur Traffic-Gen-1 :
```bash
nc -ulvp 514
```
Vous devriez voir les logs arriver des FGTs.

---

## 16. Scenarios de test

| # | Scenario | Comment tester | Resultat attendu |
|---|---|---|---|
| 1 | Acces Internet | `ping 8.8.8.8` depuis PC1 ou Ubuntu | Reponse |
| 2 | SNAT | `curl ifconfig.me` depuis Traffic-Gen | IP WAN du FGT |
| 3 | Inter-LAN | `ping 192.168.20.10` depuis PC1 | Reponse via Docker Router |
| 4 | Failover HA | Deconnectez port3 du FGT-Primary | Le backup prend le relais en 3-5s |
| 5 | Blocage AV | `curl http://www.eicar.org/eicar.com.txt` | Page de blocage FortiGate |
| 6 | Blocage IPS | `curl "http://<IP-OCI>/attack?sql=payload"` | Blocage |
| 7 | Web Filter | `curl "http://<IP-OCI>/phishing"` | Blocage |
| 8 | IPsec VPN | `get vpn ipsec tunnel details` | Tunnel UP |
| 9 | SSL VPN | Connectez-vous via le portail SSL VPN | Tunnel etabli |
| 10 | App Server | `curl http://192.168.10.10/` depuis webterm | Reponse HTML |
| 11 | Grafana | Ouvrez `http://192.168.20.11:3000` | Dashboard |
| 12 | PostgreSQL | `curl http://192.168.10.10/db-check` | `{"database":"connected"}` |

---

## 17. Captures d'ecran

Prenez ces 15 captures et deposez-les dans `Rapport-de-Stage/figures/` :

| # | Nom du fichier | Ce a capturer |
|---|---|---|
| 1 | `topo-gns3.png` | Canvas GNS3 complet (topologie modifiee) |
| 2 | `gns3-dashboard.png` | Dashboard GNS3 avec tous les nodes actifs |
| 3 | `interfaces-fgtp.png` | FGT-Primary : `get system interface physical` |
| 4 | `politiques-fw.png` | FGT-Primary WebUI : Policy and Objects |
| 5 | `ha-status.png` | FGT-Primary : `get system ha status` |
| 6 | `docker-router.png` | Docker Router : `ip addr` + `ip route` |
| 7 | `ipsec-status.png` | FGT-Primary : `diagnose vpn ike gateway list` |
| 8 | `grafana-dashboard.png` | Tableau de bord Grafana |
| 9 | `oci-instance.png` | Console OCI montrant l'instance A1.Flex |
| 10 | `test-ping-wan.png` | FGT-Primary : `ping 8.8.8.8` (5 reponses) |
| 11 | `test-traceroute.png` | FGT-Primary : `execute traceroute 8.8.8.8` |
| 12 | `test-ha-failover.png` | Logs du basculement HA (deconnectez port3) |
| 13 | `test-ipsec.png` | Tunnel IPsec UP avec SA actives |
| 14 | `test-eicar.png` | Navigateur : page de blocage EICAR |
| 15 | `topologie-annexe.png` | Export haute resolution de la topologie |

---

## 18. Depannage

### Probleme : FGT n'obtient pas d'IP DHCP sur port1

```bash
# Verifier que NAT1 est connecte a Switch1
# Verifier que Switch1 est connecte a FGT port1
# Verifier que virbr0 fonctionne : ip addr show virbr0
```

### Probleme : Ping inter-LAN ne fonctionne pas

```bash
# Verifier que le Docker Router est demare
# Verifier IP forwarding : cat /proc/sys/net/ipv4/ip_forward
# Verifier que les OVS bridges sont configures : ovs-vsctl show
# Verifier les routes sur les FGT : get router info routing-table all
```

### Probleme : HA ne s'active pas

```bash
# Verifier que les 2 FGT du meme cluster ont le meme group-name
# Verifier que port3 est connecte entre les 2 FGT
# Verifier les priorites : get system ha status
# Les 2 FGTs doivent etre sur la meme version FortiOS
```

### Probleme : Docker Router ne ping pas les FGTs

```bash
# Verifier les IPs : ip addr show
# Verifier que eth0 est sur OVS-LAN1 et eth1 sur OVS-LAN2
# Verifier que les OVS bridges incluent les ports du Docker Router
```

### Probleme : Services Docker inaccessibles

```bash
# Verifier les IPs configurees : ip addr show
# Verifier la route par defaut : ip route
# Verifier que le FGT a une politique autorisant le trafic
```

---

## Adressage IP complet

| Segment | Subnet | Passerelle | DHCP |
|---|---|---|---|
| WAN | 192.168.122.0/24 | 192.168.122.1 (virbr0) | virbr0/libvirt |
| LAN1 | 192.168.10.0/24 | 192.168.10.1 (FGT-P port2) | FGT-Primary (100-200) |
| LAN2 | 192.168.20.0/24 | 192.168.20.1 (FGT-S port2) | Alpine DHCP (100-200) |
| HA Heartbeat | 169.254.0.0/30 | — | Statique |
| IPsec Tunnel | 10.0.1.0/24 | — | — |

| Node | IP |
|---|---|
| FGT-Primary port1 | DHCP (192.168.122.x) |
| FGT-Primary port2 | 192.168.10.1/24 |
| FGT-Primary port3 | 169.254.0.1/30 |
| FGT-Primary-HA port1 | DHCP (192.168.122.x) |
| FGT-Primary-HA port2 | 192.168.10.2/24 |
| FGT-Primary-HA port3 | 169.254.0.2/30 |
| FGT-Secondary port1 | DHCP (192.168.122.x) |
| FGT-Secondary port2 | 192.168.20.1/24 |
| FGT-Secondary port3 | 169.254.0.3/30 |
| FGT-Secondary-HA port1 | DHCP (192.168.122.x) |
| FGT-Secondary-HA port2 | 192.168.20.2/24 |
| FGT-Secondary-HA port3 | 169.254.0.4/30 |
| Docker Router eth0 | 192.168.10.254/24 |
| Docker Router eth1 | 192.168.20.254/24 |
| Alpine DHCP | 192.168.20.2/24 |
| App-Server | 192.168.10.10/24 |
| PostgreSQL | 192.168.10.11/24 |
| Grafana | 192.168.20.11/24 |
| Prometheus | 192.168.20.12/24 |
| Traffic-Gen | 192.168.20.10/24 |
