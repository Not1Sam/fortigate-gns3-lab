---
title: Agent Init File
tags:
  - agent/init
  - meta/guide
---

> [!warning] Read this first
> If you are an AI agent assisting with this project, you MUST follow this entry sequence.

## 1. Orientation

This vault documents a **FortiGate NGFW hybrid-cloud lab** built on GNS3 — the practical deliverable for a cybersecurity internship. 15 nodes: dual FortiGate 7.4.12 HA pair, OVS fabric, Docker clients, Ubuntu desktop, and an OCI cloud threat simulator.

### Must-read files (in order)

| File | Why |
|---|---|
| `[[Topology-Setup-Guide]]` | Step-by-step setup — start here to understand what's been done and what's next |
| `[[Full-Topology-Spec]]` | Complete blueprint — architecture, IPs, node specs, wiring |
| `[[Nodes-Reference]]` | Quick ref — node names, ports, images, credentials |
| `[[memory/facts]]` | Technical facts — eval limits, IP scheme, port maps, OCI endpoints |
| `[[memory/progress]]` | Current state — what's completed, what's next |
| `[[memory/decisions]]` | Architectural decisions with rationale |
| `[[memory/log]]` | Session history — what previous agents and the user have done |

### Also read the actual project file
GNS3 project at `/home/bingus/GNS3/projects/d311a72f-2416-4426-9138-96ccd23fe8fd/Internship.gns3`
Use `python3 -c "import json; ..."` to parse it and check actual wiring/node configs.

## 2. Key Facts

| Item | Value |
|---|---|
| Hypervisor | GNS3 on Arch Linux (QEMU/KVM + Podman) |
| FortiOS | 7.4.12 KVM, permanent eval (3 ports, 3 policies, 3 routes per FGT) |
| Ubuntu user/pass | `ubuntu` / `gns3` (NOPASSWD sudo) |
| Root password | `gns3` |
| WAN subnet | `192.168.122.0/24` (NAT1 / virbr0) |
| LAN1 | `192.168.10.0/24` (FGT-Primary port2) |
| LAN2 | `192.168.20.0/24` (FGT-Secondary port2) |
| HA link | `169.254.0.0/30` (port3 on both FGTs) |
| Clients on LAN1 | PC1 (VPCS), webterm-1 |
| Clients on LAN2 | Alpine-DHCP (dnsmasq server), Ubuntu-Desktop-Client-1 |
| GNS3 project path | `/home/bingus/GNS3/projects/d311a72f-2416-4426-9138-96ccd23fe8fd/Internship.gns3` |
| Vault path | `/home/bingus/obsidian-vaults/fortigate-lab` |
| Git repo | `https://github.com/Not1Sam/fortigate-gns3-lab` |

## 3. Agent Workflow

### Golden rules

1. **Always ask the user before running any command** — present each step, wait for confirmation
2. **Prefer Web UI for FortiGate config** — user explicitly requested this. Only use CLI if the Web UI can't do it
3. **Step by step** — one action at a time. Don't batch multiple configs
4. **Verify each step** — after every change, confirm it worked
5. **Update vault after completing work** — commit to `memory/progress.md` and `memory/log.md`
6. **Commit to GitHub** — vault changes go to the public repo

### Entry sequence

```
1. Read this file (_INIT_.md)
2. Read memory/progress.md → understand current state
3. Read memory/log.md → understand what happened in previous sessions
4. Check actual GNS3 project file → verify wiring matches docs
5. Read Topology-Setup-Guide.md → follow the step sequence
6. Ask user "what do you want to tackle next?"
```

### Before starting work

Check GNS3 services:
```bash
gns3-control status
```
If NAT forwarding is partial, ask user to run `gns3-control forward-enable` (requires fingerprint sudo).

### While working

- **FGT Web UI**: `https://192.168.122.2` (Primary), `https://192.168.122.3` (Secondary). Login with `admin` / no password
- **FGT CLI**: SSH to the same IPs, or right-click → Console in GNS3
- **Alpine/OVS console**: right-click → Console (Telnet)
- **Ubuntu/Webterm VNC**: right-click → Console (VNC)
- **VPCS**: right-click → Console (Telnet)
- **ALWAYS check actual GNS3 wiring** before making assumptions — document may be stale

### After completing work

```bash
# Commit vault changes
cd /home/bingus/obsidian-vaults/fortigate-lab
git add -A && git commit -m "description"
git pull --rebase && git push
```

## 4. Common Pitfalls

| Issue | Cause | Fix |
|---|---|---|
| No DHCP on LAN | FGT DHCP has `vci-match enable` with `vci-string "FortiSwitch"` — blocks non-FortiSwitch clients | `set vci-match disable` |
| OVS not forwarding traffic | OVS Docker containers in GNS3 start with NO bridge configured | `ovs-vsctl add-br <name> && ovs-vsctl add-port <name> eth0 eth1 eth2` |
| `setup-interfaces` not found | Not available in minimal `alpine:latest` Docker image | Use `ip addr add` and `ip route` directly |
| `set dns-server1` fails | Wrong syntax on FortiOS 7.4.x | Use `set dns-service default` |
| Alpine container loses IP after restart | `ip addr add` is ephemeral | Need to make persistent or add to startup script |
| dnsmasq stops after container restart | Not configured as a service | Re-run `dnsmasq` after restart, or add to init |
| Ubuntu gets 169.254.x.x | DHCP server unreachable — likely OVS bridge not configured | Check `ovs-vsctl show` on the OVS node |

## 5. Quick Access

### GNS3 control
```bash
gns3-control status          # check services
gns3-control forward-enable  # enable NAT (needs fingerprint)
gns3-control force-stop      # kill orphan processes
```

### Check current wiring
```bash
python3 -c "
import json
with open('/home/bingus/GNS3/projects/d311a72f-2416-4426-9138-96ccd23fe8fd/Internship.gns3') as f:
    data = json.load(f)
nodes = {n['node_id']: n['name'] for n in data['topology']['nodes']}
for link in data['topology']['links']:
    a = link['nodes'][0]; b = link['nodes'][1]
    print(f'{nodes[a[\"node_id\"]]:25s} a{a[\"adapter_number\"]}p{a[\"port_number\"]} <--> {nodes[b[\"node_id\"]]:25s} a{b[\"adapter_number\"]}p{b[\"port_number\"]}')
"
```

### Check OVS bridge config
From the OVS console:
```bash
ovs-vsctl show
```

### Check dnsmasq is running (Alpine)
```bash
ps aux | grep dnsmasq
```

### DHCP request from Ubuntu
```bash
sudo dhcpcd enp2s0
```

> [!tip] Credentials
> All Docker containers use default credentials. Ubuntu VM: `ubuntu` / `gns3`. FGT web: `admin` / no password (set at first boot).
