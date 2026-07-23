# Open vSwitch (OVS) Reference

Living document for the GNS3 lab. Covers both general OVS and the `gns3/openvswitch` Docker appliance.

---

## What is OVS?

Open vSwitch is a production-quality, open-source multilayer virtual switch. It supports standard management interfaces (NetFlow, sFlow, IPFIX, RSPAN, CLI, LACP, 802.1ag) and the OpenFlow protocol for SDN control. It's the default network backend in OpenStack, KVM/Xen virtualization, and many NFV environments.

**In our lab:** OVS acts as the L2 fabric — replacing a physical managed switch. It handles 802.1Q VLAN trunking between FortiGates, VPCS endpoints, and future VMs.

---

## OVS Architecture (for context)

```
  ┌─────────────────────────────────────────┐
  │            ovs-vswitchd                  │  ← Userspace daemon, does switching logic
  │                                          │
  │  Bridge br0                              │
  │    ├─ Port eth0   (trunk, default)       │
  │    ├─ Port eth1   (access, tag=10)       │
  │    └─ Port eth2   (access, tag=20)       │
  └────────────────┬────────────────────────┘
                   │ ovsdb protocol
  ┌────────────────▼────────────────────────┐
  │          ovsdb-server                    │  ← Database daemon, stores config
  │  /etc/openvswitch/conf.db                │     (persists across reboots)
  └─────────────────────────────────────────┘
```

**Four main tools:**
| Tool | Purpose |
|------|---------|
| `ovs-vsctl` | Configure OVSDB — bridges, ports, VLANs, bonding |
| `ovs-ofctl` | Monitor/manage OpenFlow flow tables |
| `ovs-dpctl` | Admin datapath (kernel-level flows) |
| `ovs-appctl` | Runtime control of daemons — log levels, debug, hidden flows |

---

## `gns3/openvswitch` Docker Appliance

**Docker Hub:** `gns3/openvswitch:latest` (7.2 MB)
**Default config:** Ships with bridges `br0` through `br3` pre-created. All 16 ethernet interfaces are by default connected to `br0`.

**Environment variable:** `MANAGEMENT_INTERFACE=1` — if set, eth0 is NOT attached to any bridge (reserved for management).

**From the Docker Hub description:**
> "This container support 16 ethernet interface and is shipped with bridge from br0 to br3. By default all interface are connected to the br0."

**GNS3 Usage:** Add as a Docker container template in GNS3, set 16 adapters. Place in topology and connect cables. No initial configuration needed — it boots with all ports on `br0` acting as a flat switch.

**OVS-SNMP variant:** `ghcr.io/gns3/ovs-snmp` — same base but adds SNMP and LLDP support. Configurable via `NUM_BR=n` env var (default 1 bridge, all ports added to br0).

---

## Port Types

### Access Port
One VLAN, untagged traffic. The switch adds/removes 802.1Q tags automatically.

```
ovs-vsctl add-port br0 eth1 tag=10
ovs-vsctl set port eth1 tag=10        # change existing port
```

### Trunk Port
Carries multiple VLANs with 802.1Q tags. **Default for all OVS ports.**

```
ovs-vsctl add-port br0 eth0                              # trunk (all VLANs)
ovs-vsctl add-port br0 eth0 trunks=10,20,30              # trunk (restricted VLANs)
ovs-vsctl set port eth0 trunks=10,20,30                  # change existing port
```

### Native Untagged VLAN
A trunk port that also accepts untagged traffic on a specific VLAN (useful for hybrid ports).

```
ovs-vsctl set port eth0 vlan_mode=native-untagged tag=100
```

---

## Our Lab's OVS VLAN Design

```
                      ┌─────────────┐
  FGT-Primary port2 ──┤             │
                      │   OVS-Core  │
  FGT-Secondary ──────┤   (br0)     │
                      │              │
  VPCS-LAN-10 ────────┤              │
                      └─────────────┘
```

**Phase 2 plan:**
| OVS Port | Type | VLAN | Connected To |
|----------|------|------|-------------|
| eth1 (OVS port1) | Trunk: 10,20,30 | Tagged | FGT-Primary port2 |
| eth2 (OVS port2) | Trunk: 10,20,30 | Tagged | FGT-Secondary port1 |
| eth3 (OVS port3) | Access tag=10 | Untagged | VPCS-LAN-10 |

**Commands to configure this (on OVS console):**
```
# Remove ports from default br0 flat config
ovs-vsctl del-port br0 eth1
ovs-vsctl del-port br0 eth2
ovs-vsctl del-port br0 eth3

# Re-add with VLAN config
ovs-vsctl add-port br0 eth1 trunks=10,20,30
ovs-vsctl add-port br0 eth2 trunks=10,20,30
ovs-vsctl add-port br0 eth3 tag=10

# Verify
ovs-vsctl show
```

**Critical note from research (pitfall CP-03):** Misconfigured trunk/access ports on OVS are the #1 cause of "VLAN doesn't work." The OVS-to-FortiGate port must be a trunk (`trunks=10,20,30`), endpoint ports must be access (`tag=10`). Default OVS behavior is flat (no VLAN isolation) unless explicitly configured.

---

## VLAN Behavior

OVS implements **Independent VLAN Learning (IVL)** — each VLAN has its own MAC address table. A MAC learned on VLAN 10 is not assumed to be on VLAN 20 even on the same port.

**VLAN flow path:**
1. Packet arrives on access port (eth3, tag=10) — untagged
2. OVS inserts VLAN 10 tag internally
3. Packet forwarded out trunk port (eth1) with 802.1Q tag 10
4. FortiGate receives tagged packet on VLAN subinterface

**Reverse path:**
1. FortiGate sends tagged packet (VLAN 10) on trunk
2. OVS matches tag 10, strips tag
3. Forwards untagged to access port (eth3)

---

## OVS vs Linux Bridge

| Feature | OVS | Linux Bridge |
|---------|-----|-------------|
| OpenFlow support | Yes | No |
| VLAN trunking | Native (trunks=) | Via vconfig |
| VXLAN/GRE tunnels | Built-in | No |
| Flow-based forwarding | Yes | No |
| sFlow/NetFlow | Built-in | External tools |
| LACP/bonding | Native | Via bonding module |
| Complexity | Moderate | Simple |

**Why OVS for our lab:** VLAN trunking with explicit access/trunk port semantics matches real switch behavior better than Linux bridge. Also enables future SD-WAN monitoring (sFlow from OVS to analysis tools).

---

## Common Commands Cheat Sheet

### Configuration (ovs-vsctl)

| Command | What it does |
|---------|-------------|
| `ovs-vsctl show` | Overview of all bridges, ports, interfaces |
| `ovs-vsctl add-br br0` | Create a new bridge |
| `ovs-vsctl del-br br0` | Delete a bridge and all ports |
| `ovs-vsctl add-port br0 eth0` | Add port to bridge (as trunk) |
| `ovs-vsctl add-port br0 eth0 tag=10` | Add port as access port on VLAN 10 |
| `ovs-vsctl add-port br0 eth0 trunks=10,20` | Add port as trunk for VLANs 10,20 |
| `ovs-vsctl del-port br0 eth0` | Remove port from bridge |
| `ovs-vsctl list-br` | List all bridges |
| `ovs-vsctl list-ports br0` | List ports on a bridge |
| `ovs-vsctl list port eth0` | Show port details (VLAN config, etc.) |
| `ovs-vsctl list interface eth0` | Show interface details |
| `ovs-vsctl get port eth0 tag` | Get access VLAN tag of port |
| `ovs-vsctl get port eth0 trunks` | Get trunk VLAN list of port |
| `ovs-vsctl set port eth0 tag=20` | Change access VLAN on existing port |
| `ovs-vsctl set port eth0 trunks=10,20,30` | Change trunk VLANs on existing port |

### Flow Monitoring (ovs-ofctl)

| Command | What it does |
|---------|-------------|
| `ovs-ofctl dump-flows br0` | Show all OpenFlow flows |
| `ovs-ofctl dump-ports br0` | Show port statistics (packets, bytes, errors) |
| `ovs-ofctl dump-ports-desc br0` | Detailed port descriptions |
| `ovs-ofctl add-flow br0 "actions=normal"` | Add a flow rule (normal = L2 switching) |
| `ovs-ofctl del-flows br0` | Delete all flows (resets to defaults) |
| `ovs-ofctl snoop br0` | Monitor OpenFlow messages live |

### MAC Learning (ovs-appctl)

| Command | What it does |
|---------|-------------|
| `ovs-appctl fdb/show br0` | Show MAC address table (MAC→port mappings) |
| `ovs-appctl fdb/stats-show br0` | Show FDB statistics (learn/expire/evict rates) |
| `ovs-appctl fdb/stats-clear br0` | Reset FDB statistics |
| `ovs-appctl bridge/dump-flows br0` | Dump ALL flows including hidden (for debugging in-band) |
| `ovs-appctl dpif/dump-flows br0` | Dump kernel datapath flows (what's actually cached) |
| `ovs-appctl ofproto/trace br0 icmp,....` | Trace a packet through the pipeline |

### Datapath (ovs-dpctl)

| Command | What it does |
|---------|-------------|
| `ovs-dpctl show` | Show datapaths |
| `ovs-dpctl dump-flows` | Dump cached kernel flows (exact-match, recent traffic only) |

### Logging & Debug

| Command | What it does |
|---------|-------------|
| `ovs-appctl vlog/list` | List all log modules and their levels |
| `ovs-appctl vlog/set dpif_netdev:file:dbg` | Set log level for a module (dbg/info/warn/err) |
| `ovs-appctl coverage/show` | Show event coverage counters |
| `ovsdb-client dump` | Dump entire OVSDB contents |
| `ovs-vswitchd -V` | Show OVS version |

---

## Troubleshooting Scenarios

### Flat network (no VLAN isolation)
**Symptom:** All hosts can ping each other even though ports have different `tag=` values.
**Cause:** OVS is using its default "normal" switching which is flat. You need a controller or explicit OpenFlow rules to enforce VLAN isolation with `tag=`.
**Fix:** In standalone mode (no controller), `tag=` works correctly with the `normal` action. This is usually a configuration mistake — check that you actually removed ports from default bridge and re-added them with `tag=`. If the problem persists, verify with `ovs-vsctl list port eth3 | grep tag`.

### Port not showing up in bridge
**Symptom:** `ovs-vsctl show` doesn't list the expected port.
**Check:**
```
ovs-vsctl list-ports br0
```
**Fix:** Add the port explicitly:
```
ovs-vsctl add-port br0 eth1
```

### Port already on another bridge
**Symptom:** `ovs-vsctl add-port` gives "port already exists" or similar.
**Fix:** Remove from old bridge first:
```
ovs-vsctl del-port br1 eth1
ovs-vsctl add-port br0 eth1
```

### MAC table keeps growing rapidly
**Symptom:** high `mac_learning_learned` rate with many `mac_learning_evicted`.
**Diagnose:**
```
ovs-appctl fdb/stats-show br0
```
**If Current/maximum MAC entries shows 8192/8192 with high eviction count:** The table is wrapping — too many unique source MACs. Loop or broadcast storm possible. Check for accidental bridging loops.

### Flow table not matching
**Symptom:** `ovs-ofctl dump-flows br0` shows flows but traffic not matching.
**Check hidden flows:**
```
ovs-appctl bridge/dump-flows br0
```
Hidden flows (higher priority) may override your rules. Common with in-band control.

### No traffic across OVS at all
**Symptom:** `ovs-ofctl dump-ports br0` shows zero packets on all ports.
**Check:**
1. Are GNS3 links properly connected? (try removing and re-adding cable)
2. Is the OVS container running? (console should show login)
3. Do `ovs-vsctl show` — confirms br0 and ports exist
4. Try adding a flow that explicitly forwards:
```
ovs-ofctl add-flow br0 "in_port=1,actions=output:2"
```

---

## How OVS Fits in Our Lab (Phase-by-Phase)

| Phase | OVS Role |
|-------|----------|
| Phase 1 | Boot OVS container, verify 16-port br0, flat switching |
| Phase 2 | Configure VLAN trunk/access ports for 802.1Q |
| Phase 3 | (No change — OVS stays as L2 fabric) |
| Phase 4 | (No change — SD-WAN is on FortiGate, not OVS) |
| Phase 5 | (No change — NGFW is on FortiGate, not OVS) |

---

*Last updated: 2026-07-17*
*Sources: docs.openvswitch.org, Docker Hub (gns3/openvswitch), GNS3 docs, community guides*
