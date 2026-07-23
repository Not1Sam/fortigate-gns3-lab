# FortiGate SD-WAN & OVS Fabric — Internship Project

## What This Is

An enterprise-grade hybrid cloud infrastructure built as the hands-on deliverable for a FortiGate firewall internship. Dual FortiGate 7.4.12 KVM VMs on a local GNS3 workstation, connected through an OVS layer-2 fabric with 802.1Q VLANs, and linked to an OCI cloud VPS via dual IPsec tunnels under SD-WAN orchestration. The Coursera FortiGate Administrator course provides the structured curriculum; the lab is the practical application.

## Core Value

Deliver a working, production-representative FortiGate topology that demonstrates SD-WAN, IPsec VPN, VDOM segmentation, and NGFW security profiles — proving competency in enterprise firewall administration for the internship.

## Context

- **Motivation:** Internship project centered on FortiGate firewall administration. The Coursera FortiGate Administrator course (Fortinet, Inc.) provides the module-by-module guide; the lab is the practical delivery.
- **Nature:** Network infrastructure project — the only code is a lightweight API deployed on the OCI instance.
- **Host:** GNS3 running on a local workstation (QEMU/KVM).
- **FortiGates:** Two FortiGate 7.4.12 KVM VMs with permanent evaluation licenses — limited to 3 interfaces, 3 policies, 3 routes, 1 vCPU, 2 GB RAM each.
- **OVS:** GNS3 OVS Docker appliance for L2 switching and 802.1Q VLAN trunking.
- **Cloud:** OCI A1.Flex Always Free instance (1 OCPU, 6 GB RAM) — Libreswan IPsec endpoint + Docker services.
- **IP scheme:** 192.168.x private range for all lab subnets.
- **Linux comfort:** Comfortable with CLI.
- **WAN bridge:** Physical NIC passthrough for the second WAN interface is TBD — may use NAT-only fallback.
- **Timeline:** Target completion by end of week (July 19, 2026).
- **Role division:** I (the agent) handle GNS3 topology, FortiGate config, OVS, OCI infrastructure, IPsec/SD-WAN. You (the user) own the Python API code end-to-end — I provide occasional snippets and follow your lead.

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

- Production deployment — lab environment, not production
- digitaraJobs mock API — placeholder, minimal logic

## Constraints

- **Licenses:** FortiGate permanent evaluation license — 3 interfaces, 3 policies, 3 routes, 1 vCPU, 2 GB RAM, max 2 VDOMs, no FortiGuard
- **Bandwidth:** OCI instance bandwidth and egress costs apply
- **Host NIC:** Single physical NIC may limit WAN bridge options (TBD)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| GNS3 OVS appliance | Simple L2 fabric, no custom VM needed | — Pending |
| 192.168.x IP range | Familiar, simple, adequate for lab scale | — Pending |
| Coursera course as structure | Provides module-by-module guide for internship delivery | — Pending |
| Permanent eval license (3/3/3) | Free, no expiry — limits shape the design | — Pending |

---

*Last updated: 2026-07-15 after project initialization*
