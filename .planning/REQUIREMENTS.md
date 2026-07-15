# Requirements: FortiGate SD-WAN & OVS Lab

**Defined:** 2026-07-15
**Core Value:** The lab must let me follow each Coursera module's exercises end-to-end on real FortiGate instances, with enough flexibility to explore my own scenarios beyond the course.

## v1 Requirements

### GNS3 Infrastructure

- [ ] **GNS3-01**: Deploy two FortiGate VM KVM instances (FortiOS 7.4.12) with permanent evaluation licenses
- [ ] **GNS3-02**: Deploy Open vSwitch Docker appliance as L2 fabric
- [ ] **GNS3-03**: Configure GNS3 cloud/NAT nodes for WAN1 (internet access)
- [ ] **GNS3-04**: Configure GNS3 cloud node for WAN2 (physical bridge — TBD if NAT-only fallback)

### Network Fabric

- [ ] **NET-01**: Establish 802.1Q VLAN trunking between FortiGates and OVS
- [ ] **NET-02**: Segment resources into VLAN 10 (LAN), VLAN 20 (AD/LDAP), VLAN 30 (Docker)
- [ ] **NET-03**: Configure VLAN subinterfaces on FortiGates to work within permanent trial interface limit

### OCI Cloud Integration

- [ ] **OCI-01**: Configure OCI Security Lists to allow IPsec traffic (UDP 500, UDP 4500) and custom API ports
- [ ] **OCI-02**: Provision OCI A1.Flex instance (Always Free — 1 OCPU, 6 GB RAM)
- [ ] **OCI-03**: Deploy Libreswan as IPsec endpoint on OCI instance
- [ ] **OCI-04**: Run lightweight Dockerized test service on OCI instance

### IPsec & SD-WAN

- [ ] **VPN-01**: Configure dual WAN interfaces on primary FortiGate (NAT cloud + physical bridge)
- [ ] **VPN-02**: Establish dual IKEv2 route-based IPsec tunnels from GNS3 to OCI
- [ ] **VPN-03**: Configure SD-WAN grouping both tunnels with performance SLA probe

### Security & Segmentation

- [ ] **SEC-01**: Implement SSL/TLS Deep Inspection with self-signed CA
- [ ] **SEC-02**: Deploy Application Control, Web Filtering, IPS on outbound/cloud policies
- [ ] **SEC-03**: Enable VDOMs for Edge-VDOM (WAN/SD-WAN) and Internal-VDOM (LAN/inspection)

### Automation & Recovery

- [ ] **OPS-01**: Document config backup/restore workflow for lab rebuild after eval license changes

## v2 Requirements

### Advanced

- **ADV-01**: FortiGate HA cluster (requires dedicated heartbeat link)
- **ADV-02**: Security Fabric integration between both FortiGates
- **ADV-03**: AD/LDAP server on VLAN 20 for authentication exercises
- **ADV-04**: FSSO agent deployment for identity-based policies

## Out of Scope

| Feature | Reason |
|---------|--------|
| Production deployment | This is a learning lab only |
| digitaraJobs mock API | Placeholder — no real logic needed |
| FortiGuard-dependent features | Permanent trial cannot access FortiGuard updates |
| Cloud-managed SD-WAN (FortiManager) | Out of scope for NSE 4, adds unnecessary complexity |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| GNS3-01 | Phase 1 | Pending |
| GNS3-02 | Phase 1 | Pending |
| GNS3-03 | Phase 1 | Pending |
| GNS3-04 | Phase 2 | Pending |
| NET-01 | Phase 2 | Pending |
| NET-02 | Phase 2 | Pending |
| NET-03 | Phase 2 | Pending |
| OCI-01 | Phase 3 | Pending |
| OCI-02 | Phase 3 | Pending |
| OCI-03 | Phase 3 | Pending |
| OCI-04 | Phase 3 | Pending |
| VPN-01 | Phase 2 | Pending |
| VPN-02 | Phase 3 | Pending |
| VPN-03 | Phase 4 | Pending |
| SEC-01 | Phase 5 | Pending |
| SEC-02 | Phase 5 | Pending |
| SEC-03 | Phase 4 | Pending |
| OPS-01 | Phase 1 | Pending |

**Coverage:**
- v1 requirements: 18 total
- Mapped to phases: 18
- Unmapped: 0 ✓

---

*Requirements defined: 2026-07-15*
*Last updated: 2026-07-15 after project initialization*
