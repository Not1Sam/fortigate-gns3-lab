# FortiGate SD-WAN & OVS Lab

## What This Is

A hands-on GNS3 lab topology for the Coursera FortiGate Administrator course (Fortinet NSE 4). Dual FortiGate 7.4.12 VMs on a local workstation, connected through an OVS layer-2 fabric with 802.1Q VLANs, and linked to an OCI cloud VPS via dual IPsec tunnels under SD-WAN orchestration. Designed to practice every module in the course and serve as a reusable reference lab.

## Core Value

The lab must let me follow each Coursera module's exercises end-to-end on real FortiGate instances, with enough flexibility to explore my own scenarios beyond the course.

## Context

- **Motivation:** Mid-way through the Coursera FortiGate Administrator course (Fortinet, Inc.). Need a practical lab to cement concepts and finish the certification.
- **Host:** GNS3 running on a local workstation.
- **FortiGates:** Two FortiGate 7.4.12 VMs imported and licensed with free 14-day evaluation licenses.
- **OVS:** GNS3 OVS appliance for L2 switching and VLAN trunking.
- **Cloud:** OCI VPS already provisioned and accessible.
- **IP scheme:** 192.168.x private range for all lab subnets.
- **Linux comfort:** Comfortable with CLI — no GUI dependency for OVS or OCI management.
- **WAN bridge:** Physical NIC passthrough for the second WAN interface is TBD — may use NAT-only depending on host networking.
- **digitaraJobs mock API:** Placeholder endpoint, not a priority.
- **Timeline:** Target completion by end of week (July 19, 2026).
- **Scope approach:** Follow course module exercises first, then expand into free-form exploration.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Deploy two FortiGate VMs with active evaluation licenses in GNS3
- [ ] Deploy OVS appliance to handle LAN trunking
- [ ] Establish 802.1Q VLAN trunking between FortiGates and OVS
- [ ] Segment resources into VLAN 10 (LAN), VLAN 20 (AD/LDAP), VLAN 30 (Docker)
- [ ] Configure OCI Security Lists for IPsec (UDP 500/4500) and API ports
- [ ] Run lightweight Dockerized API service on OCI
- [ ] Configure dual WAN interfaces on primary FortiGate (NAT cloud + physical bridge)
- [ ] Establish dual IPsec tunnels from GNS3 to OCI
- [ ] Configure SD-WAN grouping both tunnels with performance SLA probe
- [ ] Implement SSL/TLS Deep Inspection with self-signed CAs
- [ ] Deploy App Control, Web Filtering, IPS on outbound/cloud policies
- [ ] Enable VDOMs (Edge-VDOM and Internal-VDOM)

### Out of Scope

- Production deployment — this is a learning lab only
- digitaraJobs mock API — placeholder, no real logic needed

## Constraints

- **Licenses:** FortiGate evaluation licenses expire after 14 days — lab must be reproducible
- **Bandwidth:** OCI instance bandwidth and egress costs apply
- **Host NIC:** Single physical NIC may limit WAN bridge options (TBD)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| GNS3 OVS appliance | Simple L2 fabric, no custom VM needed | — Pending |
| 192.168.x IP range | Familiar, simple, adequate for lab scale | — Pending |
| Study + explore scope | Course comes first, free-form after | — Pending |

---

*Last updated: 2026-07-15 after project initialization*
