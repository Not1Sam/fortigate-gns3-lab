# FortiGate Lab — Complete Setup Guide (Windows)

> A-to-Z guide for Windows hosts using GNS3 VM (VMware or VirtualBox).
> Follow these steps in order from a completely clean start.

---

## Step 0: Prerequisites

### 0.1 Install Required Software

| Software | Download | Purpose |
|---|---|---|
| **GNS3 GUI** | [gns3.com](https://gns3.com) | Main GNS3 interface |
| **VMware Workstation** (recommended) or **VirtualBox** | vmware.com / virtualbox.org | Runs the GNS3 VM |
| **GNS3 VM** | gns3.com → Downloads → GNS3 VM | Runs QEMU + Docker for you |
| **PuTTY** | putty.org | SSH/Telnet console access |
| **7-Zip** | 7-zip.org | Extract QCOW2 images |
| **Git for Windows** (optional) | git-scm.com | Git operations |

**Windows features to enable:**
- Open **Control Panel** → **Programs** → **Turn Windows features on/off**
- Enable **Telnet Client** (for VPCS console access)

### 0.2 GNS3 VM Setup (Critical Step)

The GNS3 VM is a Linux VM that runs ALL your QEMU and Docker workloads. The Windows host only runs the GUI.

**Step-by-step VMware:**
1. Download the GNS3 VM `.ova` from gns3.com
2. Open VMware → `File` → `Open` → select the `.ova` → Import
3. Before starting: right-click the VM → `Settings`:
   - **Memory**: 8192 MB (8 GB) minimum
   - **Processors**: 4 vCPUs
   - **Network adapter**: NAT (VMnet8)
   - **Virtualization**: Enable "Virtualize Intel VT-x/EPT or AMD-V/RVI" under Processors
4. Start the VM — it gets an IP via DHCP on the NAT network
5. Note the IP shown on the GNS3 VM console

**VirtualBox (if not using VMware):**
1. File → Import Appliance → select `.ova`
2. VM Settings: 8192 MB RAM, 4 CPUs, Network: NAT
3. Enable VT-x/AMD-V in System → Acceleration

### 0.3 Connect GNS3 GUI to GNS3 VM

1. Open **GNS3 GUI**
2. `Edit` → `Preferences` → `GNS3 VM`
3. Check `Enable the GNS3 VM`
4. Select **VMware** (or VirtualBox)
5. Click `Auto-discover` — it should find the running GNS3 VM
6. Click **OK** — GNS3 will now use the VM for all emulation

To verify: the GNS3 status bar (bottom) should show `GNS3 VM: Running`.

### 0.4 Download Required Images

On the Windows host, create these folders:
```
C:\Users\<you>\GNS3\images\QEMU\
C:\Users\<you>\GNS3\images\Docker\
```

| Image | Source | Filename |
|---|---|---|---|
| **FortiGate 7.4.12** | Fortinet support site | `FGT_VM64_KVM-v7.4.12.F-build2622-outfile4.qcow2` |
| **FortiGate 7.0.9 Pre-Licensed** _(alternative)_ | Pre-activated image | `fortios.qcow2` (no FortiCloud registration needed) |
| **Ubuntu 24.04 Cloud** | [cloud-images.ubuntu.com](https://cloud-images.ubuntu.com/noble/current/) | `noble-server-cloudimg-amd64.img` |

Place both in `C:\Users\<you>\GNS3\images\QEMU\`.

### 0.5 FortiGate License

**For 7.4.12 (eval):**
1. Register a FortiCloud account at `https://forticloud.com`
2. Register the VM serial number (found in the QCOW2 image properties or on first boot)
3. Download the `.lic` license file
4. You'll need **two FortiCloud accounts** if running both FGTs simultaneously (one eval per FGT)

**For 7.0.9 pre-licensed (`fortios.qcow2`):**
- No registration needed. Copy to `C:\Users\<you>\GNS3\images\QEMU\` and create the template.
- Each booted instance gets a valid eval license automatically.

### 0.6 Find Your Cloud Interface

On Windows with GNS3 VM, the Cloud node needs the right virtual interface:
1. Open **Command Prompt** (cmd.exe) or PowerShell
2. Run `ipconfig /all`
3. Look for a **VMware VMnet8** adapter (or VirtualBox Host-Only adapter)
4. Note its IP — this is the gateway for your lab's WAN (e.g., `192.168.122.1`)

---

## Step 1: Create Project & Add Nodes

### 1.1 New Project
- GNS3 GUI → `File` → `New Blank Project`
- Name: `FortiGate Lab`

### 1.2 Create GNS3 Templates

**FortiGate QEMU template:**
1. `Edit` → `Preferences` → `QEMU VMs` → `New`
2. Name: `FGT-Primary`
3. **RAM**: 2048 MB, **vCPUs**: 1, **Adapters**: 8, **Console**: telnet
4. **Image**: Browse on the GNS3 VM file system or use the path to the QCOW2
5. **Disk interface**: `virtio`, **Network adapter**: `virtio-net-pci`
6. Click `Finish`

**FGT-Secondary (linked clone):**
1. In Preferences → QEMU VMs, select `FGT-Primary`
2. Click `Clone` → Name: `FGT-Secondary`
3. Enable `Linked clone`

**Ubuntu Desktop:**
1. `Edit` → `Preferences` → `QEMU VMs` → `New`
2. Name: `Ubuntu-Desktop-Client-1`
3. **RAM**: 2048 MB, **vCPUs**: 2, **Adapters**: 1, **Console**: vnc
4. **Image**: `noble-server-cloudimg-amd64.img`
5. Click `Finish`

**Docker templates:** Create one for each Docker container:
1. `Edit` → `Preferences` → `Docker containers` → `New`

| Template Name | Image | Adapters | Console |
|---|---|---|---|
| OpenvSwitch-1 | `gns3/openvswitch:latest` | 16 | Telnet |
| OpenvSwitch-2 | `gns3/openvswitch:latest` | 16 | Telnet |
| webterm-1 | `gns3/webterm:latest` | 1 | VNC |
| Alpine-DHCP | `alpine:latest` *(custom, build later)* | 1 | Telnet |
| appServer-1 | `python:3.12-alpine` | 1 | Telnet |
| PostgreSQL-1 | `postgres:16-alpine` | 1 | Telnet |
| Grafana-1 | `grafana/grafana:latest` | 1 | VNC |
| Prometheus-1 | `prom/prometheus:latest` | 1 | VNC |
| Traffic-Gen-1 | `alpine:latest` | 1 | Telnet |

For PostgreSQL, add environment variable: `POSTGRES_PASSWORD=gns3`.

### 1.3 Drag Nodes onto the Workspace

From the **Devices Toolbar** (left side), drag each node:

| Node | Toolbar Category | Template |
|---|---|---|
| NAT1 | **Clouds** | Create new: name `NAT1`, select the VMnet8 interface |
| Switch1 | **Switches** | `Ethernet switch` |
| FGT-Primary | **QEMU VMs** | `FGT-Primary` |
| FGT-Secondary | **QEMU VMs** | `FGT-Secondary` |
| Ubuntu Desktop | **QEMU VMs** | `Ubuntu-Desktop-Client-1` |
| PC1 | **VPCS** | `PC1` |
| OpenvSwitch-1/2 | **Docker** | `OpenvSwitch-1` / `OpenvSwitch-2` |
| Alpine-DHCP | **Docker** | `Alpine-DHCP` |
| webterm-1 | **Docker** | `webterm-1` |
| appServer-1 | **Docker** | `appServer-1` |
| PostgreSQL-1 | **Docker** | `PostgreSQL-1` |
| Grafana-1 | **Docker** | `Grafana-1` |
| Prometheus-1 | **Docker** | `Prometheus-1` |
| Traffic-Gen-1 | **Docker** | `Traffic-Gen-1` |

### 1.4 Configure NAT1 (Cloud Node)
1. Right-click NAT1 → `Configure`
2. Under **Cloud interfaces**, check the VMware VMnet8 adapter
3. Click **OK**

### 1.5 Wire Everything

Click **Link Mode** (`Ctrl+L`), then click pairs:

**WAN chain:**
1. NAT1 → Switch1
2. Switch1 → FGT-Primary (select port `a0p0` = port1)
3. Switch1 → FGT-Secondary (select port `a0p0` = port1)

**LAN1 (OVS-LAN1 = OpenvSwitch-1):**
4. FGT-Primary → OpenvSwitch-1 (FGT port `a1p0` = port2)
5. OpenvSwitch-1 → PC1 (OVS port `a1p0`)
6. OpenvSwitch-1 → webterm-1 (OVS port `a2p0`)
7. OpenvSwitch-1 → appServer-1 (OVS port `a3p0`)
8. OpenvSwitch-1 → PostgreSQL-1 (OVS port `a4p0`)

**LAN2 (OVS-LAN2 = OpenvSwitch-2):**
9. FGT-Secondary → OpenvSwitch-2 (FGT port `a1p0` = port2)
10. OpenvSwitch-2 → Alpine-DHCP (OVS port `a1p0`)
11. OpenvSwitch-2 → Ubuntu-Desktop-Client-1 (OVS port `a2p0`)
12. OpenvSwitch-2 → Traffic-Gen-1 (OVS port `a3p0`)
13. OpenvSwitch-2 → Grafana-1 (OVS port `a4p0`)
14. OpenvSwitch-2 → Prometheus-1 (OVS port `a5p0`)

**Transit link:**
15. FGT-Primary → FGT-Secondary (both use port `a2p0` = port3)

---

## Step 2: Start Nodes & Configure

### 2.1 Start Order
```
1. NAT1 (Cloud)        3. FGT-Primary          5. OpenvSwitch-1
2. Switch1             4. FGT-Secondary        6. OpenvSwitch-2
```
Then start the rest (Alpine-DHCP, Ubuntu, Docker nodes, PC1).

### 2.2 Open Consoles

| Node | Right-click → Console type |
|---|---|
| FGT | Telnet (opens PuTTY or Windows telnet) |
| Ubuntu Desktop | VNC |
| Docker | Telnet |
| VPCS | Telnet |

### 2.3 Login to FGTs

At the `login:` prompt:
```
Username: admin
Password: (press Enter, leave blank)
```

When asked to set password, type `n` (keep blank for now).

### 2.4 Apply Eval License
1. From the FGT console: `get system interface physical` → note port1's IP
2. Open a browser on Windows to `https://<wan-ip>`
3. Login: `admin` / no password
4. **System** → **FortiGuard** → **License** → **Upload** → select your `.lic` file
5. The FGT will reboot. Repeat for the second FGT.

### 2.5 Configure OVS Bridges

Right-click each OVS node → Console (Telnet):

**OVS-LAN1:**
```bash
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone
```

**OVS-LAN2:**
```bash
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4 eth5; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone
```

### 2.6 Build Alpine DHCP Image (on GNS3 VM)

SSH into the GNS3 VM to build the custom Docker image:
```bash
# From PowerShell or Putty
ssh gns3@<GNS3-VM-IP>   # password: gns3

# Inside the GNS3 VM:
docker build -t alpine-dhcp:latest - << 'EOF'
FROM alpine:latest
RUN apk add --no-cache dnsmasq
CMD ["sh", "-c", "echo 'interface=eth0' > /etc/dnsmasq.conf && echo 'dhcp-range=192.168.20.100,192.168.20.200,12h' >> /etc/dnsmasq.conf && echo 'dhcp-option=3,192.168.20.1' >> /etc/dnsmasq.conf && echo 'dhcp-option=6,8.8.8.8' >> /etc/dnsmasq.conf && ip addr add 192.168.20.2/24 dev eth0 && ip link set eth0 up && ip route add default via 192.168.20.1 && dnsmasq --no-daemon"]
EOF
```

Back in GNS3 GUI: `Edit` → `Preferences` → Docker containers → Alpine-DHCP → change image to `alpine-dhcp:latest`.

### 2.7 Modify Ubuntu Image (One-Time)

You need the Ubuntu cloud image to have password `gns3` for user `ubuntu`.

**Option A: Use the pre-modified image from this repo** (if available).

**Option B: Modify via the GNS3 VM:**
```bash
ssh gns3@<GNS3-VM-IP>
sudo apt update && sudo apt install -y libguestfs-tools

# Path to the Ubuntu image on the GNS3 VM
# (usually /opt/gns3/images/QEMU/ or similar)
sudo guestfish -a /path/to/noble-server-cloudimg-amd64.img << 'EOF'
  run
  mount /dev/sda1 /
  sh 'echo "ubuntu:$(openssl passwd -6 gns3)" | chpasswd -e'
  sh 'echo "ubuntu ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/ubuntu'
EOF
```

---

---

## Step 3: Configure FGT-Primary

Connect via right-click → Console (Telnet). At the `#` prompt, paste each block.

**Interfaces, route, and DNS:**
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
config router static
    edit 1
        set device port1
        set gateway 192.168.122.1
    next
end
config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end
```

**Verify:** `execute ping 8.8.8.8` — should get replies.

---

## Step 4: Configure FGT-Secondary

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
config router static
    edit 1
        set device port1
        set gateway 192.168.122.1
    next
end
config system dns
    set primary 8.8.8.8
    set secondary 1.1.1.1
end
```

**Verify:** `execute ping 8.8.8.8`

---

## Step 5: LAN Services

### 5.1 DHCP on FGT-Primary (LAN1)
> **Important**: FortiGate DHCP has `vci-match enable` by default, which blocks non-FortiSwitch devices.
> Add `set vci-match disable` if clients don't get IPs.

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
        set lease-time 86400
    next
end
```

### 5.2 Test LAN1 DHCP
Open PC1 console: `dhcp` then `show ip` — should show 192.168.10.x.

### 5.3 Alpine DHCP (LAN2)
The custom Alpine-DHCP container handles DHCP for LAN2 automatically (built in Step 2.6).
Start the Alpine-DHCP node, then verify from Ubuntu:
```bash
ip addr show enp2s0   # Should show 192.168.20.x
ping 8.8.8.8
```

---

## Step 6: Firewall Policies & NAT

**FGT-Primary:**
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

**FGT-Secondary:**
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

**Verify SNAT:** `curl ifconfig.me` from Traffic-Gen console → shows FGT-Secondary's WAN IP.

---

## Step 7: OSPF Routing

**FGT-Primary:**
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

**FGT-Secondary:**
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

**Inter-LAN policies:**
```
# On FGT-Primary — Transit-to-LAN1
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

# On FGT-Secondary — Transit-to-LAN2
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

**Verify:** `get router info ospf neighbor` from either FGT.

---

## Step 8: Docker Services

On Windows, configure Docker container IPs via **right-click → Console (Telnet)** on each container, then run:

**PostgreSQL-1:**
```bash
ip addr add 192.168.10.11/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.10.1
```
Then create the database (from PostgreSQL container console):
```bash
psql -U postgres -c "CREATE DATABASE appdb;"
```

**appServer-1:**
```bash
ip addr add 192.168.10.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.10.1
pip install flask psycopg2-binary
```

Then deploy the Flask app (from appServer-1 console):
```bash
cat > /app.py << 'EOF'
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
EOF
python /app.py &
```

**Grafana-1:**
```bash
ip addr add 192.168.20.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
```

**Prometheus-1:**
```bash
ip addr add 192.168.20.11/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
```

**Traffic-Gen-1:**
```bash
ip addr add 192.168.20.12/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
apk add --no-cache curl busybox-extras
```

**Verify:** Open webterm-1 VNC console, browse to `http://192.168.10.10/`.

---

## Step 9: Security Profiles

Apply these on **both FGTs** via their Telnet consoles:

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

## Step 10: VPN

**IPsec Site-to-Site (FGT-Primary → OCI):**
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

**SSL VPN Portal:**
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

## Step 11: OCI Cloud Integration

1. Deploy Ubuntu 24.04 on Oracle Cloud (or any cloud provider)
2. Assign a public IP
3. Open security group: UDP 500/4500 from FGT WAN IPs (IPsec), TCP 80/443 from FGT WAN IPs (threat sim)

**On the OCI instance:**
```bash
ssh ubuntu@<oci-public-ip>
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
sudo apt install -y python3-flask socat

sudo tee /opt/threat-sim/app.py << 'EOF'
from flask import Flask, request
app = Flask(__name__)
@app.route('/')
def index(): return 'OCI Threat Simulator'
@app.route('/eicar')
def eicar(): return 'X5O!P%@AP[4\\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*'
@app.route('/attack')
def attack():
    sql = request.args.get('sql', '')
    xss = request.args.get('xss', '')
    if sql: return f'SIMULATED SQLi: {sql}'
    if xss: return f'SIMULATED XSS: {xss}'
    return 'No payload'
@app.route('/phishing')
def phishing(): return '<html><title>phishing-login</title><body>Fake Bank Login</body></html>'
@app.route('/inspect')
def inspect(): return f'Your IP: {request.remote_addr}'
@app.route('/hacking-tools')
def hacking(): return 'Blocked tools page'
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
EOF

python3 /opt/threat-sim/app.py &
```

**Verify tunnel:** From FGT-Primary: `get vpn ipsec tunnel details` → should show `up`.

---

## Step 12: Logging & Demo

**Syslog forwarding (both FGTs):**
```
config log syslogd setting
    set status enable
    set server 192.168.20.12
    set port 514
    set facility local7
end
```

**Verify:** From Traffic-Gen console: `nc -ulvp 514` — logs appear after traffic.

**Demo scenarios:**
| # | Test | How | Expected Result |
|---|---|---|---|
| 1 | Internet access | `ping 8.8.8.8` from any client | Replies received |
| 2 | SNAT | `curl ifconfig.me` from Traffic-Gen | Shows FGT WAN IP |
| 3 | Inter-LAN | `ping 192.168.20.1` from FGT-Primary | Success via transit |
| 4 | App-Server | Browse `http://192.168.10.10/` | "Hello from App-Server!" |
| 5 | DB check | Browse `http://192.168.10.10/db` | "DB OK" |
| 6 | Grafana | Browse `http://192.168.20.10:3000` | Login page |
| 7 | Prometheus | Browse `http://192.168.20.11:9090` | UI loads |
| 8 | AV block | Curl `/eicar` from LAN client | FGT blocks with warning |
| 9 | IPS block | Curl `/attack?sql=payload` | FGT blocks with warning |
| 10 | Web filter | Curl `/phishing` | FGT blocks with warning |
| 11 | IPsec | `get vpn ipsec tunnel details` | Tunnel is up |
| 12 | SSL VPN | Connect from Ubuntu browser | VPN session established |

---

## Windows-Specific Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| GNS3 VM not detected | Firewall blocking | Allow GNS3 in Windows Defender Firewall |
| Can't SSH to GNS3 VM | Wrong network type | Change VM adapter from NAT to Bridged |
| Docker nodes show grayed out | Images not pulled on GNS3 VM | SSH into GNS3 VM, `docker pull <image>` |
| Cloud node has no interfaces | Wrong adapter selected | Reconfigure NAT1 with correct VMnet adapter |
| Lab is very slow | GNS3 VM under-resourced | Give VM 8+ GB RAM, 4+ vCPUs |
| VNC console blank | VNC viewer not installed | Install TightVNC or use GNS3 built-in viewer |
| Git operations fail | No Git installed | Install Git for Windows from git-scm.com |
| Can't find QCOW2 image in template | Path wrong | Images go in GNS3 VM's storage, not Windows host |

---

## Complete Reset

1. GNS3 GUI: `File` → `Delete Project`
2. In VMware: reset the GNS3 VM to snapshot
3. Start fresh from Step 1
