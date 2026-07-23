---
title: Topology Setup Guide
tags:
  - lab/setup
  - guide
  - agent/reference
---

# Topology Setup Guide

Complete step-by-step guide to build, configure, and verify the full 15-node FortiGate hybrid-cloud lab. Designed to be followed by any AI agent assisting with this project.

> [!info] References
> Read these before starting: `[[Full-Topology-Spec.md]]`, `[[Nodes-Reference.md]]`, `[[memory/facts]]`, `[[memory/decisions]]`, `[[_INIT_]]`

> [!tip] Web UI preference
> When configuring FortiGate, always use the Web UI (`https://192.168.122.2` for Primary, `https://192.168.122.3` for Secondary) unless the user requests CLI. Login with `admin` / no password.

## Prerequisites

- GNS3 installed and running
- FortiGate 7.4.12 KVM image imported into GNS3 (`fgt-v7.4.12.qcow2`)
- Ubuntu 24.04 minimal cloud image imported — base image already pre-provisioned with `ubuntu` / `gns3`
- OVS Docker appliance available in GNS3
- Podman/Docker available on host
- OCI instance provisioned and reachable (Optional, for threat sim)

## Phase 1 — GNS3 Topology Build

### 1.1 Create GNS3 Project

- Project name: `Internship`
- Location: default GNS3 projects directory
- No template — start from blank project

### 1.2 Add Nodes

| QEMU VMs | RAM | vCPU | Adapters | Image |
|---|---|---|---|---|
| **FGT-Primary** | 2048 MB | 1 | 8 | `fgt-v7.4.12.qcow2` |
| **FGT-Secondary** | 2048 MB | 1 | 8 | linked clone of FGT-Primary |
| **Ubuntu-Desktop-Client** | 2048 MB | 2 | 1 (e1000) | `ubuntu-24.04-minimal-cloudimg-amd64.img` |

> [!note] Ubuntu credentials
> The base image has been modified to include a pre-created `ubuntu` user with password `gns3` and NOPASSWD sudo. Root password also `gns3`. Cloud-init is disabled from overwriting these. Any new linked clone inherits these credentials.

| Docker Nodes | Image | Adapters | Console |
|---|---|---|---|
| **OVS-LAN1** | `gns3/openvswitch:latest` | 16 | Telnet |
| **OVS-LAN2** | `gns3/openvswitch:latest` | 16 | Telnet |
| **webterm-1** | `gns3/webterm:latest` | 1 | VNC |
| **Alpine DHCP** | `alpine:latest` | 1 | Telnet |
| **App Server** | `python:3.12-alpine` | 1 | Telnet |
| **PostgreSQL** | `postgres:16-alpine` | 1 | Telnet |
| **Monitoring Stack** | `grafana/grafana` | 1 | VNC |
| **Traffic Gen + Syslog** | `alpine:latest` | 1 | Telnet |

| Built-in | Type | Purpose |
|---|---|---|
| **NAT1** | Cloud (virbr0) | WAN access — `192.168.122.1` |
| **PC1** | VPCS | Lightweight CLI client |

### 1.3 Wiring

#### WAN Segment

```
NAT1 (port0) --- WAN Switch --- FGT-Primary (port1)
                                WAN Switch --- FGT-Secondary (port1)
```

> [!note] WAN Switch
> A standard GNS3 Ethernet switch sits between NAT1 and both FGTs because NAT1 (virbr0) only has 1 port.

#### LAN1 (FGT-Primary side — 192.168.10.0/24)

```
FGT-Primary (port2) --- OVS-LAN1 (port1)
OVS-LAN1 (port2) --- Ubuntu-Desktop-Client (eth0)
OVS-LAN1 (port3) --- webterm-1 (eth0)
OVS-LAN1 (port4) --- PC1
OVS-LAN1 (port5) --- App Server (eth0)     [Docker — future]
OVS-LAN1 (port6) --- PostgreSQL (eth0)     [Docker — future]
OVS-LAN1 (port7) --- Monitoring (eth0)     [Docker — future]
OVS-LAN1 (port8) --- Traffic Gen (eth0)    [Docker — future]
```

#### LAN2 (FGT-Secondary side — 192.168.20.0/24)

```
FGT-Secondary (port2) --- OVS-LAN2 (port1)
OVS-LAN2 (port2) --- Alpine DHCP (eth0)
```

#### HA Link

```
FGT-Primary (port3) --- FGT-Secondary (port3)   [169.254.0.0/30]
```

> [!tip] Verify wiring
> After wiring, compare against the connection map in `[[Full-Topology-Spec.md]]`

> [!note] DHCP DNS syntax
> On FortiOS 7.4.x, use `set dns-service default` (not `set dns-server1 <ip>`). This tells the DHCP server to hand out the same DNS servers the FGT itself uses.

### 1.4 Docker Image Pulls

Before starting Docker-based nodes, pull images on the host:

```bash
podman pull docker.io/gns3/openvswitch:latest
podman pull docker.io/gns3/webterm:latest
podman pull docker.io/alpine:latest
podman pull docker.io/python:3.12-alpine
podman pull docker.io/postgres:16-alpine
podman pull docker.io/grafana/grafana:latest
podman pull docker.io/prom/prometheus:latest
```

## Phase 2 — Start Nodes & Verify

### 2.1 Start All Nodes

Start all 10+ nodes in GNS3. Wait for each to finish booting before proceeding.

### 2.2 Verify Console Access

| Node | Console | Check |
|---|---|---|
| FGT-Primary | Telnet | CLI prompt `FGT#` |
| FGT-Secondary | Telnet | CLI prompt `FGT#` |
| Ubuntu-Desktop-Client | VNC | Login with `ubuntu` / `gns3` |
| webterm-1 | VNC | Browser UI |
| VPCS (PC1) | Telnet | `VPCS>` prompt |
| Alpine DHCP | Telnet | `localhost:~#` |
| OVS nodes | Telnet | `ovs` shell |

### 2.3 Verify GNS3 Host Services

```bash
gns3-control status
gns3-control forward-enable    # enables NAT for WAN access
```

## Phase 3 — Basic FGT Configuration

### 3.1 License FGT-Primary

1. Obtain a free 14-day eval license from FortiCloud (or use permanent eval if available)
2. On FGT-Primary:
```
execute factoryreset
```
3. After reboot, apply license token:
```
execute update-now
```
4. Verify:
```
get system license status
```

### 3.2 License FGT-Secondary

Repeat the same process with a different FortiCloud account (each license is per-account).

### 3.3 Initial FGT-Primary Setup

```
config system interface
  edit port1
    set mode dhcp
    set alias WAN
  next
  edit port2
    set ip 192.168.10.1/24
    set alias LAN1
    set allowaccess ping https ssh
  next
  edit port3
    set ip 169.254.0.1/30
    set alias HA
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

### 3.4 Initial FGT-Secondary Setup

```
config system interface
  edit port1
    set mode dhcp
    set alias WAN
  next
  edit port2
    set ip 192.168.20.1/24
    set alias LAN2
    set allowaccess ping https ssh
  next
  edit port3
    set ip 169.254.0.2/30
    set alias HA
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

Verify: `execute ping 8.8.8.8`

### 3.5 Verify Connectivity

From FGT-Primary:
```
execute ping 8.8.8.1
execute ping 192.168.10.2       # if Ubuntu client is booted
```

## Phase 4 — HA Cluster (A-P被动)

### 4.1 Configure FGT-Primary (Primary)

```
config system ha
  set group-name "FGT-HA"
  set mode a-p
  set password <ha-secret>
  set hbdev port3 100
  set session-pickup enable
  set override disable
  set priority 200
end
```

After apply, FGT-Primary may restart. Wait for it to come back up.

### 4.2 Configure FGT-Secondary (Secondary)

```
config system ha
  set group-name "FGT-HA"
  set mode a-p
  set password <ha-secret>
  set hbdev port3 100
  set session-pickup enable
  set override disable
  set priority 100
end
```

### 4.3 Verify HA

```
get system ha status
```

Expected: One node shows `master`, the other shows `slave`. Both show `group-name: FGT-HA`.

> [!note] HA config sync
> After HA is established, most config changes made on the primary are synced to the secondary automatically. Manage config from the primary only.

## Phase 5 — LAN Services

### 5.1 FGT-Primary DHCP (LAN1)

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
  next
end
```

### 5.2 Alpine DHCP Server (LAN2)

On Alpine DHCP container:

```bash
# Configure interface manually (setup-interfaces not available in minimal image)
ip addr add 192.168.20.2/24 dev eth0
ip link set eth0 up
ip route add default via 192.168.20.1

# Set DNS
echo "nameserver 8.8.8.8" > /etc/resolv.conf

# Install dnsmasq
apk update
apk add dnsmasq

# Configure
cat > /etc/dnsmasq.conf << 'EOF'
interface=eth0
dhcp-range=192.168.20.100,192.168.20.200,12h
dhcp-option=3,192.168.20.1
dhcp-option=6,8.8.8.8
EOF

# Start in foreground first to test (Ctrl+C to stop)
dnsmasq -d

# Then run in background:
dnsmasq
```

> [!note] Alpine Docker networking
> `setup-interfaces` is not available in the minimal `alpine:latest` Docker image. Use `ip` commands directly to configure the interface.

## Phase 6 — Policies & NAT

> [!warning] Policy budget
> Each FGT has only 3 policy slots. Design carefully.

### 6.1 FGT-Primary Policies

```
# Policy 1: LAN1 to WAN (SNAT, internet access)
config firewall policy
  edit 1
    set name "LAN1-to-WAN"
    set srcintf port2
    set dstintf port1
    set srcaddr "all"
    set dstaddr "all"
    set action accept
    set schedule "always"
    set service "ALL"
    set nat enable
  next
end
```

### 6.2 FGT-Secondary Policies

```
# Policy 1: LAN2 to WAN (SNAT, internet access)
config firewall policy
  edit 1
    set name "LAN2-to-WAN"
    set srcintf port2
    set dstintf port1
    set srcaddr "all"
    set dstaddr "all"
    set action accept
    set schedule "always"
    set service "ALL"
    set nat enable
  next
end
```

> [!tip] Web UI alternative
> Go to **Policy & Objects → Firewall Policy → Create New**. Same settings. Make sure NAT is checked under Firewall/Network Options.

## Phase 7 — Docker Service Nodes

### 7.1 App Server (Python/Flask)

Create a simple Flask app and serve it on port 80. Verify from LAN1 client.

### 7.2 PostgreSQL

Set up with a test database for the App Server to consume.

### 7.3 Monitoring Stack

Configure Prometheus to scrape FGT SNMP metrics. Configure Grafana dashboard.

### 7.4 Traffic Gen + Syslog

Configure the traffic generator to produce HTTP/HTTPS traffic to the App Server and OCI. Set up syslog forwarding from FGTs to this node.

## Phase 8 — OCI Cloud & IPsec VPN

### 8.1 Configure OCI Instance as Libreswan Endpoint

Set up the OCI instance with Libreswan, listening for IPsec connections from both FGTs.

### 8.2 Configure FGT IPsec Tunnels

```
config vpn ipsec phase1-interface
  edit "to-oci"
    set interface port1
    set ike-version 2
    set keylife 28800
    set peertype any
    set net-device enable
    set proposal aes128-sha1
    set remote-gw <OCI-public-IP>
    set psk <pre-shared-key>
  next
end
```

### 8.3 Configure SD-WAN

Route traffic to OCI through the IPsec tunnel based on application or destination rules.

## Phase 9 — UTM & Security Profiles

### 9.1 Antivirus

```
config antivirus profile
  edit "default-av"
    config http
      set options av
    end
    config ftp
      set options av
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
      next
    end
  next
end
```

### 9.3 Web Filter

```
config webfilter profile
  edit "default-wf"
    config static-url-filter
      set status enable
      edit 1
        set url "phishing.test.lab"
        set action block
      next
    end
  next
end
```

### 9.4 Apply to Policies

Attach UTM profiles to policy 1 on each FGT:

```
config firewall policy
  edit 1
    set utm-status enable
    set av-profile "default-av"
    set ips-sensor "default-ips"
    set webfilter-profile "default-wf"
  next
end
```

## Phase 10 — Verification & Demos

### 10.1 Connectivity Tests

| Test | Command | Expected |
|---|---|---|
| WAN access | `execute ping 8.8.8.1` | Reply |
| Web access | Browse from webterm or Ubuntu | Page loads |
| DHCP | Check client got IP in correct range | 192.168.10.x or 192.168.20.x |
| HA failover | Stop FGT-Primary, ping from LAN2 | Traffic continues |
| IPsec | `diagnose vpn ike gateway list` | Tunnel up |

### 10.2 UTM Demos

| Demo | Trigger | Expected |
|---|---|---|
| AV block | `curl http://<OCI>/eicar` | Block page |
| IPS block | `curl http://<OCI>/attack?sql=payload'` | IPS alert in logs |
| URL filter | `curl https://phishing.test.lab` | Block page |

### 10.3 SNAT Verification

Access OCI `/api/status` — the source IP should show the FGT's WAN IP (192.168.122.x), not the client's LAN IP.

## Agent Workflow Notes

- **Always ask the user before making changes** — present each step, wait for confirmation
- **Use GNS3 console** (Telnet) for FGT, Alpine, OVS config
- **Use VNC** for Ubuntu Desktop and webterm
- **Docker/Podman on host** for image management
- **FGT CLI** via `execute ssh` from GNS3 or direct Telnet to port in GNS3 (right-click → Console)
- **Update [[memory/progress]]** after completing each phase
- **After any GNS3 wiring change**, update the [[Topology.canvas]] to reflect it
- **After any IP/config change**, update [[Full-Topology-Spec.md]] and [[memory/facts]]

## Recovery

### GNS3 Control

```bash
gns3-control status          # check all services
gns3-control start           # start GNS3 server + docker + libvirtd
gns3-control forward-enable  # enable NAT forwarding
gns3-control force-stop      # kill orphan processes
```

### Backup Locations

| Item | Path |
|---|---|
| Original Ubuntu base image | `~/.local/share/gns3/images/QEMU/ubuntu-24.04-minimal-cloudimg-amd64.img.bak.*` |
| Original project overlay | `./project-files/qemu/<uuid>/hda_disk.qcow2.bak.*` |
| Git repo | `https://github.com/Not1Sam/fortigate-gns3-lab` |

