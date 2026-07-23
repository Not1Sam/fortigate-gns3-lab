# Project Vision: Secure Multi-Cloud SD-WAN & OVS Fabric

## Goal
Build an enterprise-grade hybrid cloud infrastructure using GNS3 as the local HQ environment and an external Oracle Cloud Infrastructure (OCI) VPS as the distributed cloud datacenter. This setup is designed to put the FortiGate Administrator (NSE 4) curriculum into practice, utilizing dual FortiGate evaluation VMs, Open vSwitch (OVS) layer 2 segmentation, and OCI-hosted microservices.

## Core Architecture
*   **Edge Firewalls (GNS3):** 2x FortiGate VMs configured with active evaluation licenses running FortiOS.
*   **Virtual Switches (GNS3):** Open vSwitch (OVS) acting as a high-performance L2 switch backplane handling 802.1Q VLAN trunking.
*   **Cloud Node (OCI):** 1x Ubuntu/AlmaLinux VPS running a containerized mockup API (for digitaraJobs) and remote logging endpoints.
*   **Routing & Security:** Dual-path IPsec tunnels grouped under an SD-WAN zone running OSPF, with VDOM isolation.
