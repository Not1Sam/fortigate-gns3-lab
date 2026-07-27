# FortiGate Lab — Setup Guide (Windows)

> For Windows hosts using GNS3 VM (VMware or VirtualBox) to run QEMU/Docker.

## How GNS3 Works on Windows

GNS3 on Windows uses a **GNS3 VM** (a Linux VM) to run all QEMU and Docker workloads. The Windows host only runs the GNS3 GUI. This means:
- FortiGate VMs, Docker containers, and Ubuntu all run inside the GNS3 VM
- Console access is via GNS3 GUI right-click → Console
- Docker commands (`docker exec`) must run **inside** the GNS3 VM (SSH in or use the GNS3 VM console)
- The Windows host does NOT need Docker, libvirt, or QEMU installed

## Prerequisites

### Required Software
- **GNS3 GUI** 2.2.x+ from [gns3.com](https://gns3.com)
- **VMware Workstation** (recommended) or **VirtualBox**
- **GNS3 VM** appliance (download from gns3.com — `.ova` file)
- **PuTTY** or **Windows Terminal** (for SSH/telnet)
- 7-Zip or similar (for extracting QCOW2 images)
- Git for Windows (optional)

### Host Environment
- Windows 10/11 64-bit
- At least 8 GB RAM free (~7.3 GB for the lab + GNS3 VM overhead)
- Intel VT-x or AMD-V enabled in BIOS
- Hardware virtualization enabled in BIOS for the GNS3 VM

### GNS3 VM Setup
1. Download the GNS3 VM `.ova` from gns3.com
2. Import into VMware Workstation (File → Open → select `.ova`)
3. **VM Settings**:
   - vCPUs: 4
   - RAM: 8192 MB (8 GB) minimum
   - Network adapter: NAT (VMnet8) — this provides internet for the lab
   - Enable virtualization (VT-x/AMD-V) for the VM
4. Start the GNS3 VM — it will get an IP via DHCP on the NAT network
5. In GNS3 GUI: `Edit → Preferences → GNS3 VM` → enable → select VMware → auto-discover the VM

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
Open GNS3 GUI → File → New Project → name it.

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
| NAT1 | Cloud (VMnet) | — | — |
| Switch1 | Ethernet switch | — | 4 |

### 1.3 Cloud Node (NAT1)
On Windows with GNS3 VM, the Cloud node uses the GNS3 VM's NAT interface:
- If using **VMware**: use `VMnet8` (NAT network)
- If using **VirtualBox**: use the host-only adapter or NAT network
- The GNS3 VM's IP on this interface is the gateway (e.g., 192.168.122.1)
- Create a Cloud node template with the correct VMware/VirtualBox interface

**Finding the right interface:**
1. Right-click in GNS3 workspace → `Show GNS3 VM console` (or SSH into GNS3 VM)
2. Run `ip addr show` to see the GNS3 VM's IP
3. On Windows host, run `ipconfig` to find the matching VMware/VirtualBox adapter
4. Create a Cloud node pointing to that adapter

### 1.4 Console Access
| Node Type | Method |
|---|---|
| FGT (QEMU) | Right-click → Console (Telnet via PuTTY) |
| Docker | Right-click → Console (Telnet) |
| Ubuntu (QEMU) | Right-click → Console (VNC) |
| VPCS | Right-click → Console (Telnet) |

### 1.5 Docker Images
Docker images are managed by the GNS3 VM, not the Windows host. GNS3 will automatically pull images when you start a Docker node. If images fail to pull:
1. SSH into the GNS3 VM: `ssh gns3@<GNS3-VM-IP>` (password: `gns3`)
2. Manually pull: `docker pull <image-name>`
3. For custom images (alpine-dhcp), build directly on the GNS3 VM

**Building alpine-dhcp on the GNS3 VM:**
```bash
# SSH into the GNS3 VM first
ssh gns3@<GNS3-VM-IP>

# Then build
docker build -t alpine-dhcp:latest - << 'EOF'
FROM alpine:latest
RUN apk add --no-cache dnsmasq
CMD ["sh", "-c", "echo 'interface=eth0' > /etc/dnsmasq.conf && echo 'dhcp-range=192.168.20.100,192.168.20.200,12h' >> /etc/dnsmasq.conf && echo 'dhcp-option=3,192.168.20.1' >> /etc/dnsmasq.conf && echo 'dhcp-option=6,8.8.8.8' >> /etc/dnsmasq.conf && ip addr add 192.168.20.2/24 dev eth0 && ip link set eth0 up && ip route add default via 192.168.20.1 && dnsmasq --no-daemon"]
EOF
```

### 1.6 Wiring
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

### 1.7 OVS Bridge Setup
Right-click each OVS node → Console, then run:
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

### 1.8 Validate
- All 15 nodes visible and green in GNS3
- OVS bridges show all ports with `ovs-vsctl show`
- NAT1 has internet (confirm from a FGT console: `execute ping 8.8.8.8`)

---

## Phase 2: FGT Configuration

### 2.1 FGT-Primary
Via right-click → Console (Telnet). Enter config mode at `#` prompt.

**Port1 — WAN (DHCP):**
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

**Default Route + DNS:**
```
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

### 2.2 FGT-Secondary
**Port1 — WAN (DHCP):**
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

**Default Route + DNS:**
```
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

### 3.2 Alpine DHCP on LAN2
Build the image on the GNS3 VM (see Phase 1.5), then start the Alpine-DHCP Docker node in GNS3. The container's CMD handles IP config + dnsmasq automatically.

### 3.3 Verify Clients
- **PC1 (VPCS)**: enter `dhcp`, then `show ip`
- **Ubuntu Desktop**: right-click → Console (VNC), login `ubuntu`/`gns3`, check `ip addr`
- **Internet test**: `ping 8.8.8.8`

---

## Phase 4: Policies & NAT

### 4.1 FGT-Primary Policies
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

### 4.2 FGT-Secondary Policies
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

---

## Phase 5: OSPF Routing

### 5.1 FGT-Primary
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

### 5.2 FGT-Secondary
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
On FGT-Primary: `edit 3`, name `Transit-to-LAN1`, srcintf `port3`, dstintf `port2`
On FGT-Secondary: `edit 3`, name `Transit-to-LAN2`, srcintf `port3`, dstintf `port2`

### 5.4 Verify
```
get router info ospf neighbor
execute ping 10.0.0.2
execute ping 192.168.20.1
```

---

## Phase 6: Docker Services

> **Important:** Docker containers run inside the GNS3 VM. To run `docker exec` commands:
> - Option A: Right-click each Docker node → Console (Telnet) and configure IP manually
> - Option B: SSH into the GNS3 VM and run `docker exec` from there

### 6.1 Option A — Via Container Console (Recommended)
Right-click each container → Console, then run:
```bash
# PostgreSQL
ip addr add 192.168.10.11/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.10.1

# App-Server
ip addr add 192.168.10.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.10.1

# Grafana
ip addr add 192.168.20.10/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1

# Prometheus
ip addr add 192.168.20.11/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1

# Traffic-Gen
ip addr add 192.168.20.12/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1
```

### 6.2 Option B — Via GNS3 VM SSH
```bash
# SSH into GNS3 VM
ssh gns3@<GNS3-VM-IP>

# Find container IDs
docker ps | grep <container-name>

# Configure IP
docker exec <container-id> ip addr add <ip>/24 dev eth0
docker exec <container-id> ip link set eth0 up
docker exec <container-id> ip route add default via <gateway>
```

---

## Phase 7-10: Security, VPN, OCI, Logging

These phases are OS-independent — all config is on the FortiGates or the OCI instance. Follow the same CLI commands as the Linux guide:

- **Phase 7**: Security profiles on both FGTs
- **Phase 8**: IPsec + SSL VPN config
- **Phase 9**: OCI cloud deployment (same regardless of host OS)
- **Phase 10**: Syslog forwarding + demo scenarios

See `Device-Setup-Guide.md` for per-device commands, or `Setup-Guide-Linux.md` for detailed phase configs.

---

## Windows-Specific Notes

### Console Shortcuts
| Action | Method |
|---|---|
| Open console | Right-click node → Console |
| Open VNC | Right-click node → Console (uses built-in VNC or TightVNC) |
| SSH into GNS3 VM | `ssh gns3@<GNS3-VM-IP>` from PowerShell/WSL |
| Browse to web services | Open `http://<container-ip>:<port>` in Windows browser |

### File Paths
| Item | Windows Path |
|---|---|
| GNS3 project files | `C:\Users\<you>\GNS3\projects\` |
| QEMU images | `C:\Users\<you>\GNS3\images\QEMU\` |
| GNS3 config | `C:\Users\<you>\.config\GNS3\` |

### Troubleshooting
| Issue | Fix |
|---|---|
| GNS3 VM not detected | Check VMware/VirtualBox network adapter type. GNS3 VM must use NAT (VMnet8) |
| Docker containers won't start | SSH into GNS3 VM, run `docker pull <image>` manually |
| No internet in lab | Check GNS3 VM has internet (ping google.com from GNS3 VM console) |
| Cant reach container web UI from Windows | Use GNS3 VNC console instead, or add host route to container subnet |
