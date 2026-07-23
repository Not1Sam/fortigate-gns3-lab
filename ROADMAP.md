# Implementation Roadmap

- [ ] Phase 1: GNS3 Infrastructure Setup & OVS Layer 2 Configuration
  - **Goal:** Get GNS3 nodes mapped, OVS trunking configured, and local VLAN sub-interfaces up on the FortiGates.
  - **Requirements:** REQ-01, REQ-02, REQ-03, REQ-04

- [ ] Phase 2: OCI VPS Security, Bridging, and Docker Services
  - **Goal:** Set up OCI firewalls, deploy the mock API container, and establish end-to-end host-to-cloud ping routing.
  - **Requirements:** REQ-05, REQ-06, REQ-07

- [ ] Phase 3: Dual IPsec VPN, Routing, and SD-WAN Integration
  - **Goal:** Configure both IPsec tunnels, run OSPF dynamic routing, and group links under an active SD-WAN SLA zone.
  - **Requirements:** REQ-08, REQ-09

- [ ] Phase 4: VDOM Segmentation & Next-Gen UTM Rules (SSL Inspection, IPS)
  - **Goal:** Split the FortiGate into VDOMs, generate CAs for SSL Deep Decryption, and test security profiles against live traffic.
  - **Requirements:** REQ-10, REQ-11, REQ-12

- [ ] Phase 5: Verification, Logs Routing, and Admin CLI Diagnostics
  - **Goal:** Stream UTM logs to an external syslog vm/server and run live debugging diagnostics during automated simulations.
  - **Requirements:** All REQs
