# Roadmap: FortiGate SD-WAN & OVS Lab

## Overview

Build a GNS3-based FortiGate lab from bare workstation to a fully operational NSE 4 practice environment. The journey follows real-world deployment order: infrastructure first (GNS3, VMs, OVS), then Layer 2 fabric (VLANs, WAN), then cloud endpoint (OCI), then tunnels (IPsec), then orchestration (SD-WAN, VDOMs), and finally security inspection (SSL, NGFW profiles). Every phase delivers a testable increment — a lab that does more than it did before. The permanent trial's 3-interface/3-policy/3-route limit is designed around from the start using VLAN subinterfaces and consolidated address groups.

## Phases

- [ ] **Phase 1: GNS3 Lab Foundation** - Deploy two licensed FortiGate VMs, OVS Docker appliance, and GNS3 internet gateway; establish config backup workflow
- [ ] **Phase 2: VLAN Fabric & Dual WAN** - Build OVS L2 fabric with 802.1Q VLAN trunking, VLAN subinterfaces to stretch trial interface budget, and dual WAN interfaces on primary FortiGate
- [ ] **Phase 3: OCI Cloud & IPsec Tunnels** - Provision OCI endpoint (Libreswan + Docker), establish both IKEv2 IPsec tunnels from GNS3 to cloud
- [ ] **Phase 4: SD-WAN & VDOM Segmentation** - Group IPsec tunnels under SD-WAN with SLA performance probes; split FortiGate into Edge and Internal VDOMs
- [ ] **Phase 5: NGFW Security Profiles** - Apply SSL/TLS Deep Inspection, App Control, Web Filter, and IPS while staying within the 3-policy trial budget

## Phase Details

### Phase 1: GNS3 Lab Foundation
**Goal**: GNS3 topology with two fully licensed FortiGate VMs (permanent trial), OVS Docker appliance, and internet gateway — everything needed to build upon
**Mode**: mvp
**Depends on**: Nothing (first phase)
**Requirements**: GNS3-01, GNS3-02, GNS3-03, OPS-01
**Success Criteria** (what must be TRUE):
  1. Both FortiGate 7.4.12 VMs are imported into GNS3, boot, and show valid permanent trial license status via `get system license` — not expired, not in 14-day eval mode
  2. OVS Docker appliance boots in GNS3 and shows all 16 ports operational on the default `br0` bridge (`ovs-vsctl show`)
  3. GNS3 NAT cloud node provides internet connectivity — a test node connected to it can ping 8.8.8.8
  4. Config backup (`execute backup config tftp`) and restore (`execute restore config tftp`) workflow is documented and verified working for both FortiGates
**Plans**: TBD

### Phase 2: VLAN Fabric & Dual WAN
**Goal**: OVS L2 fabric with 802.1Q VLAN trunking, VLAN subinterfaces on FortiGates (stretching trial's 3-interface limit), and dual WAN paths on primary FortiGate
**Mode**: mvp
**Depends on**: Phase 1
**Requirements**: NET-01, NET-02, NET-03, GNS3-04, VPN-01
**Success Criteria** (what must be TRUE):
  1. VLAN 10, 20, 30 subinterfaces exist on both FortiGates and pass 802.1Q tagged traffic over the OVS trunk port — verified via `show vlan` and packet capture on the trunk link
  2. OVS trunk port is configured with `trunks=10,20,30` and endpoint ports use `tag=10|20|30` for correct access port behavior
  3. Two WAN cloud nodes exist in GNS3 — WAN1 (NAT cloud for internet) and WAN2 (physical NIC bridge or NAT fallback) — both show carrier on the primary FortiGate
  4. Primary FortiGate can ping internet (8.8.8.8) from WAN1 and reach WAN2 next-hop IP
  5. A VPCS host on VLAN 10 receives a DHCP-assigned IP from the FortiGate and can ping its gateway
**Plans**: TBD

### Phase 3: OCI Cloud & IPsec Tunnels
**Goal**: OCI A1.Flex instance provisioned with Libreswan and Docker; both IKEv2 IPsec tunnels established from GNS3 to OCI
**Mode**: mvp
**Depends on**: Phase 2
**Requirements**: OCI-01, OCI-02, OCI-03, OCI-04, VPN-02
**Success Criteria** (what must be TRUE):
  1. OCI A1.Flex instance (1 OCPU, 6 GB RAM, Ubuntu) is provisioned and SSH-accessible via public IP
  2. OCI Security Lists permit UDP 500/4500 and ICMP from the GNS3 public IP, block all other inbound traffic
  3. Libreswan 3.18+ is installed and configured as IPsec responder with route-based VTI interfaces for each tunnel
  4. Docker test container (e.g., nginx or a simple HTTP server) runs on OCI and responds to requests from GNS3
  5. Both IKEv2 IPsec tunnels show Phase 1 and Phase 2 established (`diagnose vpn ike gateway list`, `diagnose vpn tunnel list`) — traffic from VLAN 10 reaches the OCI Docker container through the tunnels
**Plans**: TBD

### Phase 4: SD-WAN & VDOM Segmentation
**Goal**: SD-WAN grouping both tunnels with performance SLA probes steering traffic; VDOMs separating Edge (WAN/SD-WAN) from Internal (LAN/inspection) domains
**Mode**: mvp
**Depends on**: Phase 3
**Requirements**: VPN-03, SEC-03
**Success Criteria** (what must be TRUE):
  1. SD-WAN zone contains both IPsec tunnel members as members — both show healthy status with configured performance SLA probes
  2. SLA probes (ping to OCI Docker IP or another target) show accurate latency/jitter/loss metrics per tunnel — probes are routable through the correct tunnel
  3. SD-WAN rules steer cloud-destined traffic to one tunnel and general internet traffic to the other (or similar application-based steering)
  4. VDOM mode is enabled — Edge-VDOM owns WAN interfaces and SD-WAN zone, Internal-VDOM owns VLAN subinterfaces, inter-VDOM links bridge them
  5. Traffic flows through inter-VDOM link with explicit firewall policies enforcing what crosses between VDOMs — verified by traffic logs
**Plans**: TBD

### Phase 5: NGFW Security Profiles
**Goal**: SSL/TLS Deep Inspection, Application Control, Web Filter, and IPS profiles configured and applied within the 3-policy trial budget
**Mode**: mvp
**Depends on**: Phase 4
**Requirements**: SEC-01, SEC-02
**Success Criteria** (what must be TRUE):
  1. SSL/TLS Deep Inspection with self-signed CA decrypts HTTPS traffic — traffic logs show decrypted URLs and content categories, not "ssl-inspection-invalid"
  2. Application Control profile logs matched application signatures on outbound traffic (e.g., ssl, dns, web-browsing detected)
  3. Web Filter profile enforces category-based blocking — a test request to a blocked category returns a FortiGate block page
  4. IPS engine is enabled on policies and generates alerts for matching traffic patterns
  5. All three NGFW profiles function within the 3-policy trial limit — policies consolidated via address groups, only one inspection policy serving all traffic
**Plans**: TBD

## Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. GNS3 Lab Foundation | 0/0 | Not started | - |
| 2. VLAN Fabric & Dual WAN | 0/0 | Not started | - |
| 3. OCI Cloud & IPsec Tunnels | 0/0 | Not started | - |
| 4. SD-WAN & VDOM Segmentation | 0/0 | Not started | - |
| 5. NGFW Security Profiles | 0/0 | Not started | - |
