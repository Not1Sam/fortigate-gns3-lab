# Project Research Summary

**Project:** FortiGate SD-WAN & OVS Lab
**Domain:** FortiGate Security / GNS3 Network Emulation / NSE 4 Certification Lab
**Researched:** 2026-07-15
**Confidence:** HIGH

## Executive Summary

This project is a GNS3-based virtual lab topology for the Coursera FortiGate Administrator (NSE 4) course. It pairs two FortiGate 7.4.12 VMs with an Open vSwitch L2 fabric (802.1Q VLAN trunking) on a local workstation, linked to an OCI Always-Free cloud VPS via dual IPsec tunnels under SD-WAN orchestration. The lab is designed to support all 15 NSE 4 course modules end-to-end while serving as a reusable reference environment.

The recommended approach is a phased build that mirrors real-world FortiGate deployment patterns: start with GNS3 infrastructure and license activation, establish base connectivity through OVS and VLANs, then progressively add IPsec, VDOM segmentation, SD-WAN, and NGFW policy layers. The architecture uses a three-VDOM design (root management, Edge-VDOM for WAN/SD-WAN, Internal-VDOM for LAN/inspection) with dual IPsec tunnels over distinct WAN transports to demonstrate enterprise SD-WAN path selection with performance SLA probes.

**Key risk:** The PROJECT.md references "14-day evaluation licenses," but Fortinet removed the automatic 14-day evaluation in FortiOS 7.2.1+. The **permanent trial license** (no expiry) is the correct option, but it's limited to 3 interfaces, 3 policies, 3 routes, 1 vCPU, and 2 GB RAM — and can only be registered once per FortiCare account. This constraint fundamentally changes the policy and routing design budget. **Mitigation:** Use VLAN subinterfaces to bypass the 3-interface limit, consolidate policies via address groups, and accept that FortiGuard-dependent features (IPS signature updates) won't work on the permanent trial.

## Key Findings

### Recommended Stack

The stack is unusually well-documented because GNS3 FortiGate labs are a mature category with excellent community and vendor documentation. The core technologies are:

**Core technologies:**
- **FortiGate VM KVM (FortiOS 7.4.12):** Primary firewall instances (x2) — the latest stable 7.4 train aligned with NSE 4 course material. Uses the permanent evaluation license (no expiry). KVM qcow2 is the native format for GNS3 VM deployment.
- **Open vSwitch Docker (`gns3/openvswitch:latest`):** L2 fabric / VLAN trunking — a 16-port Docker appliance from the GNS3 marketplace. Lighter than a VyOS VM, supports 802.1Q VLANs, and boots with zero config (all ports on `br0` bridge).
- **Libreswan 3.18+ (on OCI):** OCI-side IPsec endpoint — OCI's own canonical CPE reference for Site-to-Site VPN. Route-based via VTI interfaces for clean routing integration with FortiGate SD-WAN.
- **OCI Compute (VM.Standard.A1.Flex):** Docker host + IPsec CPE — Always Free tier (1 OCPU, 6 GB RAM). Runs both Libreswan and Docker concurrently. ARM64 (Ampere) is natively supported by all needed software.
- **GNS3 2.2.52+:** Topology emulation host — running on GNS3 VM with KVM nesting for FortiGate VMs.
- **VPCS 0.6+:** Lightweight endpoint hosts (ping/traceroute) in each VLAN — far lighter than full Linux VMs.
- **Ubuntu 24.04 LTS:** AD/LDAP server VM on VLAN 20 for authentication exercises.

**Critical version note:** FortiOS 7.4.12 is the specific version required — earlier 7.4.x builds (7.4.0–7.4.2) have known inter-VLAN routing bugs. 7.6.x is too new for course material. The permanent trial VM must use exactly 1 vCPU and 2048 MB RAM — exceeding either causes license activation failure.

### Expected Features

**Must have (table stakes — required for Coursera 7-module coverage):**
- Interface configuration & IP addressing (P1) — Module 1 foundation
- Static routing & default routes (P1) — Module 3
- Firewall policies (allow/deny) with source NAT (P1) — Module 2
- DHCP server on internal interfaces (P1) — Module 1
- Security profiles (Web Filter + App Control) (P1) — Modules 6–7
- SSL-VPN remote access (P1) — Module 5
- IPsec Site-to-Site VPN (P1) — Module 11
- Certificate operations (P1) — Module 6
- Local user + firewall authentication (P1) — Module 4
- Logging & traffic log review (P1) — cross-module
- Config backup/restore (P1) — eval lifecycle

**Should have (differentiators — full NSE 4 coverage):**
- SD-WAN with performance SLA probes (P2) — Module 12, the project's signature feature
- Dual IPsec tunnels to OCI (P2) — enables SD-WAN path diversity
- SSL/TLS Deep Inspection with self-signed CA (P2) — Module 6
- Full NGFW stack: IPS + AV (P2) — Modules 7–9
- VDOM segmentation (Edge + Internal) (P2) — Module ~13/15
- HA cluster between both FortiGates (P2) — Module 14
- Security Fabric (P2) — Module 13

**Defer (v2+):**
- FortiAnalyzer integration (NSE 5 scope, high RAM cost)
- FortiManager (NSE 5 scope, extra VM)
- BGP/OSPF dynamic routing (adds complexity without course alignment)
- Automation stitches (post-NSE-4, nice-to-have)
- Multi-cloud IPsec (advanced practice)
- Terraform/Ansible rebuild automation (mitigates eval expiry but is a project in itself)

### Architecture Approach

The architecture uses OVS as a central L2 fabric connecting both FortiGates and downstream VLAN segments, eliminating the need for a physical managed switch. On the WAN side, dual transport paths (GNS3 NAT cloud + optional physical NIC bridge) provide path diversity for the two IPsec tunnels to OCI. The primary FortiGate is split into three VDOMs: root (management), Edge-VDOM (WAN, IPsec, SD-WAN), and Internal-VDOM (VLAN subinterfaces, NGFW inspection, inter-VLAN routing). Inter-VDOM links form the only traffic crossing points, making security policy enforcement auditable.

**Major components:**
1. **FGT-Primary** — SD-WAN edge, VDOM host, NGFW, NAT, dual IPsec to OCI (FortiOS 7.4.12 KVM)
2. **FGT-Secondary** — secondary unit for cross-device testing / HA (separate eval license)
3. **OVS-Core** — L2 fabric: 802.1Q VLAN trunking, per-port tagging (GNS3 OVS Docker appliance, 16 ports)
4. **WAN1-Cloud / WAN2-Cloud** — dual WAN transport via GNS3 NAT cloud + optional physical NIC bridge
5. **OCI VPS** — IPsec responder, Docker API host, SLA probe target, syslog receiver (A1.Flex, Libreswan + Docker)
6. **VPCS / Client VMs** — test endpoints per VLAN (VPCS for lightweight, Ubuntu VM for AD/LDAP)

### Critical Pitfalls

11 critical pitfalls were identified from multiple authoritative sources. The top 5 that must be addressed during planning:

1. **License Assumption (CP-01/CP-02):** The 14-day evaluation license no longer exists in FortiOS 7.2.1+. Use the **permanent trial** (no expiry) via `execute vm-license` with a FortiCloud account. One trial per account — create a second account if your primary is already used. Allocate exactly 1 vCPU and 2 GB RAM or activation fails.

2. **OVS VLAN Trunking (CP-03):** Misconfigured trunk/access ports on OVS are the #1 cause of "VLAN doesn't work." The OVS-to-FortiGate port must be a trunk (`trunks=10,20,30`), endpoint ports must be access (`tag=10`). Default OVS behavior is flat (no VLAN isolation) unless explicitly configured.

3. **IPsec NAT-T Tunnel Flap (CP-04):** When both ends detect NAT, NAT-T negotiation can conflict with GNS3's NAT node causing tunnel oscillation. Solution: ensure OCI has a public IP on its primary VNIC (no extra NAT layer), enable `set nat-traversal enable` on FortiGate phase1, and verify security lists include UDP 500/4500 and ESP.

4. **SD-WAN SLA Probes Failing (CP-05):** SLA probes originate from the FortiGate's source IP, and if the probe target's route goes through a non-tunnel interface, all probes show 100% loss and tunnels are marked dead. Fix: set SD-WAN member source to LAN/loopback IP, verify routing table shows probe target via tunnel, and increase `probe-timeout` from 500ms to 2000ms.

5. **Trial Resource Budget Exhaustion (CP-11):** The permanent trial allows only 3 firewall policies and 3 static routes. This is extremely tight for a topology needing LAN→WAN, LAN→IPsec, IPsec→LAN, plus management and DMZ policies. Mitigation: consolidate with address groups, use floating static routes, and accept you cannot test FortiGuard features (IPS/AV signature updates) on the trial.

## Implications for Roadmap

The research reveals a clear dependency chain that maps naturally to phases. The architecture's build order (from ARCHITECTURE.md) combined with the feature prioritization matrix and pitfall-to-phase mapping yields the following suggested structure:

### Phase 1: GNS3 Project & FortiGate VM Deployment
**Rationale:** Everything depends on the FortiGate VMs and evaluation licenses being operational. This phase resolves the two highest-risk pitfalls (CP-01, CP-02) and establishes the project structure.
**Delivers:** Working GNS3 project with two licensed FortiGate VMs, OVS appliance imported, base cabling.
**Addresses:** REQ-01 (FortiGate VMs), REQ-02 (OVS), REQ-03 (VLAN trunking skeleton).
**Avoids:** CP-01 (14-day eval assumption), CP-02 (license registration failure), CP-11 (resource budget — plan policies upfront).
**Research flag:** MEDIUM — FortiCare account management is well-documented but the "one trial per account" limitation requires advance preparation. Create two FortiCloud accounts.

### Phase 2: Base Connectivity & Internet Access
**Rationale:** OVS deployment, VLAN configuration, and internet connectivity must work before any cloud-dependent features. This validates the L2 fabric and resolves the GNS3 NAT node pitfall.
**Delivers:** OVS br0 with configured VLAN trunk/access ports, FortiGate internet access via WAN1 NAT cloud, DHCP on VLAN 10.
**Addresses:** REQ-03 (VLAN trunking), REQ-04 (VLAN 10/20/30 segments).
**Avoids:** CP-03 (OVS trunk misconfiguration), CP-09 (GNS3 NAT unidirectional), CP-10 (inter-VLAN bug — use 7.4.12).
**Research flag:** LOW — well-documented OVS and GNS3 NAT patterns. Can proceed without research-phase.

### Phase 3: OCI VPS Provisioning
**Rationale:** The OCI endpoint is the other end of the IPsec tunnels. It must be provisioned, secured, and reachable before tunnel configuration begins. This is a dependency of Phase 4.
**Delivers:** OCI A1.Flex instance with Libreswan and Docker, security lists configured, instance reachable from GNS3.
**Addresses:** REQ-05 (OCI Security Lists), REQ-06 (Docker API service).
**Avoids:** CP-08 (OCI security list/routing misconfiguration).
**Research flag:** LOW — OCI Always-Free setup is well-documented. Use standard OCI quickstart patterns.

### Phase 4: FortiGate Initial Configuration (Basic)
**Rationale:** Interface IPs, static routing, and basic firewall policies are prerequisites for IPsec tunnels and VDOMs. This phase establishes the baseline config that later phases build upon. Must respect the 3-policy and 3-route budget from the start.
**Delivers:** Configured port IPs, default route via WAN1, static routes for VLAN subnets, LAN→WAN source NAT policy, DHCP on VLAN 10.
**Addresses:** Table stakes from FEATURES.md (interface config, static routing, source NAT, DHCP, admin accounts, logging, config backup).
**Avoids:** CP-11 (design policy/route budget upfront — use address groups to consolidate).
**Research flag:** MEDIUM — the 3-policy budget constraint is unusual and forces creative consolidation. Plan exact policy and route assignments before writing config.

### Phase 5: IPsec VPN to OCI
**Rationale:** Dual IPsec tunnels are the foundation for SD-WAN. They require both the OCI endpoint (Phase 3) and FortiGate basic config (Phase 4) to be operational. This is the highest-effort integration point.
**Delivers:** Two established IKEv2 IPsec tunnels (Tunnel 1 via WAN1, Tunnel 2 via WAN2), tunnel interface IPs, routing over tunnels.
**Addresses:** REQ-08 (dual IPsec tunnels), enables REQ-09 (SD-WAN).
**Avoids:** CP-04 (NAT-T tunnel flap), CP-08 (OCI route tables and security lists for return traffic).
**Research flag:** HIGH — OCI IPsec with Libreswan has many configuration knobs. Phase-specific research needed for OCI CPE object, DRG attachment, and security list exact rules. Recommend `--research-phase`.

### Phase 6: VDOM Segmentation
**Rationale:** VDOM mode fundamentally changes the FortiGate's configuration model (interface ownership, routing tables, admin scopes). It should be enabled after basic IPsec is stable but before adding SD-WAN complexity, since VDOMs change how interfaces are assigned and managed.
**Delivers:** Three VDOMs (root, Edge-VDOM, Internal-VDOM) with inter-VDOM links, interfaces assigned per VDOM, inter-VDOM firewall policies.
**Addresses:** REQ-12 (VDOMs), Feature: VDOM segmentation (P2).
**Avoids:** CP-07 (management lockout after VDOM enable — keep console open, pre-configure allowaccess on mgmt interface).
**Research flag:** MEDIUM — VDOM config patterns are well-documented but the interaction with the 3-policy budget is tricky. Need to plan which policies live in which VDOM.

### Phase 7: SD-WAN Configuration
**Rationale:** SD-WAN depends on dual IPsec tunnels (Phase 5) and VDOM structure (Phase 6) being stable. Performance SLA probes must have routable probe targets.
**Delivers:** SD-WAN zone with both IPsec tunnel members, performance SLA health-check, SD-WAN rules for cloud traffic and general internet.
**Addresses:** REQ-09 (SD-WAN with SLA), the project's primary differentiator.
**Avoids:** CP-05 (SLA probe routing failure), CP-04 (verify tunnel stability before adding SD-WAN).
**Research flag:** MEDIUM — SD-WAN config patterns on FortiOS 7.4 are well-documented by Fortinet. The tricky part is correct SLA probe routing.

### Phase 8: NGFW Security Profiles
**Rationale:** SSL inspection, IPS, AV, Web Filter, and App Control policies applied to SD-WAN rules. This is the top of the dependency chain — it sits on top of working SD-WAN with verified traffic flows.
**Delivers:** SSL/TLS Deep Inspection with custom CA, application control signatures, web filtering profiles, IPS policies on outbound/cloud traffic.
**Addresses:** REQ-10 (SSL Deep Inspection), REQ-11 (IPS/AppCtrl profiles), full UTM stack.
**Avoids:** CP-06 (SSL inspection certificate trust — generate proper CA, distribute to clients, or use certificate-only inspection for lab purposes).
**Research flag:** HIGH — SSL inspection on FortiOS 7.4 with TLS 1.3/post-quantum key exchange has known incompatibilities. Phase-specific research needed for flow-based vs proxy-based mode and CA distribution strategy. Recommend `--research-phase`.

### Phase 9: Advanced & Exploration
**Rationale:** Secondary FortiGate integration, HA cluster, Security Fabric, and free-form exploration. These features add value but are not on the critical path for course coverage.
**Delivers:** FGT-Secondary configured for cross-device testing, optional HA cluster, Security Fabric, remaining NSE 4 modules.
**Addresses:** Module 13 (Security Fabric), Module 14 (HA), Module 15 (diagnostics).
**Avoids:** Anti-patterns from FEATURES.md (HA + VDOM conflict, over-segmentation).
**Research flag:** MEDIUM — HA cluster on GNS3 with eval licenses has specific constraints. Security Fabric patterns are well-documented.

### Phase Ordering Rationale

- **Infrastructure-first:** GNS3 project, FortiGate images, and OVS must exist before anything else (Phase 1 → Phase 2).
- **Cloud before tunnel:** OCI VPS must be provisioned and reachable before IPsec tunnels can be configured (Phase 3 before Phase 5).
- **Tunnel before SD-WAN:** SD-WAN needs at least two members (the dual IPsec tunnels) to function (Phase 5 before Phase 7).
- **VDOM before SD-WAN:** VDOM mode changes interface ownership and routing domains; SD-WAN configuration is cleaner when the VDOM structure is already in place (Phase 6 before Phase 7).
- **Basic config before advanced:** Static routes, firewall policies, and NAT must work before VDOMs or SD-WAN are added (Phase 4 before Phase 6/7).
- **Inspection last:** NGFW security profiles are the final layer — they inspect traffic that is already flowing through the SD-WAN policy (Phase 8 last).

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 5 (IPsec VPN):** OCI-specific Libreswan configuration, DRG attachment, route-based VTI setup with two tunnels. Multiple integration points between two platforms.
- **Phase 8 (NGFW Profiles):** SSL deep inspection TLS 1.3 compatibility, flow-based vs proxy-based mode, CA distribution to GNS3 clients (VPCS has no CA store).

Phases with standard patterns (skip research-phase):
- **Phase 1/2 (GNS3 Project & Base Connectivity):** Well-documented patterns. FortiGate VM import in GNS3 is mature knowledge in the community.
- **Phase 3 (OCI VPS):** Standard OCI Always-Free provisioning. Use OCI quickstart guides for A1.Flex + Ubuntu.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All components verified against official documentation (Fortinet KVM Guide, OCI Compute docs, GNS3 OVS registry). Version compatibilities confirmed. |
| Features | HIGH | Course syllabus directly from Fortinet Training Institute and Coursera. Feature priorities derived from module structure. |
| Architecture | HIGH | Architecture follows standard Fortinet reference designs (VDOM segmentation, dual IPsec, SD-WAN). Patterns confirmed by Fortinet official SD-WAN guides. |
| Pitfalls | HIGH | 11 critical pitfalls sourced from Fortinet KB articles, community reports, and verified against multiple independent sources. Recovery strategies documented. |

**Overall confidence:** HIGH

### Gaps to Address

- **Physical NIC bridge feasibility:** The PROJECT.md notes WAN2 physical NIC bridge is "TBD." If only one host NIC is available, dual NAT clouds on separate VMnet adapters is the fallback. This reduces WAN diversity realism but keeps the lab functional. Decision needed during Phase 1 or Phase 2.
- **FortiGuard on permanent trial:** The permanent trial explicitly disables FortiGuard. IPS, Web Filtering, and App Control can be configured with built-in signatures, but signature updates require a paid subscription. The lab scope must clearly acknowledge that "UTM with current threat intelligence" is not achievable without a license.
- **3-policy budget design:** The exact consolidation strategy for fitting the lab's policy requirements into 3 firewall policies needs to be designed before Phase 4. Possible approach: Policy 1 = LAN→WAN (internet + FortiGuard), Policy 2 = LAN→IPsec (cloud traffic), Policy 3 = Inter-VDOM (default allow with UTM). This leaves no room for DMZ or management policies — those may need creative workarounds.
- **VPCS SSL inspection limitation:** VPCS nodes cannot trust custom CA certificates. SSL deep inspection testing requires either a Linux VM with the CA installed or certificate-only inspection mode. The lab guide must document this limitation and provide the workaround path.

## Sources

### Primary (HIGH confidence)
- Fortinet Official: FortiOS 7.4 Administration Guide — SD-WAN, VDOM, IPsec VPN, NGFW profiles
- Fortinet Official: KVM Admin Guide 7.4 — FortiGate VM deployment requirements
- Fortinet Official: Permanent Trial License documentation — limitations, activation process
- Fortinet Official: FortiOS 7.4.12 Release Notes — confirmed inter-VLAN bug fix
- OCI Official: Site-to-Site VPN Best Practices — CPE configuration, tunnel redundancy
- OCI Official: Compute Shapes — Always Free tier limits (A1.Flex, E2.1.Micro)
- OCI Official: Libreswan reference config — route-based VTI tunnels with IKEv2
- GNS3 Official: OVS appliance marketplace — 16-port Docker container, VLAN support
- Fortinet Training Institute: NSE 4 FortiGate Administrator syllabus — 15-module agenda
- Coursera: FortiGate Administrator course structure — 7-module scope

### Secondary (MEDIUM confidence)
- Fortinet Community: "FortiOS 7.4.2 Bug Causes IPsec VPN Tunnel Phase 2 Instability" — tunnel behavior patterns
- Fortinet Community: "Inter-VLAN routing issues" — 7.2.x forwarding bug confirmed
- Fortinet Community: "Issues with GNS3 and FortiOS" — deployment patterns
- Fortinet Community: Eval license workarounds — VLAN subinterface approach to bypass 3-interface limit
- GetLabsDone: "Build a FortiGate lab using GNS3" — practical GNS3 + FortiGate integration
- Grandmetric: "FortiGate Configuration: Common Mistakes" — anti-feature guidance
- Andrew Travis SD-WAN Lab Setup — reference topology designs with WANem and Hub/Spoke
- KiwiTut GNS3 FortiGate Lab Guides — LAG/VLAN topology patterns

### Tertiary (LOW confidence — needs validation)
- Tech Cyber blog (2025): FOS_VM64 1-hour timeout limitation — not official Fortinet documentation, but confirmed by multiple community sources
- University of Naples: OVS in GNS3 lab patterns — academic reference, validated against practical community guides

---

*Research completed: 2026-07-15*
*Ready for roadmap: yes*
