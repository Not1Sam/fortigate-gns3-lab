# Phase 1: GNS3 Lab Foundation — Context

**Gathered:** 2026-07-17
**Status:** Ready for planning
**Source:** Discuss-phase decisions + project initialization

<domain>
## Phase Boundary

Deliver a working GNS3 topology with both FortiGate 7.4.12 VMs licensed, OVS Docker appliance ready, WAN1 internet gateway working, and config backup/restore verified. Everything else builds on this.

</domain>

<decisions>
## Implementation Decisions

### Licensing
- Use permanent trial license (`execute vm-license` with FortiCloud account) — not 14-day eval (removed in 7.2.1+)
- Need two FortiCloud accounts (one trial per account)
- Each VM: exactly 1 vCPU, 2048 MB RAM — exceeding causes activation failure
- License persists across reboots; survives `execute restore config`

### Topology Design
- Two FortiGate 7.4.12 KVM VMs (FGT-Primary, FGT-Secondary)
- OVS Docker appliance (gns3/openvswitch:latest) as L2 fabric
- GNS3 NAT cloud as WAN1 (internet gateway)
- VPCS nodes for test endpoints
- All nodes connected via Ethernet links in GNS3

### Interface Budget Strategy
- Eval license limits to 3 interfaces per FortiGate
- Use VLAN subinterfaces on one physical port to stretch budget
- Phase 1: assign physical interfaces only (port1=WAN, port2=OVS trunk)
- VLANs configured in Phase 2

### Config Backup
- Use TFTP to GNS3 host for config backup/restore
- Document commands and verify both directions
- Essential for eval lifecycle management

### VPCS
- VPCS template registered in GNS3 database (server restarted)
- Binary at /usr/bin/vpcs v0.8.3
- Used for ping/traceroute testing only

</decisions>

<canonical_refs>
## Canonical References

### Project Documents
- `.planning/PROJECT.md` — Project context, role division
- `.planning/ROADMAP.md` — Phase 1 success criteria
- `.planning/REQUIREMENTS.md` — GNS3-01, GNS3-02, GNS3-03, OPS-01
- `.planning/research/SUMMARY.md` — Technical research summary
- `FORTIGATE-REFERENCE.md` — FortiGate command reference

</canonical_refs>

<specifics>
## Specific Ideas

- Boot both FortiGates, accept EULA, set admin password
- Activate trial licenses via `execute vm-license` with FortiCloud credentials
- Verify: `get system license | grep -i trial` shows permanent trial
- OVS boots with zero config — all 16 ports on `br0` bridge
- GNS3 NAT cloud auto-assigns 192.168.122.0/24 via libvirt DHCP

</specifics>

<deferred>
## Deferred Ideas

- WAN2 physical NIC bridge — TBD until host NIC feasibility checked
- FortiGate HA cluster — Phase 2+ feature
- VDOM mode — Phase 4 feature
- FortiManager — out of scope

</deferred>

---

*Phase: 01-gns3-lab-foundation*
*Context gathered: 2026-07-17*
