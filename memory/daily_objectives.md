# 📅 Daily Objectives & Gating Rules (Theory & On-Site Practice Plan)

This file manages your daily timeline for a 1-month internship starting on **Tuesday, July 7, 2026** and ending on **Wednesday, August 5, 2026**.

---

## 🛑 Progress Gating Rules
> [!caution] **Rule 1: Unfinished Work Gating**
> You are **NOT** allowed to start Today's tasks if any task from any previous day is left unchecked. Yesterday's unfinished tasks must be completed first.

> [!warning] **Rule 2: No Rushing Ahead**
> You are **NOT** allowed to start or check off tasks for future days early. Focus on one day at a time.

---

## ⏰ Standard Daily Time Allocations
*   **Weekdays (Monday - Friday: 9:00 AM - 5:00 PM)**:
    *   **9:00 AM - 5:00 PM (8 hrs)**: Coursera video lectures, note taking, quiz completions, and technical research.
*   **Weekends (Saturday - Sunday: 2 to 4 Hours Max)**:
    *   Study modules, review notes, and preparatory reading.

---

## 📋 30-Day Calendar & Daily Checklists

### 🟢 Week 1: Foundations & Baseline (Phase 01)

#### Day 01 (Tuesday, July 7, 2026) - 🏋️ Weekday (8h)
*   **9:00 AM - 5:00 PM (Setup & Planning)**:
    - [x] Wiped the old vault files while preserving configurations.
    - [x] Initialized vault directories and templates for Coursera courses, French study guide notes, and GNS3 labs.
    - [x] Drafted the 5-Phase Integrated Cybersecurity & FortiGate Firewall Internship Plan.
    - [x] Initialized memory files (`facts.md`, `decisions.md`, `progress.md`, `init.md`, `log.md`).
    - [x] Created `Roadmap.canvas` visual mapping file.

#### Day 02 (Wednesday, July 8, 2026) - 🏋️ Weekday (8h)
*   **9:00 AM - 12:30 PM (Theory)**:
    - [x] Read `fortigate_study.pdf` Chapters 1 & 2 (General NGFW concepts, packet lifecycle, and SPU hardware acceleration).
    - [x] Coursera: *CompTIA Security+* - Module 1 (Risk Management & Threats).
*   **1:30 PM - 5:00 PM (Lab Setup)**:
    - [x] Create `ASIC_SPU_Architecture.md` and `Packet_Lifecycle.md` files in Obsidian.
    - [x] Download the FortiOS v7.4.12 QCOW2 image and prepare the GNS3 VM.
*   **Shifted Lab Objectives** *(superseded by July 20 dual-FGT HA redesign — see Topology-spec.md and memory/decisions.md)*:
    - [x] ~~GNS3 Action: Build the physical wiring topology (NAT Cloud to port1, Switch to port2).~~
    - [x] ~~CLI Action: Configure `port2` static IP (`10.0.1.1/24`) and administrative protocols.~~
    - [x] ~~License Action: Bind the VM to FortiCloud to apply the Free Evaluation License.~~

#### Day 03 (Thursday, July 9, 2026 - TODAY) - 🏋️ Weekday (8h)
*   **9:00 AM - 5:00 PM (Theory)**:
    - [x] Coursera: *FortiGate Administrator* - Module 2 (Firewall Policies).
    - [x] Study rule evaluation logic and "Implicit Deny" rules.

#### Day 04 (Friday, July 10, 2026) - 🏋️ Weekday (3h)
*   **9:00 AM - 12:00 PM (Review)**:
    - [x] Review all Week 1 notes for formatting and wikilinks.
    - [x] GNS3 Action: Backup baseline configuration to `/03_GNS3_Labs/Phase_01_Baseline/HQ_FGT_Base.conf`.

#### Day 05 (Saturday, July 11, 2026) - 🧘 Weekend (3h)
*   **Theory (FortiGate Administrator)**:
    - [ ] Coursera: *FortiGate Administrator* - Module 3 (Routing).
    - [ ] Study route lookup parameters, distance administrative (AD), and dynamic route priority.

#### Day 06 (Sunday, July 12, 2026) - 🧘 Weekend (2h)
*   **Theory (FortiGate Administrator)**:
    - [ ] Coursera: *FortiGate Administrator* - Module 4 (Firewall Authentication).
    - [ ] Study local database, LDAP directory connector, and single sign-on (FSSO) methods.

---

### 🔵 Week 2: FortiGate Admin Completion & Enterprise Start

#### Day 07 (Monday, July 13, 2026) - 🏋️ Weekday (8h)
*   **Theory (FortiGate Administrator)**:
    - [ ] Coursera: *FortiGate Administrator* - Module 5 (SSL-VPN).
    - [ ] Study split-tunneling, web portal configuration, and endpoint security checks.

#### Day 08 (Tuesday, July 14, 2026) - 🏋️ Weekday (8h)
*   **Theory (FortiGate Administrator)**:
    - [ ] Coursera: *FortiGate Administrator* - Module 6 (Web Filtering).
    - [ ] Study URL filtering, FortiGuard categories, and proxy vs flow inspection modes.

#### Day 09 (Wednesday, July 15, 2026) - 🏋️ Weekday (8h)
*   **Theory (FortiGate Administrator)**:
    - [ ] Coursera: *FortiGate Administrator* - Module 7 (Application Control) — **Course Complete!**
    - [ ] Study protocol signature matching and port evasion bypasses.

#### Day 10 (Thursday, July 16, 2026) - 🏋️ Weekday (8h)
*   **Theory (Enterprise Firewall Administrator)**:
    - [ ] Coursera: *Enterprise Firewall Administrator* - Module 1 (Enterprise Network Topologies & WAN).
    - [ ] Study multi-site architectures, hub-and-spoke topologies, and redundant uplinks.

#### Day 11 (Friday, July 17, 2026) - 🏋️ Weekday (8h)
*   **Theory (Enterprise Firewall Administrator)**:
    - [ ] Coursera: *Enterprise Firewall Administrator* - Module 2 (Dynamic Routing Protocols: OSPF & BGP).
    - [ ] Study OSPF area link-state databases, BGP path attributes, and peerings.

#### Day 12 (Saturday, July 18, 2026) - 🧘 Weekend (3h)
*   **Theory (Enterprise Firewall Administrator)**:
    - [ ] Coursera: *Enterprise Firewall Administrator* - Module 3 (High Availability Clustering & FGCP).
    - [ ] Study Active-Passive vs Active-Active HA, heartbeat synchronizations, and session failover.

#### Day 13 (Sunday, July 19, 2026) - 🧘 Weekend (2h)
*   **Theory (Enterprise Firewall Administrator)**:
    - [ ] Coursera: *Enterprise Firewall Administrator* - Module 4 (Enterprise IPSec VPNs & SD-WAN).
    - [ ] Study IKEv2 phase parameter negotiations, SD-WAN rules, and SLA performance probes.

---

### 🟡 Week 3: Enterprise Completion & FortiAnalyzer

#### Day 14 (Monday, July 20, 2026) - 🏋️ Weekday (8h)
*   **Theory (Enterprise Firewall Administrator)**:
    - [ ] Coursera: *Enterprise Firewall Administrator* - Module 5 (Diagnostics, Sniffers & Troubleshooting) — **Course Complete!**
    - [ ] Study CLI packet tracing commands, debug flows, and system resource monitors.

#### Day 15 (Tuesday, July 21, 2026) - 🏋️ Weekday (8h)
*   **Theory (FortiAnalyzer Administrator)**:
    - [ ] Coursera: *FortiAnalyzer Administrator* - Module 1 (Log Collection & Registration Process).
    - [ ] Study FortiAnalyzer ADOMs, secure log transmission protocols, and device authorization.

#### Day 16 (Wednesday, July 22, 2026) - 🏋️ Weekday (8h)
*   **Theory (FortiAnalyzer Administrator)**:
    - [ ] Coursera: *FortiAnalyzer Administrator* - Module 2 (Log Storage, Disk Quotas & Archive Policies).
    - [ ] Study log retention databases, SQL index structures, and disk quota alerts.

#### Day 17 (Thursday, July 23, 2026) - 🏋️ Weekday (8h)
*   **Theory (FortiAnalyzer Administrator)**:
    - [ ] Coursera: *FortiAnalyzer Administrator* - Module 3 (FortiView & Incident Analytics).
    - [ ] Study log filtering dashboards, drill-down analytics, and incident handling.

#### Day 18 (Friday, July 24, 2026) - 🏋️ Weekday (8h)
*   **Theory (FortiAnalyzer Administrator)**:
    - [ ] Coursera: *FortiAnalyzer Administrator* - Module 4 (Designing Custom Compliance Reports) — **Course Complete!**
    - [ ] Study SQL query reporting, chart builders, and regulatory compliance templates (PCI-DSS, ISO27001).

#### Day 19 (Saturday, July 25, 2026) - 🧘 Weekend (3h)
*   **Study Notes Integration**:
    - [ ] Review all Coursera study files to ensure links and tags are fully established.

#### Day 20 (Sunday, July 26, 2026) - 🧘 Weekend (2h)
*   **Staging Staging Prep**:
    - [ ] Study standard hardware deployment checklist and physical console cabling.

---

### 🟣 Week 4: Physical Staging & On-Site Practice Prep

#### Day 21 (Monday, July 27, 2026) - 🏋️ Weekday (8h)
*   **On-Site Prep (Hardware Deployments)**:
    - [ ] Read `fortigate_study.pdf` Ch 7 (Security Hardening best practices).
    - [ ] Document physical FortiGate device staging guidelines (trusted host configuration, disabling insecure protocols: HTTP/Telnet).

#### Day 22 (Tuesday, July 28, 2026) - 🏋️ Weekday (8h)
*   **On-Site Prep (VLANs & Trunks)**:
    - [ ] Study configuring VLAN sub-interfaces on physical FortiGate ports and trunking into network switches (Core switches, access ports).

#### Day 23 (Wednesday, July 29, 2026) - 🏋️ Weekday (8h)
*   **On-Site Prep (High Availability Staging)**:
    - [ ] Study practical physical HA clustering procedures (cabling redundant heartbeat links, power redundancy staging).

#### Day 24 (Thursday, July 30, 2026) - 🏋️ Weekday (8h)
*   **On-Site Prep (SD-WAN & WAN Staging)**:
    - [ ] Study physical staging of SD-WAN connections (connecting ISP modems, testing WAN gateway recovery).

#### Day 25 (Friday, July 31, 2026) - 🏋️ Weekday (8h)
*   **On-Site Prep (Log Forwarding & FAZ)**:
    - [ ] Study physical FortiAnalyzer connection procedures, network logging requirements, and VPN syslog paths.

#### Day 26 (Saturday, August 1, 2026) - 🧘 Weekend (3h)
*   **Report Outlining**:
    - [ ] Draft the supervisor report framework inside Obsidian.

#### Day 27 (Sunday, August 2, 2026) - 🧘 Weekend (2h)
*   **Report Outlining**:
    - [ ] Refine report chapters for NGFW, HA, Sizing, and Staging guidelines.

---

### 📄 Final Compilation & Report Drafting

#### Day 28 (Monday, August 3, 2026) - 🏋️ Weekday (8h)
*   **Report Compilation**:
    - [ ] Write Report Chapter 1 (Introduction) and Chapter 2 (NGFW Architecture, SPU, and Sizing).

#### Day 29 (Tuesday, August 4, 2026) - 🏋️ Weekday (8h)
*   **Report Compilation**:
    - [ ] Write Report Chapter 3 (Staging Guidelines & Troubleshooting) and Chapter 4 (HA Clustering & Log forwarding).

#### Day 30 (Wednesday, August 5, 2026 - FINAL DAY) - 🏁 Completed (0h)
*   **Internship Objectives Completed!**
    - [ ] No tasks scheduled. All theoretical courses completed and staging guidelines documented!
