# 🪵 Agent Activity Log

This log tracks the history of AI agents assisting in this project, including timestamps and accomplishments.

## ✍️ Log Entry Template
```markdown
### [YYYY-MM-DD HH:MM] - Agent <Name>
*   **Objectives**: <What you set out to do>
*   **Actions Taken**:
    *   <Action 1>
    *   <Action 2>
*   **Current Status**: <Completed/In-Progress>
```

---

## 📜 Session History

### [2026-07-07 15:54] - Agent Antigravity
*   **Objectives**: Setup internship roadmap, initialize vault templates, and configure memory folders.
*   **Actions Taken**:
    *   Wiped previous lab folder contents using the Obsidian CLI.
    *   Drafted the 5-Phase Integrated Cybersecurity Study Plan.
    *   Created directories and populated core study, French notes, and lab phase template documents.
    *   Initialized the vault `memory/` tracker folders (`progress.md`, `facts.md`, `decisions.md`).
    *   Created the onboarding instructions (`init.md`) and log tracker (`log.md`).
    *   Created `Roadmap.canvas` visual Canvas mapping the 5 GNS3 lab phases.
    *   Adapted daily objectives to a 30-day (1-month) calendar with 8-hour weekdays (9:00 AM - 5:00 PM) and 2-4 hour weekends.
    *   Shifted daily objectives schedule forward by one day (setting Tuesday, July 7 as Day 1 / Completed Planning).
    *   Fixed `Roadmap.canvas` validation errors (replaced custom names with valid 16-char hex IDs) and resized file nodes to 550x450.
    *   Added instruction to prioritize graphical/visual representations (Mermaid, Canvas, tables) over raw text in `init.md`.
    *   Analyzed `gns3-control` and added instructions/facts for GNS3 server and NAT forwarding controls.
*   **Current Status**: Completed. Vault successfully initialized, memory management active, visual canvas fixed and scaled, daily calendar shifted, formatting guidelines updated, and GNS3 setup controls integrated.

### [2026-07-08 09:44] - Agent Antigravity
*   **Objectives**: Read Chapters 1 & 2 of the study guide, complete CompTIA Security+ Module 1, enrich study files for ASIC SPU architecture and packet lifecycle, and prepare the GNS3 VM template for FortiOS v7.4.12.
*   **Actions Taken**:
    *   Verified GNS3 host services (GNS3 server, docker, libvirtd) are active and IP forwarding is enabled (`net.ipv4.ip_forward = 1`).
    *   Extracted key NGFW, SPU acceleration, and packet flow concepts from `fortigate_study.pdf` (Chapters 1 & 2).
    *   Documented CompTIA Security+ Module 1 study notes in `01_Coursera_Study/CompTIA_SecurityPlus.md` covering risk calculations, strategies, and STRIDE framework.
    *   Enriched `02_French_Study_Notes/ASIC_SPU_Architecture.md` with detailed descriptions of NP, CP, SoC processors, and a role comparison table.
    *   Enriched `02_French_Study_Notes/Packet_Lifecycle.md` with detailed descriptions of packet processing steps and a styled Mermaid flowchart.
    *   Identified that `fgt-v7.4.12.qcow2` was already registered in GNS3, and updated the GNS3 `FortiGate-7.4.12` template to use high-performance `virtio` disk interfaces and `virtio-net-pci` network adapters.
    *   Cleaned up database duplication of FortiGate templates in the GNS3 sqlite3 database.
    *   Shifted all future objectives (Days 03 to 30) forward by 1 calendar day, appending original Day 03 tasks to Day 02, allowing the user to start them today.
    *   Verified GNS3 physical wiring topology links and static IP `10.0.1.1` configuration on `port2` from the active GNS3 project.
    *   Guided user to run `gns3-control forward-enable` on their host to fix routing/NAT rules, enabling internet connectivity (`execute ping 8.8.8.8`) on the FortiGate VM console.
    *   Checked off Day 03 lab tasks (host forwarding & default static route) in the objectives calendar.
    *   Populated study notes for FortiGate Administrator Modules 1 & 2, and Chapter 3 sizing specs in `01_Coursera_Study/FortiGate_Admin.md`.
    *   Verified and checked off the Day 04 baseline backup task after the user successfully saved the active configuration to `/03_GNS3_Labs/Phase_01_Baseline/HQ_FGT_Base.conf`.
*   **Current Status**: Completed. Day 02 and Day 03 lab objectives and Day 04 baseline backup successfully verified, GNS3 template optimized, and study notes updated.

### [2026-07-08 13:53] - Agent Antigravity
*   **Objectives**: Record Coursera FortiGate Administrator Module 1 completion and grade in study notes.
*   **Actions Taken**:
    *   Updated `01_Coursera_Study/FortiGate_Admin.md` to document the user's completion of Coursera Module 1 (*Introduction and Initial Configuration*) with a grade of 100%.
    *   Realigned the remaining module trackers and study note placeholders in `01_Coursera_Study/FortiGate_Admin.md` and `memory/daily_objectives.md` to match the actual 7-module Coursera curriculum you provided.
    *   Created `03_GNS3_Labs/Sandbox_Expansion_Guide.md` providing a comprehensive device inventory, physical wiring schematics, CLI configurations (VLAN tagging sub-interfaces, Open vSwitch port mapping, and inter-VLAN security policies), and verification commands for a single-firewall multi-user enterprise topology.
    *   Refactored `03_GNS3_Labs/Sandbox_Expansion_Guide.md` to utilize rich Obsidian-Flavored Markdown (OFM) properties, tags, internal wikilinks (`[[Packet_Lifecycle]]`, `[[FortiGate_Admin]]`, `[[HQ_FGT_Base.conf]]`), and lowercase system callouts.
    *   Restored the multi-firewall HQ + Branch OSPF dynamic routing integration setup in `03_GNS3_Labs/Sandbox_Expansion_Guide.md` and memory files, utilizing separate FortiCloud accounts for licensing, with a direct admin connection at HQ to maintain 3-interface limits on each VM.
    *   Spiced up the topology in `03_GNS3_Labs/Sandbox_Expansion_Guide.md` by inserting a central WAN Edge Router connecting both firewalls, dividing each office into 2 separate physical LAN subnets (4 LANs in total), and configuring OSPF routing across all three routing nodes (HQ, Branch, and Router).
    *   Adapted the WAN Edge Router configurations in `03_GNS3_Labs/Sandbox_Expansion_Guide.md` to use OpenWrt UCI network and firewall commands, and FRRouting (`vtysh`) OSPF setups.
    *   Checked local GNS3 disk images and recorded the list of GNS3 templates to add (VyOS, Alpine Linux, OpenLDAP) to `memory/facts.md`.
*   **Current Status**: Completed.

### [2026-07-09 11:31] - Agent Antigravity
*   **Objectives**: Complete Day 03 and Day 04 shifted objectives, and restructure the calendar around Coursera course completions and physical hardware on-site prep.
*   **Actions Taken**:
    *   Documented comprehensive study notes for Coursera *FortiGate Administrator* Module 2 (Firewall Policies, Stateful Inspection, matching criteria, Implicit Deny, and Actions) in `01_Coursera_Study/FortiGate_Admin.md` and checked it off in the tracker.
    *   Dropped the GNS3 WAN Router OSPF multi-firewall expansion lab objectives per user decision.
    *   Performed Week 1 notes formatting and wikilinks review, establishing links between `FortiGate_Admin.md`, `Packet_Lifecycle.md`, `UTM_Inspection_Modes.md`, and `CompTIA_SecurityPlus.md`.
    *   Restructured the entire 30-day objectives plan in `memory/daily_objectives.md` to prioritize the 14 remaining modules of the 3 FortiGate Coursera courses (*FortiGate Administrator*, *Enterprise Firewall*, and *FortiAnalyzer*) and focus the remaining weeks on physical hardware staging checklists and on-site practice preparation.
    *   Checked off Day 03 and Day 04 objectives in the newly structured `daily_objectives.md`.
    *   Set the next steps for Day 05 (FortiGate Admin Module 3: Routing) in `memory/progress.md`.
*   **Current Status**: Completed. Calendar restructured around Coursera completions, Day 03 & 04 objectives marked done, and study notes updated.

### [2026-07-20 14:07] - Session Restart
*   **Objectives**: Redesign topology for dual-FGT HA, create full tech spec, research eval license limits
*   **Actions Taken**:
    *   Redesigned topology from single-FGT VLAN-on-a-stick to dual-FGT HA cluster with 2 OVS switches
    *   Designed 14-node topology: 2 FGTs, 2 OVS, VPCS, webterm, Alpine DHCP, App Server, PostgreSQL, Ubuntu Desktop, Monitoring Stack, Traffic Gen + Syslog, NAT1, OCI threat sim
    *   Created `Topology.canvas` with full visual topology diagram
    *   Created `Full-Topology-Spec.md` (13 KB) — complete spec: addressing, node specs, port maps, OCI endpoints, resource budget, demo matrix, security baseline
    *   Created `Nodes-Reference.md` — quick ref for all 14 nodes
    *   Fixed HA heartbeat from port7 → port3 (eval license limits to ports 1-3)
    *   Researched FortiGate 7.4.12 eval license limits officially — documented 3-interface, 3-policy, 3-route caps
    *   Researched FortiGuard availability — confirmed no eval license exists, documented workarounds (static URL lists, factory IPS signatures, EICAR hardcoded)
    *   Modified Ubuntu VM disk directly via fuse2fs — set password `gns3`, seeded cloud-init
    *   Updated all memory files (`facts.md`, `decisions.md`, `progress.md`) with current architecture and constraints
*   **Current Status**: Pre-Execution — planning complete, Phase A Wave 1 ready to start
