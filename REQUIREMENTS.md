# Requirements Specification

## 1. Local Network Infrastructure (GNS3)
*   [REQ-01] Deploy two FortiGate VMs with active evaluation licenses.
*   [REQ-02] Deploy an Open vSwitch (OVS) appliance to handle LAN traffic.
*   [REQ-03] Establish 802.1Q VLAN trunking between the FortiGates and the OVS.
*   [REQ-04] Segment internal resources into separate VLANs (e.g., VLAN 10 for LAN, VLAN 20 for Active Directory/LDAP, VLAN 30 for local Docker).

## 2. Cloud Node Integration (OCI)
*   [REQ-05] Configure OCI Security Lists to allow IPsec traffic (UDP 500, UDP 4500) and custom API ports.
*   [REQ-06] Run a lightweight Dockerized API service (digitaraJobs mock) on the OCI instance.

## 3. Security & VPN
*   [REQ-07] Configure two distinct WAN interfaces on the primary FortiGate (NAT cloud and physical bridge).
*   [REQ-08] Establish dual, distinct IPsec tunnels from GNS3 up to the OCI public IP.
*   [REQ-09] Set up FortiGate SD-WAN grouping both tunnels with an active performance SLA probe target.

## 4. NGFW & Monitoring
*   [REQ-10] Implement full SSL/TLS Deep Inspection (utilizing self-signed certificates pushed to GNS3 clients).
*   [REQ-11] Deploy Application Control, Web Filtering, and IPS on outbound and cloud-bound traffic policies.
*   [REQ-12] Enable VDOMs to segment WAN edge (Edge-VDOM) and internal routing (Internal-VDOM).
