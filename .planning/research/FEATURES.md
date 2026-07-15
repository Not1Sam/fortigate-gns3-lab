# Feature Landscape: FortiGate Administrator Lab

**Domain:** FortiGate Security Lab (GNS3) for NSE 4 / Coursera FortiGate Administrator curriculum
**Researched:** 2026-07-15
**Confidence:** HIGH

## Feature Landscape

### Table Stakes (Every FortiGate Lab Must Support)

Features that are non-negotiable. Missing these means the lab cannot follow the course exercises.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Interface configuration & IP addressing | Module 1 (Initial Configuration) requires setting IPs on WAN/LAN ports | LOW | FortiGates need at least 3 functional interfaces: WAN1, WAN2, LAN |
| Static routing & default routes | Module 3 (Routing) — static and default route config, route table analysis | LOW | Must support at least one upstream default route + inter-VLAN static routes |
| Firewall policies (allow/deny) | Module 2 (Firewall Policies) — core of everything FortiGate does | LOW | Need at least LAN→WAN allow, inter-VLAN rules, and explicit deny |
| Source NAT (Overload / IP Pool) | Module 2 (NAT) — SNAT is the default outbound NAT behavior | LOW | Central NAT or per-policy NAT; both approaches should be exercisable |
| Destination NAT (Virtual IP / Port Forwarding) | Module 2 (NAT) — DNAT for inbound services | LOW | VIP configuration for inbound access to lab servers |
| DHCP server on internal interfaces | Modules 1-2 — LAN clients must get IPs automatically | LOW | FortiGate DHCP server for each internal VLAN/interface |
| Administrator accounts & access profiles | Module 1 — admin user creation, permissions, trusted hosts | LOW | Need at least one non-admin profile (read-only) for the exercise |
| System settings (hostname, DNS, NTP) | Module 1 — basic system configuration | LOW | NTP is critical for logs and certificate validation |
| Logging & traffic log visibility | Module 2 (policies) + implicit — must see denied/allowed traffic in logs | LOW | Built-in FortiGate log buffer suffices for labs; no external syslog required |
| Configuration backup & restore | Module 1 — backup via CLI or GUI (needed for eval license rebuilds) | LOW | Both CLI `execute backup config` and GUI download |
| Basic troubleshooting (ping, traceroute, packet capture) | Module 15 (Diagnostics) — `diag debug flow`, `diag sniffer packet`, `execute ping` | LOW | CLI debug tools are examinable; must work across VLANs |
| Security profiles (at least one category) | Modules 7-9 (AV, Web Filter, App Control) — profile creation and policy attachment | MEDIUM | Even one working profile demonstrates the concept; full stack is a differentiator |
| SSL-VPN (Remote Access) | Module 5 (SSL-VPN) — full-tunnel remote access configuration | MEDIUM | Requires a public IP or reachable WAN interface; cert management involved |
| IPsec VPN (Site-to-Site) | Module 11 (IPsec VPN) — tunnel between two FortiGates or to cloud | MEDIUM | Policy-based and route-based modes; IKEv1 and IKEv2 |
| Certificate operations | Module 6 — self-signed CA, certificate generation, SSL inspection certificates | MEDIUM | Needed for SSL inspection and SSL-VPN portal customization |
| Firewall authentication (local users) | Module 4 — local user database, captive portal, policy-based authentication | MEDIUM | Local user + group creation, authentication rules |
| Route table analysis | Module 3 — reading `get router info routing-table`, understanding route selection | LOW | CLI command exposure; no special topology needed beyond multi-path |
| Aggregating multiple WAN links | Module 12 (SD-WAN) — SD-WAN requires 2+ WAN members to function | MEDIUM | Must have at least 2 paths (e.g., NAT cloud + physical bridge, or two logical paths) |

### Differentiators (Advanced Lab Capabilities)

Features that go beyond the Coursera 7-module scope and make the lab a serious NSE 4 practice environment.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| SD-WAN with performance SLA probes | Core NSE 4 Infrastructure topic; dual IPsec tunnels to OCI as SD-WAN members | HIGH | Requires cloud endpoint + SLA probe target; the project's primary differentiator |
| VDOM segmentation (Edge / Internal) | NSE 4 topic; demonstrates multi-tenant firewall design | HIGH | Two VDOMs (Edge-VDOM, Internal-VDOM) with inter-VDOM links; requires careful routing |
| SSL/TLS Deep Inspection (full) | Module 6 + security profiles; decrypting HTTPS to apply AV/IPS/WF | HIGH | Self-signed CA + cert pushed to endpoints; known pain point that proves competency |
| Full NGFW stack (IPS + AV + Web Filter + App Control) | Modules 7-9; combined UTM profile on outbound policies | MEDIUM | Requires valid evaluation license for FortiGuard updates; IPS signatures need connectivity |
| Dual FortiGate HA cluster (A-P or A-A) | Module 14; FGCP cluster setup, failover testing, session synchronization | HIGH | Both FortiGates in a cluster; requires heartbeat, management, and sync interfaces |
| Security Fabric between both FortiGates | Module 13; root + downstream FortiGate fabric setup, single-pane monitoring | MEDIUM | SSO between fabric members; requires reachability and proper authorization |
| LDAP / RADIUS authentication integration | Module 4 (advanced); external auth server for firewall policies | MEDIUM | Would need a lightweight LDAP container (OpenLDAP) in VLAN 20; adds complexity |
| FSSO (Fortinet Single Sign-On) | Module 5; Windows AD polling for user-to-IP mapping | HIGH | Requires a Windows VM or Samba AD domain controller; heavy for a lab |
| BGP dynamic routing | Module 3 (advanced); route redistribution, path selection with SD-WAN | MEDIUM | BGP between FortiGates and upstream OCI VPS for realistic routing |
| OSPF dynamic routing | Module 3 (advanced); useful for internal routing between VDOMs or VLANs | MEDIUM | Can be used between FortiGate VDOMs or between FortiGate and OVS layer |
| Traffic shaping / QoS policies | Module ~12; bandwidth management, per-policy shaper, guaranteed/ max bandwidth | MEDIUM | Queue-based and policy-based shapers; visible in SD-WAN rules |
| Policy-based routing (PBR) | Module 3; forwarding traffic based on criteria beyond destination | MEDIUM | PBR for selective forwarding; complements SD-WAN rules |
| Automation stitches | Post-NSE-4 but valuable; trigger-based incident response (e.g., auto-block IP on IOC) | MEDIUM | CLI + GUI automation; FortiOS 7.4+ native feature; low overhead |
| FortiAnalyzer integration (external) | Not in Coursera but NSE 5 + valuable for logs; centralized log storage | MEDIUM | Requires separate FortiAnalyzer VM or FAZ Cloud; RAM-heavy for lab host |
| Remote access via FortiClient (post SSL-VPN) | Module 5 extension; actual remote client testing from host machine | MEDIUM | Requires host routing to reach SSL-VPN virtual IP pool |
| VLAN segmentation (3+ VLANs) | Project requirement (LAN, AD/LDAP, Docker) — realistic enterprise segmentation | MEDIUM | OVS handles trunking; FortiGate tags/untags per interface |
| OCI cloud as realistic internet endpoint | Future job relevance; hybrid cloud VPN gives real-world credibility | HIGH | The project's cloud bridge — not just simulated internet but real remote endpoint |
| Dual WAN with different transport types | SD-WAN realism: one path is NAT-to-cloud, other is physical bridge | MEDIUM | Shows SD-WAN handles heterogeneous underlay networks |
| Automation scripts for lab rebuild | Evaluation license expires every 14 days; automated rebuild saves hours | MEDIUM | Ansible playbooks or bash scripts to re-deploy base config; enables rapid iteration |

### Anti-Features (Things That Cause Problems in FortiGate Labs)

Features or approaches that seem useful but create issues in a lab context.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Evaluation license manipulation (system date rollback) | Users want to extend past 14 days without re-licensing | FortiOS detects time tampering; eval license expires permanently; VM becomes unusable | Create a documented rebuild process (scripted config restore); budget 15 min for rebuild |
| Running both FortiGates as standalone + HA | Want HA for learning but also independent practice | HA requires dedicated heartbeat interfaces and sync links; switching between modes requires factory reset | Build HA config on separate VDOMs or dedicate one session to HA; snapshot GNS3 project before HA setup |
| Full enterprise LDAP/AD infrastructure for auth | Realism — "real companies use AD" | Windows Server VM consumes 4-8 GB RAM; complex setup time exceeds learning value | Use FortiGate local users for authentication labs; add LDAP only if specifically needed |
| Routing protocols across the OVS fabric | "Real networks use dynamic routing" | OVS is L2 switch; routing protocols between FortiGates over L2 creates confusion (direct adjacencies vs via switch) | Keep OVS as pure L2; run routing protocols only over routed interfaces or IPsec tunnels |
| FortiManager + FortiAnalyzer in same lab | "Central management is enterprise standard" | 2 extra VMs require 6-8 GB RAM each + separate eval licenses; huge resource drain | Deploy after core FortiGate lab is stable; use FortiCloud demo for management exposure |
| Over-segmenting VLANs (6+) | "More realistic enterprise" | Each VLAN needs separate DHCP, policies, routing; management overhead drowns learning signal | Stick to 3-4 VLANs (LAN, Servers, DMZ/Docker, Management) — enough for segmentation practice |
| Building entire config in one pass | Speed — "type it all and verify at end" | Guaranteed to fail; troubleshooting interleaved dependencies is nightmare; no learning reinforcement | Step-by-step, verify-each-layer approach: interfaces → ping → routes → policies → advanced features |
| Simulated internet (GNS3 router + public IP block) | "Lab should work offline" | Adds router VM + BGP config + static route complexity; still not real internet | Use NAT cloud (local) for simple traffic; use real OCI endpoint for advanced scenarios |
| Making lab accessible from outside | Remote access convenience | Security risk: lab FortiGate has eval security holes, weak passwords, no hardening | Keep lab on isolated GNS3 network; use GNS3 console proxy if remote access needed |
| Dual WAN physical bridge on single host NIC | "Need true external connectivity" | If only one physical NIC, splitting WAN1/WAN2 is complex; creates routing conflicts with host | Use NAT-only for WAN1; OCI IPsec for WAN2; avoid physical bridge unless second NIC available |

## Feature Dependencies

```
FortiGate VM Deployment
    └──requires──> KVM/QEMU image + evaluation license activation

Interface Configuration (WAN1, WAN2, LAN)
    └──requires──> FortiGate VM Deployment

VLAN Segmentation
    └──requires──> OVS Appliance Deployment
    └──requires──> Interface Configuration (trunk port)

Static Routing
    └──requires──> Interface Configuration

Firewall Policies
    └──requires──> Interface Configuration
         └──or──> VLAN Segmentation

Source NAT
    └──requires──> Firewall Policies (matching policy)

DHCP Server
    └──requires──> Interface Configuration (on serving interface)

IPsec VPN
    └──requires──> Interface Configuration (WAN reachability)
    └──requires──> Static Routing (or policy routes for tunnel traffic)
    └──requires──> OCI Security List (cloud endpoint)
         └──enhanced_by──> Certificate Operations (IKE cert auth)

SD-WAN
    └──requires──> IPsec VPN (multiple tunnels as SD-WAN members)
    └──requires──> Two WAN paths (NAT + IPsec or dual IPsec)
         └──enhanced_by──> Performance SLA probes (SLA target must be reachable)

SSL-VPN
    └──requires──> Interface Configuration (WAN/LAN reachability)
    └──requires──> Firewall Policies (SSL-VPN → internal)

SSL/TLS Deep Inspection
    └──requires──> Certificate Operations (generate CA + server cert)
    └──requires──> Firewall Policies (SSL inspection profile attached)
         └──enhanced_by──> FortiGuard connectivity (updated CA certs)

Security Profiles (AV, IPS, Web Filter, App Control)
    └──requires──> Firewall Policies (profile attached to policy)
    └──requires──> FortiGuard license (for signature updates)
    └──requires──> SSL Inspection (for HTTPS traffic inspection)

VDOMs
    └──requires──> VDOM license or eval license
    └──requires──> Advance planning (interface-to-VDOM mapping)
         └──conflicts──> HA (HA + VDOM = higher complexity; test separately first)

HA Cluster
    └──requires──> Two FortiGate VMs (same model/firmware)
    └──requires──> Dedicated HA heartbeat interface
    └──requires──> Consistent interface naming across both units
         └──conflicts──> VDOMs (test VDOMs before HA or vice versa — not both at once)

Firewall Authentication
    └──requires──> Firewall Policies (captive portal / auth-enabled policy)
    └──requires──> User/Group objects (local or remote)
         └──enhanced_by──> LDAP/AD server

Automation Stitches
    └──requires──> Firewall Policies (to trigger on event)
    └──requires──> Logging enabled on trigger policies

Configuration Backup Automation
    └──requires──> CLI access (SSH or console)
    └──requires──> Working FortiGate boot
         └──enhances──> Eval license rebuild process

Security Fabric
    └──requires──> Two FortiGates (primary + downstream)
    └──requires──> Reachability between fabric members
```

### Dependency Notes

- **IPsec VPN → OCI Security List:** The cloud VPS must allow UDP 500/4500 inbound. Without this, tunnels won't establish.
- **SD-WAN → dual IPsec tunnels:** SD-WAN requires at least two members. The project's dual tunnels to OCI are the natural SD-WAN members.
- **SSL/Deep Inspection → Certificate Operations:** You cannot do SSL inspection without generating or importing CA certificates. This is a two-step dependency — not optional.
- **VDOMs vs HA:** These conflict if attempted simultaneously. Each fundamentally changes interface ownership and routing domains. Configure VDOMs first, snapshot, then try HA on a separate branch/snapshot.
- **Security Profiles → FortiGuard:** AV, IPS, and Web Filter signatures need FortiGuard updates. Without internet connectivity, they work with built-in signatures but won't catch recent threats. For lab demo purposes, built-in signatures are sufficient.
- **Automation → config backup:** Automation scripts that back up config help mitigate the 14-day eval license expiry problem. Not required but strongly complementary.

## MVP Definition

### Launch With (v1 — Minimum Viable Lab)

The smallest lab that supports all 7 Coursera modules end-to-end.

- [x] **Two FortiGate VMs with eval licenses** — Everything depends on this (REQ-01)
- [x] **OVS appliance for L2 switching** — VLAN trunking foundation (REQ-02)
- [x] **VLAN trunking between FortiGates and OVS** — 802.1Q for inter-VLAN routing (REQ-03)
- [x] **VLAN 10 (LAN) + VLAN 30 (Docker) segments** — Basic segmentation (REQ-04)
- [ ] **Interface configuration (WAN1, WAN2, LAN)** — IP setup for basic connectivity
- [ ] **Static routing (default route + inter-VLAN)** — Module 3 coverage
- [ ] **Source NAT on LAN→WAN policy** — Module 2 (NAT) coverage
- [ ] **DHCP server on VLAN 10 (LAN)** — Module 1 system config
- [ ] **Two firewall policies (LAN→WAN allow + explicit deny)** — Module 2 core
- [ ] **Web filtering profile on LAN→WAN** — Module 6 coverage
- [ ] **Application control profile on LAN→WAN** — Module 7 coverage
- [ ] **Local user + authentication policy** — Module 4 coverage
- [ ] **SSL-VPN (one portal, one user)** — Module 5 coverage
- [ ] **Logging + traffic log review** — Cross-module essential
- [ ] **Config backup (CLI at minimum)** — Eval lifecycle management

The above list supports all Coursera modules. Estimated: ~8-10 hours of configuration.

### Add After Validation (v1.x — NSE 4 Full Coverage)

Features that bring the lab to full NSE 4 FortiGate Administrator curriculum (15 modules).

- [ ] **Dual IPsec tunnels to OCI** — Module 11 + enables SD-WAN (REQ-08)
- [ ] **SD-WAN with performance SLA** — Module 12 (REQ-09)
- [ ] **OCi security list config** — Enables IPsec to cloud (REQ-05)
- [ ] **SSL/TLS Deep Inspection** — Module 6 + enables full UTM on HTTPS (REQ-10)
- [ ] **IPS profile on outbound policies** — Module 9 extension (REQ-11)
- [ ] **Antivirus profile** — Module 7 coverage
- [ ] **Certificate operations (self-signed CA)** — Module 6, prerequisite for deep inspection
- [ ] **VDOMs (Edge + Internal)** — Module ~13/15, advanced segmentation (REQ-12)
- [ ] **HA cluster between both FortiGates** — Module 14
- [ ] **FortiGate diagnostics lab** — Module 15: `diag debug flow`, `diag sniffer`, TAC report
- [ ] **FSSO (Fortinet Single Sign-On)** — Module 5 (if time/VM resources permit)
- [ ] **Security Fabric between both FortiGates** — Module 13

### Future Consideration (v2+ — Beyond NSE 4)

Features that extend into NSE 5/6/7 territory or professional practice.

- [ ] **FortiAnalyzer VM for centralized logging** — NSE 5 scope; high RAM cost
- [ ] **FortiManager for central management** — NSE 5 scope; adds another VM
- [ ] **BGP between FortiGates and OCI VPS** — Advanced routing practice
- [ ] **Automation stitches (auto-block IP / auto quarantine)** — FortiOS 7.4+ native
- [ ] **Traffic shaping with bandwidth guarantees** — Enterprise practice
- [ ] **Terraform/Ansible automation for lab rebuild** — Mitigates eval expiry problem
- [ ] **FortiClient EMS integration** — NSE 5/6 scope; requires Windows EMS VM
- [ ] **Multi-cloud (OCI + AWS/Azure) IPsec** — Advanced SD-WAN practice
- [ ] **ZTN (Zero Trust Network Access) policies** — FortiOS 7.4+ feature

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Interface config + IP addressing | HIGH | LOW | P1 |
| Static routing | HIGH | LOW | P1 |
| Firewall policies + NAT | HIGH | LOW | P1 |
| DHCP server | HIGH | LOW | P1 |
| Source NAT (LAN→WAN) | HIGH | LOW | P1 |
| Web filtering profile | HIGH | MEDIUM | P1 |
| Application control profile | HIGH | MEDIUM | P1 |
| Local user authentication | HIGH | MEDIUM | P1 |
| SSL-VPN portal | HIGH | MEDIUM | P1 |
| Logging & traffic review | HIGH | LOW | P1 |
| Config backup | HIGH | LOW | P1 |
| OVS + VLAN trunking | HIGH | MEDIUM | P1 |
| Dual IPsec to OCI | HIGH | MEDIUM | P2 |
| SD-WAN with SLA probe | HIGH | HIGH | P2 |
| SSL Deep Inspection | HIGH | HIGH | P2 |
| IPS + Antivirus profiles | MEDIUM | MEDIUM | P2 |
| Certificate operations | HIGH | MEDIUM | P2 |
| VDOM segmentation | MEDIUM | HIGH | P2 |
| HA cluster | MEDIUM | HIGH | P2 |
| Troubleshooting labs | HIGH | LOW | P1 |
| FSSO | MEDIUM | HIGH | P3 |
| Security Fabric | MEDIUM | MEDIUM | P2 |
| Automation stitches | MEDIUM | MEDIUM | P3 |
| Traffic shaping | LOW | MEDIUM | P3 |
| BGP/OSPF dynamic routing | MEDIUM | MEDIUM | P3 |
| FortiAnalyzer integration | MEDIUM | HIGH | P3 |

**Priority key:**
- P1: Required for Coursera 7-module coverage — must be in v1 lab
- P2: Required for full NSE 4 coverage — core of v1.x expansion
- P3: Nice-to-have, future or expert-level practice

## Competitor Feature Analysis

| Feature | Official Fortinet Virtual Lab | EVE-NG Community Labs | This Lab (GNS3 + OVS + OCI) |
|---------|-----------------------------|----------------------|------------------------------|
| FortiGate count | 1 (typical) | 2-3 | 2 |
| SD-WAN practice | Limited (simulated WAN) | Single WAN type | Dual WAN (NAT + OCI IPsec) |
| Cloud integration | None | None | Real OCI endpoint |
| Full UTM stack | Included | Varies | Full (AV, IPS, WF, AppCtrl) |
| SSL Deep Inspection | Included | Partial | Full with self-signed CA |
| VDOM segmentation | Not in basic labs | Sometimes | Yes (Edge + Internal) |
| HA cluster | Not in basic labs | Sometimes | Planned (A-P) |
| Automation scripts | None | Rare | Planned (Ansible/scripts) |
| Eval license rebuild | Inconvenient | Manual reinstall | Scripted backup + restore |
| VLAN segmentation | 2 VLANs | 2-3 VLANs | 3-4 VLANs via OVS |
| Security Fabric | Not in labs | Rare | Planned |
| Remote access (SSL-VPN) | Basic | Basic | Full FortiClient testing |
| OCI integration | N/A | N/A | Dual IPsec tunnels |
| Cost | Official training required | Free (labor) | Free (labor + OCI egress) |
| Scalability | Fixed by training | Flexible | Limited by host RAM |

**Key differentiation:** This lab's real OCI cloud endpoint with dual IPsec tunnels under SD-WAN is not available in any other free lab topology. Most FortiGate labs simulate internet/LAN with another router — this one connects to an actual cloud provider.

## Sources

- Fortinet Training Institute: FortiGate Administrator (NSE 4) course syllabus — 15-module agenda from training.fortinet.com
- Coursera FortiGate Administrator — 7-module structure (from coursera.org/learn/fortigate-administrator)
- Class Central syllabus details for the Coursera offering
- Global Knowledge / MUK Training course outlines for FCP-FGT-AD 7.4
- Grandmetric "FortiGate Configuration: How to Avoid Common Mistakes" — anti-feature guidance
- GNS3 FortiGate appliance registry (appliance requirements, version notes)
- Fortinet Community: eval license discussion, GNS3 setup guidance
- UNDERC0DETESTING: Virtual Fortinet Lab for FCSS blog post
- GetLabsDone: Build a FortiGate lab using GNS3 guide
- FortiOS 7.4 Best Practices (Fortinet documentation)

---

*Feature research for: FortiGate Administrator GNS3 Lab*
*Researched: 2026-07-15*
