<!-- GSD:project-start source:PROJECT.md -->

## Project

**FortiGate SD-WAN & OVS Lab**

A hands-on GNS3 lab topology for the Coursera FortiGate Administrator course (Fortinet NSE 4). Dual FortiGate 7.4.12 VMs on a local workstation, connected through an OVS layer-2 fabric with 802.1Q VLANs, and linked to an OCI cloud VPS via dual IPsec tunnels under SD-WAN orchestration. Designed to practice every module in the course and serve as a reusable reference lab.

**Core Value:** The lab must let me follow each Coursera module's exercises end-to-end on real FortiGate instances, with enough flexibility to explore my own scenarios beyond the course.

### Constraints

- **Licenses:** FortiGate evaluation licenses expire after 14 days — lab must be reproducible
- **Bandwidth:** OCI instance bandwidth and egress costs apply
- **Host NIC:** Single physical NIC may limit WAN bridge options (TBD)

<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->

## Technology Stack

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| FortiGate VM (KVM) | FortiOS 7.4.12 | Primary firewall instances (x2) | Latest stable 7.4 train; 7.4.12 is the current GA release for the 7.4 branch as of mid-2026. All Coursera NSE 4 material targets 7.4.x. KVM qcow2 image is the native format for GNS3/GNS3 VM deployment. Uses Fortinet's **permanent evaluation license** (no 14-day expiry, though limited to 3 interfaces / 3 routes / 3 policies). |
| Open vSwitch (Docker) | `gns3/openvswitch:latest` | L2 fabric / VLAN trunking | Native GNS3 marketplace appliance. 16-port Docker container, zero configuration boot (all ports on `br0` bridge by default). Supports 802.1Q VLANs, LACP, RSPAN, OpenFlow. Simpler and lighter than running a full VyOS or Linux bridge VM. |
| Libreswan | 3.18+ (v4.x kernel recommended) | OCI-side IPsec endpoint | Open-source, OCI-documented CPE reference. OCI's own documentation uses Libreswan as the canonical on-prem CPE for Site-to-Site VPN. Route-based VPN via VTI interfaces for straightforward routing integration with FortiGate SD-WAN. |
| OCI Compute (Always Free) | VM.Standard.A1.Flex (1 OCPU, 6 GB RAM) or VM.Standard.E2.1.Micro | Docker host + IPsec CPE | A1.Flex is Always Free (up to 2 OCPU / 12 GB across all A1 instances as of June 2026). Sufficient for Libreswan + Docker API container. E2.1.Micro (1/8 OCPU, 1 GB) is also free but too weak for Libreswan and Docker simultaneously. |
| GNS3 | 2.2.52+ (latest stable) | Topology emulation host | The orchestrator. Running on GNS3 VM (recommended for KVM nesting) or directly on Linux host with KVM support. Current stable as of mid-2026 is 2.2.x. |
| Docker | 24.x+ (on OCI) | Lightweight API endpoint | Runs the `digitaraJobs` mock API or any simple HTTP service behind the FortiGate SD-WAN for inspection/SSL inspection practice. |

### Supporting Components

| Component | Version | Purpose | When to Use |
|-----------|---------|---------|-------------|
| GNS3 OVS appliance | `gns3/openvswitch:latest` | VLAN trunking between FortiGates and LAN segments | Required for 802.1Q VLAN segmentation (VLAN 10/20/30) |
| OCI Dynamic Routing Gateway (DRG) | N/A (OCI service) | OCI-side VPN termination point | Required for IPsec tunnel termination on OCI side. Paired with Libreswan CPE. |
| VPCS (in GNS3) | 0.6+ | Lightweight endpoint hosts for ping/traceroute/simple tests | Far lighter than full Linux VM clients inside the topology. Perfect for VLAN 10/20/30 endpoints. |
| Ubuntu Server Cloud Image (GNS3) | 24.04 LTS | AD/LDAP server VM on VLAN 20 | Required for AD/LDAP integration exercises in the Coursera course. |
| empty30G.qcow2 | N/A | Empty disk image for FortiGate VM boot volume | Required by every FortiGate VM in GNS3 — FortiOS requires a writable disk for config storage and logs. |

### Development & Management Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| FortiGate CLI (SSH/console) | All configuration | The evaluation license limits you to 3 policies and 3 routes — GUI is usable but CLI is faster for rebuilds |
| OCI Console / CLI | Manage IPsec CPE, security lists, compute | Security lists must open UDP 500, 4500, and ESP (protocol 50) for IPsec |
| ovs-vsctl (on OVS appliance console) | Configure OVS VLANs, check bridge status | Access via GNS3 console on the OVS node |
| ipsec (Libreswan CLI) | Check tunnel status on OCI side | `ipsec auto --status`, `ipsec whack --trafficstatus` |
| Wireshark (host-side, via GNS3 capture) | Packet inspection, VPN debug | Use GNS3's built-in packet capture on any link |

## Installation

### FortiGate VM (GNS3)

# 1. Download from Fortinet Support Portal

#    Go to: Support > Downloads > VM Images > KVM

#    Download: FGT_VM64_KVM-v7.4.12.M-buildXXXX-FORTINET.out.kvm.qcow2

#    Download also: empty30G.qcow2 (or create with qemu-img)

# 2. Import into GNS3

#    GNS3 > Edit > Preferences > QEMU > Qemu VMs > New

#    Name: FortiGate-7.4.12

#    RAM: 2048 MB (max for eval license)

#    vCPUs: 1 (max for eval license)

#    Boot priority: hda (qcow2 image), hdb (empty30G.qcow2 for config storage)

#    Network adapters: up to 8 (type virtio)

# 3. First boot — apply permanent evaluation license

#    Console in, run:

#    This reboots. After reboot license is valid permanently.

### Open vSwitch (GNS3)

# Via GNS3 GUI:

# 1. File > Import appliance > openvswitch.gns3a (from marketplace)

# 2. Install on: GNS3 VM (or local server)

# 3. Add to topology, connect via Ethernet interfaces

# Default: all ports on br0 bridge

# For VLAN configuration via console (once running):

### OCI Always-Free Instance (IPsec + Docker)

# Shape: VM.Standard.A1.Flex (1 OCPU, 6 GB RAM — within Always Free limits)

# OS: Oracle Linux 8 / Ubuntu 24.04 LTS

# Boot volume: 50 GB (within 200 GB Always Free block storage)

# Install Libreswan:

# or

# Configure per OCI IPSec connection (see ARCHITECTURE.md for full config)

# /etc/ipsec.d/oci-ipsec.conf — two tunnels with VTI interfaces

# Install Docker:

# Run mock API container:

### GNS3 Project Setup

# Directory structure:

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| L2 fabric | GNS3 OVS appliance (Docker) | VyOS VM | VyOS is overkill for pure L2 switching; OVS is lighter, purpose-built, and handles VLAN trunking natively |
| L2 fabric | GNS3 OVS appliance (Docker) | Linux bridge (via GNS3) | The Docker OVS appliance is pre-built and maintained by GNS3 team; custom Linux bridge requires manual VM creation and networking |
| OCI endpoint | Libreswan on Always Free instance | OCI FortiGate-VM marketplace image | FortiGate on OCI is BYOL ($) or PAYG — the Always Free tier supports Libreswan natively. The lab goal is to practice on-prem FortiGate → cloud IPsec, not run FortiGate on the cloud side |
| OCI endpoint | Libreswan on Always Free instance | OCI VPN Connect (DRG only) | DRG-only VPN (without CPE) doesn't give you a Linux endpoint for Docker. You need both a Docker host and a VPN endpoint — combining them into one Always Free A1 instance is the cheapest path |
| Client endpoints | VPCS | Linux/Windows full VMs | VPCS is ultra-light, boots instantly, uses minimal resources. Full VMs for AD/LDAP on VLAN 20 only where necessary |
| FortiGate license | Permanent evaluation (3-interface limit) | 14-day eval (older FortiOS) | The 14-day eval had no interface limit but expired. The permanent eval never expires — the tradeoff is worth it for a lab you want reusable. See "What NOT to Use" below for workaround to the 3-interface limit |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| FortiOS 7.6.x | Too new for course material; NSE 4 courses and community guides target 7.4.x. 7.6 permanent eval license also exists but adds complexity without benefit for this lab. | FortiOS 7.4.12 |
| GNS3 built-in Ethernet switch (cloud/ethernet) | No VLAN support, L2-only, no way to configure 802.1Q trunks or tagged interfaces | GNS3 OVS appliance (Docker) |
| OCI VPN Connect only (no CPE) | Pure DRG termination means no Linux host for Docker/API endpoint on cloud side. You lose the ability to run the mock API and test SSL inspection. | Libreswan + Docker on a single A1.Flex instance |
| FortiGate-VM on VMware (.vmdk) in GNS3 | Requires nested virtualization on top of GNS3 VM, poor performance, unnecessary complexity | FortiGate-VM KVM (.qcow2) directly on GNS3 VM |
| The `FOS_VM64_...` image type | Has full features and no interface/policy limits, but **stops routing after 1 hour** — unusable for a lab that needs to stay up for exploring. Only suitable for automated 10-minute test runs. | Standard `FGT_VM64_KVM*` image with permanent eval — 3 interface limit is manageable with careful planning |
| IKEv1 for IPsec tunnels | IKEv1 is legacy; IKEv2 is more robust, supports multiple SAs, better DPD, and is the default in FortiOS 7.x | IKEv2 |
| Policy-based IPsec VPN | More complex to extend; changes require editing phase 2 selectors for every new subnet. Route-based is the modern pattern on both FortiGate and OCI/libreswan. | Route-based IPsec (VTI interfaces) |

## Stack Patterns by Variant

- Apply license to the first FortiGate V.
- For the second, you'll need a second FortiCloud account (free, just another registration). The permanent eval is one per account.
- Alternatively: decommission the first VM's license via Fortinet support portal after finishing work on it, then apply to the second. But this is tedious — just create a second account.
- Use GNS3 server on a separate Linux host or a dedicated lab machine.
- GNS3 VM with nested KVM for FortiGate requires at minimum 8 GB RAM just for the host, plus 2 GB per FortiGate (total 12 GB minimum). Without GNS3 VM, you can run GNS3 directly on Linux with KVM — no nesting overhead but setup is more involved.
- Use VLAN subinterfaces. The 3-interface limit applies to **physical** interfaces. You can trunk multiple VLANs over a single physical port and create VLAN subinterfaces in FortiOS. This is the officially documented workaround.
- Example: port1 = WAN1 (physical), port2 = WAN2 (physical), port3 = trunk to OVS with subinterfaces port3.10 (LAN), port3.20 (AD/LDAP), port3.30 (Docker).
- Use policy routing or BGP if possible. The route limit applies to static routes only. Dynamic routes via BGP/OSPF do not count.
- For the SD-WAN lab specifically, BGP between FortiGates and ADVPN is standard anyway.

## Version Compatibility

| Component A | Compatible With | Notes |
|-------------|-----------------|-------|
| FortiOS 7.4.12 | GNS3 2.2.x (QEMU/KVM) | Must use QEMU VM type in GNS3, not the older Dynamips. KVM qcow2 image, virtio network adapters. |
| FortiOS 7.4.12 (eval) | 1 vCPU, 2048 MB RAM | Exceeding either will cause license status to show invalid. The system will still boot but the license check will fail. |
| FortiOS 7.4.12 | OCI DRG (IKEv2) | Use `set ike-version 2` + `set proposal aes256-sha256` + `set dhgrp 14` on FortiGate. OCI DRG supports these params. |
| OVS Docker (gns3/openvswitch) | GNS3 2.2.x | Works on both GNS3 VM and local GNS3 server. Must have Docker runtime available. |
| Libreswan 3.18+ | OCI DRG | OCI DRG expects IKEv1 or IKEv2 with specific proposals. Reference OCI's Libreswan config templates. Libreswan VTI interfaces require kernel 3.x+ (4.x preferred). |
| VM.Standard.A1.Flex (ARM) | Docker, Libreswan, Ubuntu 24.04 | All three support ARM64 natively. Oracle Linux 8 also runs on ARM. No x86 emulation needed. |
| FortiOS 7.4.12 | NSE 4 Coursera material | The course material targets 7.4.x specifically. 7.4.12 is the latest 7.4 train and is fully compatible. |

## Sources

- **Fortinet docs** — Permanent trial mode for FortiGate-VM (FortiOS 7.4.10 Administration Guide). Confirms eval license: 1 vCPU, 2 GB RAM, 3 interfaces/routes/policies, no FortiGuard. [HIGH confidence — official vendor documentation]
- **Fortinet KVM Admin Guide 7.4** — Deployment requirements, disk sizing (30 GB minimum), SR-IOV options. [HIGH confidence]
- **Fortinet Community** — Setup guide for FortiGate in GNS3, confirms image download process, GNS3 import steps, known issue with curl 28 error for license activation (needs internet connectivity from VM). [MEDIUM confidence — community KB, validated against multiple sources]
- **Fortinet Community (eval license forum)** — Confirms 3-interface limit is a major pain point for SD-WAN labs; workaround is VLAN subinterfaces over a single physical trunk port. [MEDIUM confidence — forum users reporting real experience]
- **GNS3 OVS marketplace page** — Appliance provides 16 adapters, `gns3/openvswitch:latest` Docker image, all ports bridged to br0 by default. [HIGH confidence — official GNS3 registry]
- **GNS3 project structure docs** — Project format (.gns3 JSON), snapshot system, portable projects (.gns3project). [HIGH confidence — official documentation]
- **OCI Compute Shapes docs** — VM.Standard.A1.Flex (ARM Ampere) available on Always Free. VM.Standard.E2.1.Micro (1/8 OCPU, 1 GB) also free. A1 limits reduced to 2 OCPU / 12 GB as of June 2026. [HIGH confidence — official OCI docs]
- **OCI Libreswan reference** — Official OCI guide for connecting via Libreswan CPE with route-based VTI tunnels, IKEv1/IKEv2 support, ECMP configuration. [HIGH confidence — official OCI documentation]
- **OCI VPN Connect pricing** — ~$0.05–$0.10 per hour per tunnel, or free via Always Free resources (50 IPSec connections included). [HIGH confidence — official OCI docs]
- **Libreswan version compatibility** — VTI support requires Libreswan 3.18+ and Linux kernel 3.x/4.x. OCI validates against Libreswan 3.25+. [HIGH confidence — OCI documentation]
- **FOS_VM64 limitation** — 1-hour timeout confirmed by community sources and Tech Cyber blog (2025). Useful only for short automated tests. [MEDIUM confidence — blog post, not official Fortinet documentation]

<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->

## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->

## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->

## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->

## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:

- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->

## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
