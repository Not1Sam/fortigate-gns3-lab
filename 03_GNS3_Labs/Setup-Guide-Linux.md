# FortiGate Lab — Complete Setup Guide (Linux)

> A-to-Z guide for Linux hosts running GNS3 natively with QEMU/KVM.
> Follow these steps in order from a completely clean start.

---

## Step 0: Prerequisites

### 0.1 Install Required Software

**Ubuntu/Debian:**
```bash
sudo apt update && sudo apt install -y \
    gns3 gns3-server qemu-kvm libvirt-daemon-system virt-manager \
    docker.io telnet git python3
sudo systemctl enable --now libvirtd docker
sudo usermod -aG docker,kvm,libvirt $USER
# Log out and back in for group changes to take effect
```

**Arch Linux:**
```bash
sudo pacman -S gns3-server gns3-gui qemu-desktop libvirt docker telnet git python
sudo systemctl enable --now libvirtd docker
sudo usermod -aG docker,libvirt $USER
```

**Fedora:**
```bash
sudo dnf install -y gns3-server gns3-gui qemu-kvm libvirt docker telnet git python3
sudo systemctl enable --now libvirtd docker
sudo usermod -aG docker,libvirt $USER
```

### 0.2 Download Required Images

You need these image files before starting:

| Image | Source | Filename |
|---|---|---|
| **FortiGate 7.4.12** | Fortinet support site | `FGT_VM64_KVM-v7.4.12.F-build2622-outfile4.qcow2` |
| **Ubuntu 24.04 Cloud** | [cloud-images.ubuntu.com](https://cloud-images.ubuntu.com/noble/current/) | `noble-server-cloudimg-amd64.img` |

Place images in:
```bash
mkdir -p ~/GNS3/images/QEMU
mv FGT_VM64_KVM-*.qcow2 ~/GNS3/images/QEMU/
mv noble-server-cloudimg-amd64.img ~/GNS3/images/QEMU/
```

**FortiGate license (eval):**
1. Go to `https://forticloud.com` and register a free account
2. Navigate to **Product Registration** and register the FortiGate VM with the serial number from your image
3. You'll receive a license file (`.lic`). Two accounts needed if running both FGTs simultaneously (one eval per FGT)

### 0.3 Verify Host Readiness
```bash
# Check virtualization
egrep -c '(vmx|svm)' /proc/cpuinfo   # Should be > 0

# Check services
systemctl status libvirtd --no-pager | head -3
systemctl status docker --no-pager | head -3

# Check NAT network (virbr0)
ip addr show virbr0   # Should show 192.168.122.1/24

# If virbr0 missing, create it:
sudo virsh net-start default
sudo virsh net-autostart default
```

### 0.4 Enable NAT for Internet Access
```bash
# This allows VMs on virbr0 to reach the internet
sudo iptables -t nat -A POSTROUTING -s 192.168.122.0/24 ! -d 192.168.122.0/24 -j MASQUERADE
sudo iptables -A FORWARD -s 192.168.122.0/24 -j ACCEPT
```

### 0.5 Register Images in GNS3

**FortiGate template:**
1. Open GNS3 GUI → `Edit` → `Preferences` → `QEMU VMs`
2. Click `New` → Name: `FGT-Primary`
3. **RAM**: 2048 MB, **vCPUs**: 1, **Adapters**: 8
4. **Console type**: telnet
5. **Image**: Browse to `FGT_VM64_KVM-*.qcow2`
6. **Disk interface**: `virtio`, **Network adapter**: `virtio-net-pci`
7. Click `Finish`

**FGT-Secondary as linked clone:**
1. In `Edit` → `QEMU VMs`, select the FGT-Primary template
2. Click `Clone` → Name: `FGT-Secondary`
3. Enable **Linked clone** (uses the same base image, much smaller disk usage)

**Ubuntu Desktop template:**
1. `Edit` → `Preferences` → `QEMU VMs` → `New`
2. Name: `Ubuntu-Desktop-Client-1`
3. **RAM**: 2048 MB, **vCPUs**: 2, **Adapters**: 1
4. **Console type**: vnc
5. **Image**: Browse to `noble-server-cloudimg-amd64.img`
6. **Disk interface**: `virtio`, **Network adapter**: `e1000`
7. Click `Finish`

**Docker templates:** Create one for each Docker image:
1. `Edit` → `Preferences` → `Docker containers` → `New`
2. For each image below, create a template with 1 adapter and no start command:

| Template Name | Image | Adapters |
|---|---|---|
| `OpenvSwitch-1` | `gns3/openvswitch:latest` | 16 |
| `OpenvSwitch-2` | `gns3/openvswitch:latest` | 16 |
| `webterm-1` | `gns3/webterm:latest` | 1 |
| `Alpine-DHCP` | `alpine:latest` *(custom image, build later)* | 1 |
| `appServer-1` | `python:3.12-alpine` | 1 |
| `PostgreSQL-1` | `postgres:16-alpine` | 1 |
| `Grafana-1` | `grafana/grafana:latest` | 1 |
| `Prometheus-1` | `prom/prometheus:latest` | 1 |
| `Traffic-Gen-1` | `alpine:latest` | 1 |

For each Docker template:
- **Console type**: telnet (or vnc for Grafana/Prometheus/webterm)
- **Environment variables**: For PostgreSQL, add `POSTGRES_PASSWORD=gns3`
- **Start command**: Leave empty (will use image defaults)

**Built-in templates (no setup needed):**
- **VPCS**: Built into GNS3, just use `PC1`
- **Cloud**: Built-in, name it `NAT1`
- **Ethernet switch**: Built-in, name it `Switch1`

---

## Step 1: Create Project & Add Nodes

### 1.1 New Project
- GNS3 GUI → `File` → `New Blank Project`
- Name: `FortiGate Lab` or whatever you prefer
- Save location: default

### 1.2 Drag Nodes onto the Workspace

Drag each node from the **Devices Toolbar** (left side) onto the canvas:

**To find each node type:**
| Node | Toolbar Category | Template Name |
|---|---|---|
| NAT1 | **Clouds** | `NAT1` |
| Switch1 | **Switches** | `Switch1` |
| FGT-Primary | **QEMU VMs** | `FGT-Primary` |
| FGT-Secondary | **QEMU VMs** | `FGT-Secondary` |
| Ubuntu-Desktop-Client-1 | **QEMU VMs** | `Ubuntu-Desktop-Client-1` |
| PC1 | **VPCS** | `PC1` |
| OpenvSwitch-1 | **Docker containers** | `OpenvSwitch-1` |
| OpenvSwitch-2 | **Docker containers** | `OpenvSwitch-2` |
| webterm-1 | **Docker containers** | `webterm-1` |
| Alpine-DHCP | **Docker containers** | `Alpine-DHCP` |
| PostgreSQL-1 | **Docker containers** | `PostgreSQL-1` |
| grafana-1 | **Docker containers** | `Grafana-1` |
| Prometheus-1 | **Docker containers** | `Prometheus-1` |
| appServer-1 | **Docker containers** | `appServer-1` |
| Traffic-Gen-1 | **Docker containers** | `Traffic-Gen-1` |

Arrange them roughly in the topology layout (use the canvas in `Topology.canvas` as reference).

### 1.3 Configure NAT1 (Cloud Node)
1. Right-click NAT1 → `Configure`
2. Under **Cloud interfaces**, find your `virbr0` interface and check it
3. Click **OK**
4. This gives NAT1 access to the `192.168.122.0/24` network via libvirt

### 1.4 Wire It All Together

Click the **Link Mode** icon (or press `Ctrl+L`), then click each pair to connect:

**WAN chain:**
1. Click NAT1 → click Switch1 (creates link)
2. Click Switch1 → click FGT-Primary → select port `a0p0` (FGT port1)
3. Click Switch1 → click FGT-Secondary → select port `a0p0` (FGT port1)

**LAN1 (left side):**
4. Click FGT-Primary → click OpenvSwitch-1 → select FGT port `a1p0` (port2)
5. Click OpenvSwitch-1 → click PC1 → select OVS port `a1p0`
6. Click OpenvSwitch-1 → click webterm-1 → select OVS port `a2p0`
7. Click OpenvSwitch-1 → click appServer-1 → select OVS port `a3p0`
8. Click OpenvSwitch-1 → click PostgreSQL-1 → select OVS port `a4p0`

**LAN2 (right side):**
9. Click FGT-Secondary → click OpenvSwitch-2 → select FGT port `a1p0` (port2)
10. Click OpenvSwitch-2 → click Alpine-DHCP → select OVS port `a1p0`
11. Click OpenvSwitch-2 → click Ubuntu-Desktop-Client-1 → select OVS port `a2p0`
12. Click OpenvSwitch-2 → click Traffic-Gen-1 → select OVS port `a3p0`
13. Click OpenvSwitch-2 → click Grafana-1 → select OVS port `a4p0`
14. Click OpenvSwitch-2 → click Prometheus-1 → select OVS port `a5p0`

**Transit link:**
15. Click FGT-Primary → click FGT-Secondary → select FGT-Primary port `a2p0` (port3) and FGT-Secondary port `a2p0` (port3)

When connecting, GNS3 will ask which adapter/port to use for each node. Match the wiring table.

### 1.5 Verify Wiring

To double-check, right-click the workspace → `Show/Hide interface labels`. Every link should show the correct port numbers.

---

## Step 2: Start Nodes & Configure Cloud

### 2.1 Start Order

Start nodes in this order (critical for proper initialization):

```text
1. NAT1 (Cloud)
2. Switch1 (Ethernet switch)
3. FGT-Primary
4. FGT-Secondary
```

Wait for both FGTs to finish booting (1-2 minutes — you'll see the login prompt in their consoles).

Then start the rest:
```text
5. OpenvSwitch-1    7.  Ubuntu-Desktop-Client-1  9.  webterm-1
6. OpenvSwitch-2    8.  Alpine-DHCP               10. PostgreSQL-1
                    11-15. appServer-1, Grafana-1, Prometheus-1, PC1, Traffic-Gen-1
```

### 2.2 Open Consoles

Right-click each node → `Console`:

| Node | Console Type | What You'll See |
|---|---|---|
| FGT-Primary | Telnet | `FortiGate-VM64-KVM login:` |
| FGT-Secondary | Telnet | Same as above |
| Ubuntu Desktop | VNC | Graphical desktop (or terminal if cloud image) |
| OVS nodes | Telnet | `bash` shell |
| Docker containers | Telnet | Container shell |
| PC1 (VPCS) | Telnet | `PC1>` prompt |

### 2.3 Login to FGTs

At the `login:` prompt, type:
```
Username: admin
Password: (leave blank, press Enter)
```

On first boot, you'll be asked:
- Do you want to set a new password? → Type `n` (keep blank for now)

You should see the `#` prompt (config mode).

### 2.4 Apply Eval License

**Via Web UI (recommended):**
1. Get the WAN IP: from the FGT console, run `get system interface physical` and note port1's IP
2. Open a browser on your Linux host to `https://<wan-ip>`
3. Login: `admin` / no password
4. Click **System** → **FortiGuard** → **License** → **Upload**
5. Upload the `.lic` file from FortiCloud
6. The FGT will reboot

**Via CLI:**
```
execute restore config license tftp <filename> <tftp-server>
```
Or copy via SCP to the FGT, then:
```
execute restore config license <path>
```

Repeat for both FGTs (you need two FortiCloud accounts for two eval licenses).

### 2.5 Configure OVS Bridges

For each OVS node, right-click → Console (Telnet), then run:

**OVS-LAN1 (OpenvSwitch-1):**
```bash
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone
ovs-vsctl show
```

**OVS-LAN2 (OpenvSwitch-2):**
```bash
ovs-vsctl add-br br0
for port in eth0 eth1 eth2 eth3 eth4 eth5; do
    ovs-vsctl add-port br0 $port
done
ovs-vsctl set-fail-mode br0 standalone
ovs-vsctl show
```

Expected output: `bridge "br0"` with all ports listed.

### 2.6 Build Alpine DHCP Image

Before starting the Alpine-DHCP container, build the custom image:
```bash
docker build -t alpine-dhcp:latest - << 'EOF'
FROM alpine:latest
RUN apk add --no-cache dnsmasq
CMD ["sh", "-c", "echo 'interface=eth0' > /etc/dnsmasq.conf && echo 'dhcp-range=192.168.20.100,192.168.20.200,12h' >> /etc/dnsmasq.conf && echo 'dhcp-option=3,192.168.20.1' >> /etc/dnsmasq.conf && echo 'dhcp-option=6,8.8.8.8' >> /etc/dnsmasq.conf && ip addr add 192.168.20.2/24 dev eth0 && ip link set eth0 up && ip route add default via 192.168.20.1 && dnsmasq --no-daemon"]
EOF
```

Edit the `Alpine-DHCP` Docker template in GNS3 (`Edit` → `Preferences` → Docker containers → Alpine-DHCP → General settings`) and change the image to `alpine-dhcp:latest`.

### 2.7 Modify Ubuntu Image (One-Time)

If using a stock Ubuntu cloud image, it won't have persistent credentials. To set password `gns3` for user `ubuntu`:

```bash
# Install libguestfs-tools
sudo apt install libguestfs-tools   # Ubuntu/Debian
sudo pacman -S libguestfs           # Arch

# Mount and modify the image
sudo guestfish -a ~/GNS3/images/QEMU/noble-server-cloudimg-amd64.img << 'EOF'
  run
  mount /dev/sda1 /
  sh 'echo "ubuntu:$(openssl passwd -6 gns3)" | chpasswd -e'
  sh 'echo "ubuntu ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/ubuntu'
EOF
```

Or skip this and use the pre-modified image from this repo if available.

---

## Step 3: Configure FGT-Primary

### 3.1 Interfaces

Connect to FGT-Primary console (Telnet) and enter config mode:

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

### 3.4 Verify Internet
```
execute ping 8.8.8.8
```
Expected: `Reply from 8.8.8.8: bytes=56 time=Xms TTL=XXX`

If ping fails, check:
- NAT1 has internet (ping from host to 8.8.8.8)
- iptables NAT rules are active (Step 0.4)
- FGT port1 got an IP via DHCP (`get system interface physical`)

---

## Step 4: Configure FGT-Secondary

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

### 4.4 Verify Internet
```
execute ping 8.8.8.8
```

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

Open PC1 console (Telnet):
```
PC1> dhcp
PC1> show ip
```
Expected: `192.168.10.100` (or similar in the 100-200 range)

Open webterm-1 console (VNC) — it should auto-get an IP.
Test internet:
```
PC1> ping 8.8.8.8
```

### 5.3 Start Alpine DHCP (LAN2)

Start the `Alpine-DHCP` Docker node in GNS3. The container's CMD handles:
- Static IP `192.168.20.2/24` on eth0
- Default route via `192.168.20.1`
- dnsmasq serving `192.168.20.100`-`200`

Verify dnsmasq is running (from the Alpine console):
```bash
ps aux | grep dnsmasq
```

### 5.4 Test LAN2 DHCP

Open Ubuntu Desktop console (VNC) and login with `ubuntu` / `gns3`:
```bash
ip addr show enp2s0
```
Expected: `192.168.20.100` (or similar)

```bash
ping 8.8.8.8
```
Expected: Successful replies

---

## Step 6: Firewall Policies & NAT

### 6.1 FGT-Primary Policies

**LAN1 to WAN (internet access with SNAT):**
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

**Transit to WAN (internet for cross-LAN traffic):**
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

### 6.2 FGT-Secondary Policies

**LAN2 to WAN:**
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

**Transit to WAN:**
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

### 6.3 Verify SNAT

From Traffic-Gen console:
```bash
curl ifconfig.me
```
Expected: Shows FGT-Secondary's WAN IP (not the container's local IP)

---

## Step 7: OSPF Routing

### 7.1 Enable OSPF on FGT-Primary
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

### 7.2 Enable OSPF on FGT-Secondary
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

### 7.3 Inter-LAN Policies

**FGT-Primary (allow transit traffic into LAN1):**
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

**FGT-Secondary (allow transit traffic into LAN2):**
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

### 7.4 Verify OSPF

From FGT-Primary:
```
get router info ospf neighbor
```
Expected: Shows FGT-Secondary as neighbor (10.0.0.2)

```
get router info ospf route
```
Expected: Shows 192.168.20.0/24 learned via OSPF

```
execute ping 192.168.20.1
```
Expected: Successful — traffic goes through the transit link

---

## Step 8: Deploy Docker Services

### 8.1 PostgreSQL
```bash
docker exec GNS3.PostgreSQL-1.* ip addr add 192.168.10.11/24 dev eth0
docker exec GNS3.PostgreSQL-1.* ip link set eth0 up
docker exec GNS3.PostgreSQL-1.* ip route add default via 192.168.10.1
docker exec GNS3.PostgreSQL-1.* psql -U postgres -c "CREATE DATABASE appdb;"
```

### 8.2 App-Server (Flask)
```bash
docker exec GNS3.appServer-1.* ip addr add 192.168.10.10/24 dev eth0
docker exec GNS3.appServer-1.* ip link set eth0 up
docker exec GNS3.appServer-1.* ip route add default via 192.168.10.1
docker exec GNS3.appServer-1.* sh -c "pip install flask psycopg2-binary"
```

Deploy the app:
```bash
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

### 8.3 Grafana & Prometheus
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

### 8.4 Traffic-Gen
```bash
docker exec GNS3.Traffic-Gen-1.* ip addr add 192.168.20.12/24 dev eth0
docker exec GNS3.Traffic-Gen-1.* ip link set eth0 up
docker exec GNS3.Traffic-Gen-1.* ip route add default via 192.168.20.1
docker exec GNS3.Traffic-Gen-1.* apk add --no-cache curl busybox-extras
```

### 8.5 Verify Web Access
From webterm-1 (VNC console):
- Browse to `http://192.168.10.10/` → should see "Hello from App-Server!"
- Browse to `http://192.168.10.10/db` → should see "DB OK"
- Browse to `http://192.168.20.10:3000` → Grafana login page
- Browse to `http://192.168.20.11:9090` → Prometheus UI

---

## Step 9: Security Profiles

Apply these on **both FGTs** (same commands for Primary and Secondary):

### 9.1 Antivirus
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

### 9.2 IPS
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

### 9.3 Web Filter (Static URL)
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

### 9.4 App Control
```
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
```

### 9.5 SSL Inspection
```
config firewall ssl-ssh-profile
    edit "deep-inspection"
        set ssl-inspection deep-inspection
    next
end
```

### 9.6 Apply to Policy
On the LAN-to-WAN policy (policy ID 1):
```
config firewall policy
    edit 1
        set groups "default-av" "default-ips" "default-wf" "default-app"
        set ssl-ssh-profile "deep-inspection"
    next
end
```

---

## Step 10: IPsec VPN

### 10.1 Site-to-Site IPsec (FGT-Primary → OCI)

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

### 10.2 SSL VPN Portal
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

### 11.1 Deploy OCI Instance
1. Create an Oracle Cloud (or any cloud) Ubuntu 24.04 VM
2. Assign a public IP
3. Configure security group / firewall:

| Protocol | Port | Source | Purpose |
|---|---|---|---|
| UDP | 500 | FGT-Primary WAN IP | IPsec IKE |
| UDP | 4500 | FGT-Primary WAN IP | IPsec NAT-T |
| TCP | 80, 443 | Both FGT WAN IPs | Threat simulator |
| ICMP | — | Both FGT WAN IPs | Ping test |

### 11.2 Install Libreswan (OCI side)
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
```

### 11.3 Deploy Threat Simulator
```bash
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

### 11.4 Test IPsec Tunnel
From FGT-Primary:
```
get vpn ipsec tunnel details
```
Expected: Tunnel state = `up`

```
execute ping <OCI-private-IP>
```
Expected: Replies via the encrypted tunnel

---

## Step 12: Logging & Demo

### 12.1 Syslog Forwarding (both FGTs)
```
config log syslogd setting
    set status enable
    set server 192.168.20.12
    set port 514
    set facility local7
end
```

### 12.2 Verify Syslog
From Traffic-Gen console:
```bash
nc -ulvp 514
```
Expected: FGT log messages appear (after some traffic)

### 12.3 Demo Scenarios

Test each of these to verify the lab works end-to-end:

| # | Test | How | Expected Result |
|---|---|---|---|
| 1 | Internet access | `ping 8.8.8.8` from any client | Replies received |
| 2 | SNAT | `curl ifconfig.me` from Traffic-Gen | Shows FGT WAN IP |
| 3 | Inter-LAN | `ping 192.168.20.1` from FGT-Primary | Success via transit |
| 4 | App-Server | Browse `http://192.168.10.10/` | "Hello from App-Server!" |
| 5 | PostgreSQL | Browse `http://192.168.10.10/db` | "DB OK" |
| 6 | Grafana | Browse `http://192.168.20.10:3000` | Login page |
| 7 | Prometheus | Browse `http://192.168.20.11:9090` | UI loads |
| 8 | AV block | Curl `/eicar` from LAN client | FGT blocks with warning page |
| 9 | IPS block | Curl `/attack?sql=payload` | FGT blocks with warning |
| 10 | Web filter | Curl `/phishing` | FGT blocks with warning |
| 11 | IPsec | `get vpn ipsec tunnel details` | Tunnel is up |
| 12 | SSL VPN | Connect from Ubuntu via browser | VPN session established |

---

## Troubleshooting

### Lab won't start / nodes stuck
| Symptom | Cause | Fix |
|---|---|---|
| FGT boot loops | Insufficient RAM | Set to 2048 MB minimum |
| Docker nodes crash immediately | `su` binary missing in image | Use host `docker exec` or patch init.sh |
| OVS doesn't forward | No bridge configured | Run `ovs-vsctl add-br br0` commands |
| No DHCP on LAN1 | VCI match enabled by default | `set vci-match disable` in DHCP config |
| No DHCP on LAN2 | dnsmasq config empty | Rebuild alpine-dhcp image with CMD |
| Ubuntu gets 169.254.x.x | DHCP server unreachable | Check OVS bridge, check Alpine DHCP running |
| FGT can't ping internet | NAT not enabled on host | Run iptables commands from Step 0.4 |
| Can't reach web UIs | Wrong subnet routing | Verify OSPF neighbors show both routes |

### Complete Reset
To start completely from scratch:
1. Delete the GNS3 project (File → Delete Project)
2. Delete all Docker containers: `docker rm -f $(docker ps -aq)`
3. Recreate everything from Step 1
