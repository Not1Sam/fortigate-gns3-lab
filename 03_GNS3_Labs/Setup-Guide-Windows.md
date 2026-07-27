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
|---|---|---|
| **FortiGate 7.4.12** | Fortinet support site | `FGT_VM64_KVM-v7.4.12.F-build2622-outfile4.qcow2` |
| **Ubuntu 24.04 Cloud** | [cloud-images.ubuntu.com](https://cloud-images.ubuntu.com/noble/current/) | `noble-server-cloudimg-amd64.img` |

Place both in `C:\Users\<you>\GNS3\images\QEMU\`.

### 0.5 FortiGate Eval License

1. Register a FortiCloud account at `https://forticloud.com`
2. Register the VM serial number (found in the QCOW2 image properties or on first boot)
3. Download the `.lic` license file
4. You'll need **two FortiCloud accounts** if running both FGTs simultaneously (one eval per FGT)

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

## Step 3-12: Configure the Lab

The FortiGate and service configuration steps are OS-independent. Follow the sections from the Linux guide:

| Step | What To Do | Linux Guide Section |
|---|---|---|
| **3** | FGT-Primary interfaces, route, DNS | Step 3.1-3.4 |
| **4** | FGT-Secondary interfaces, route, DNS | Step 4.1-4.4 |
| **5** | DHCP (FGT-P + Alpine), verify clients | Step 5.1-5.4 |
| **6** | Firewall policies, SNAT | Step 6.1-6.3 |
| **7** | OSPF routing, inter-LAN policies | Step 7.1-7.4 |
| **8** | Docker service IP config | Step 8.1-8.5 |
| **9** | Security profiles (AV, IPS, Web Filter) | Step 9.1-9.6 |
| **10** | IPsec + SSL VPN | Step 10.1-10.2 |
| **11** | OCI cloud deployment | Step 11.1-11.4 |
| **12** | Syslog + demo scenarios | Step 12.1-12.3 |

> **Docker note**: On Windows, `docker exec` commands must run from the GNS3 VM. Instead, right-click each Docker node → Console (Telnet) and configure IPs directly in the container shell:
> ```bash
> ip addr add <IP>/24 dev eth0
> ip link set eth0 up
> ip route add default via <gateway>
> ```

### Docker Service IPs to Set
| Container | IP | Gateway |
|---|---|---|
| PostgreSQL-1 | 192.168.10.11/24 | 192.168.10.1 |
| appServer-1 | 192.168.10.10/24 | 192.168.10.1 |
| Grafana-1 | 192.168.20.10/24 | 192.168.20.1 |
| Prometheus-1 | 192.168.20.11/24 | 192.168.20.1 |
| Traffic-Gen-1 | 192.168.20.12/24 | 192.168.20.1 |

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
