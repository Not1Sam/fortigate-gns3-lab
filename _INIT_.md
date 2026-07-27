---
title: Agent Init File
tags:
  - agent/init
  - meta/guide
---

> [!warning] Read this first
> If you are an AI agent assisting with this project, you MUST follow this entry sequence.

## 1. Orientation

This vault documents a **FortiGate NGFW hybrid-cloud lab** built on GNS3 — two independent FortiGates (no HA) with a transit link, separate LAN segments, Docker services, and an OCI cloud threat simulator.

### Layout
```
NAT1 ──Switch1──┬──FGT-Primary (port1 WAN)──port2──OVS-LAN1──[PC1, webterm, App-Server, PostgreSQL]
                 │                         └port3──10.0.0.1/30──┐
                 │                                               ├──Transit (OSPF)
                 └──FGT-Secondary (port1 WAN)──port2──OVS-LAN2──[Alpine DHCP, Ubuntu, Grafana, Prometheus, Traffic-Gen]
                                              └port3──10.0.0.2/30──┘
```

### Must-read files (in order)

| File | Why |
|---|---|
| `[[Topology-Setup-Guide]]` | Step-by-step setup — 10 phases, start here |
| `[[Full-Topology-Spec]]` | Complete blueprint — architecture, IPs, node specs, wiring |
| `[[Nodes-Reference]]` | Quick ref — node names, ports, images, credentials |
| `[[memory/facts]]` | Technical facts — eval limits, IP scheme, port maps, OCI endpoints |
| `[[memory/progress]]` | Current state — what's completed, what's next |
| `[[memory/decisions]]` | Architectural decisions with rationale |

For GNS3 project: `/home/bingus/GNS3/projects/d311a72f-2416-4426-9138-96ccd23fe8fd/Internship.gns3`

## 2. Key Facts

| Item | Value |
|---|---|
| Hypervisor | GNS3 on Arch Linux (QEMU/KVM + Podman) |
| FortiOS | 7.4.12 KVM, permanent eval (3 ports, 3 policies, 3 routes per FGT) |
| Ubuntu user/pass | `ubuntu` / `gns3` (NOPASSWD sudo) |
| WAN subnet | `192.168.122.0/24` (NAT1 / virbr0) |
| LAN1 | `192.168.10.0/24` (FGT-Primary port2) |
| LAN2 | `192.168.20.0/24` (FGT-Secondary port2) |
| Transit | `10.0.0.0/30` (port3 on both FGTs) |
| Inter-LAN routing | OSPF over transit link |
| GNS3 project path | `/home/bingus/GNS3/projects/d311a72f-2416-4426-9138-96ccd23fe8fd/Internship.gns3` |
| Vault path | `/home/bingus/obsidian-vaults/fortigate-lab` |
| Git repo | `https://github.com/Not1Sam/fortigate-gns3-lab` |
| per-user progress | `progress/{username}.md` |

## 3. Agent Workflow

### ⚠️ Golden Rule #1 — Always Ask Permission

**You MUST ask permission before making ANY of these changes:**
- Editing the GNS3 project file (`.gns3` JSON) — node properties, wiring, templates
- Building or modifying Docker images
- Modifying QEMU/VM disk images (overlays, base images)
- Writing config files inside GNS3 nodes (Alpine, Ubuntu, OVS, etc.)
- Running commands on the host (iptables, docker, podman, systemctl)
- Modifying FortiGate configuration (Web UI or CLI)
- Changing network topology, wiring, or adding/removing nodes
- Creating files or directories outside the Obsidian vault

**Wait for explicit "yes, do it" before proceeding. Show the exact command.**

### Golden rules

1. **Always ask permission** (see above) — present each step, wait for confirmation
2. **Prefer Web UI for FortiGate config** — only use CLI if the Web UI can't do it
3. **Step by step** — one action at a time
4. **Verify each step** — after every change, confirm it worked
5. **Read live state before assuming** — check GNS3 project file and node consoles
6. **Update vault after completing work** — commit to `memory/progress.md` and per-user `progress/{username}.md`
7. **Commit to GitHub** — vault changes go to the public repo

### Entry sequence

```
1. Read this file (_INIT_.md)
2. Read memory/facts.md → technical facts
3. Read memory/decisions.md → architectural decisions
4. Read memory/progress.md → current state overview
5. Read progress/{username}.md → per-user progress (if exists, create if not)
6. Check actual GNS3 project file → verify wiring matches docs
7. Read Topology-Setup-Guide.md → follow the phase sequence
8. ASK the user what they have done so far and what they want to tackle next
   → Do NOT blindly trust the progress file — the user may be ahead or behind
   → Let them tell you where they are, then verify against live state
```

### Setup wizard for new starters

When helping a **new user** start the lab for the first time:

1. **Detect OS**: Ask `uname -a` (Linux/Mac) or check for WSL/GNS3 VM
2. **Check prerequisites**:
   - ❌ GNS3 installed? → Ask user to install from gns3.com
   - ❌ FortiGate image? (`fgt-v7.4.12.qcow2`) → Provide download instructions
   - ❌ Docker? → `docker ps` to check
   - ❌ Ubuntu cloud image? → Provide URL
3. **Create user progress file**: `progress/{username}.md` from `progress/_template_.md`
4. **Guide through Phase 1** (Base Setup) → Phase 2 (FGT Config) → ...

### Before starting work

Check GNS3 services:
```bash
gns3-control status
```
If NAT forwarding is partial, ask user to run `gns3-control forward-enable` (requires fingerprint sudo).

### While working

- **FGT Web UI**: `https://<wan-ip>` (check per-user progress for WAN IPs)
- **FGT CLI**: SSH to WAN IP, or right-click → Console in GNS3
- **Alpine/OVS console**: right-click → Console (Telnet)
- **Ubuntu/Webterm VNC**: right-click → Console (VNC)
- **VPCS**: right-click → Console (Telnet)
- **WAN IPs**: FGTs use DHCP, IP may change on restart — check via Console / `get system interface physical`

### After completing work

```bash
cd /home/bingus/obsidian-vaults/fortigate-lab
git add -A && git commit -m "description"
git pull --rebase && git push
```
Update `progress/{username}.md` with new phase status.

## 4. Common Pitfalls

| Issue | Cause | Fix |
|---|---|---|
| No DHCP on LAN1 | FGT DHCP has `vci-match enable` with `vci-string "FortiSwitch"` — blocks non-FortiSwitch clients | `set vci-match disable` |
| OVS not forwarding | OVS starts with NO bridge configured | `ovs-vsctl add-br br0 && ovs-vsctl add-port br0 eth0 eth1 ...` |
| Alpine loses IP after restart | `ip addr add` is ephemeral | Create startup script or reconfigure in GNS3 |
| Ubuntu gets 169.254.x.x | DHCP server unreachable — OVS bridge not configured | `ovs-vsctl show` on OVS node |
| Docker container won't start | GNS3 init.sh `su` fails on stripped images (e.g. Grafana/Prometheus/alpine) | `su` binary missing or target user has `/bin/false` shell — init.sh patched on this host to fall back to root |
| PostgreSQL fails to start | data directory exists but is not empty | Clean data dir or use `POSTGRES_INITDB_ARGS` |
| `set dns-server1` fails | Wrong syntax on FortiOS 7.4.x | Use `set dns-service default` |
| HA config exceeds eval | HA needs 3+ interfaces but eval caps at 3 | Use two independent FGTs with transit link instead |

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
```bash
ovs-vsctl show
```

### Check dnsmasq is running (Alpine)
```bash
ps aux | grep dnsmasq
```

### Per-user progress
```bash
ls progress/
```
Create new: copy `progress/_template_.md` to `progress/{username}.md`

> [!tip] Credentials
> All Docker containers use default credentials. Ubuntu VM: `ubuntu` / `gns3`. FGT web: `admin` / no password.
