---
title: "Coursera: FortiGate Administrator"
course: "FortiGate Administrator"
tags:
  - coursera/fortigate-admin
  - theory/firewall
status: "in-progress"
---

# 🎓 FortiGate Administrator Study Notes

## 📋 Module Tracker
- [x] **Module 1**: Introduction and Initial Configuration
- [x] **Module 2**: Firewall Policies
- [ ] **Module 3**: Routing
- [ ] **Module 4**: Firewall Authentication
- [ ] **Module 5**: SSL-VPN
- [ ] **Module 6**: Web Filtering
- [ ] **Module 7**: Application Control

---

## 📝 Study Notes

### ⚙️ Module 1: Introduction and Initial Configuration
> [!NOTE]
> **Coursera Course Module Completion**: *Introduction and Initial Configuration*
> - Welcome to Fortinet Network Security Specialization (Video: 2 min)
> - Welcome to FortiGate Administrator (Video: 1 min)
> - System and Network Settings (Ungraded Plugin: 28 min)
> - Introduction and Initial Configuration Graded Assignment: **100%** 🎉

*   **Operating Modes**:
    *   **NAT/Route Mode (Default)**: The FortiGate operates as a Layer 3 gateway. Each interface is in a different subnet, and the firewall routes packets between them.
    *   **Transparent Mode**: The FortiGate acts as a Layer 2 bridge. It does not perform routing; interfaces share the same subnet. Administrative access is done via a dedicated management IP.
*   **Configuration Backup/Restore**:
    *   Backups are saved as plaintext XML files containing all CLI commands.
    *   Can be encrypted with a password. Restoring a configuration requires a reboot.
*   **Firmware Upgrades**:
    *   Always follow the official Fortinet upgrade path to avoid configuration database corruption.
*   **Administrator Profiles**:
    *   `super_admin` (default `admin` user): Full access, cannot be deleted or restricted.
    *   Custom profiles can be configured with Read/Write, Read-Only, or Custom permissions for system, network, and security configurations.
*   **Interface Types & Network Settings**:
    *   **Physical**: Hardware ports (e.g., `port1`, `port2`).
    *   **VLAN**: Virtual sub-interfaces running on a physical port (requires 802.1Q tagging).
    *   **Aggregate (LACP)**: Combines multiple physical interfaces into one logical link for redundancy and throughput.
    *   **Loopback**: Logical interface that is always up; useful for routing protocols and management.
    *   **Administrative Access Settings**: Determines allowed protocols (HTTPS, HTTP, SSH, Telnet, PING) on a specific interface.
    *   **Addressing Modes**: Static or DHCP.

### 🛡️ Firewall Policies & Stateful Inspection
*   **Stateful Inspection (Inspection d'État)**:
    *   FortiGate tracks the state of active network connections in a **session table**.
    *   When a new packet arrives, the firewall performs a policy lookup. If permitted, a session is created.
    *   All subsequent packets belonging to that connection (in both directions) are matched against the session table and allowed through without re-evaluating the security policies. This significantly boosts throughput and reduces latency.
*   **Policy Matching Criteria (Critères de Correspondance)**:
    *   To match a policy, a packet must satisfy all of the following criteria:
        *   **Incoming Interface**: Source interface.
        *   **Outgoing Interface**: Destination interface.
        *   **Source Address**: IP, subnet, FQDN, geography, or authenticated User/Group.
        *   **Destination Address**: IP, subnet, FQDN, geography, or Internet Service database (ISDB).
        *   **Schedule**: Plage horaire (Always or custom recurring/one-time schedule).
        *   **Service**: Protocol and port number (e.g., HTTP, custom TCP port).
*   **Matching Order & Policy Design**:
    *   Policies are evaluated in a **strict top-to-bottom order**.
    *   The first matching policy applies, and the search terminates immediately.
    *   *Design Rule*: Specific policies (e.g., restricted access for a single host) must be placed at the top, while general policies (e.g., broad LAN-to-WAN access) must go at the bottom.
*   **Implicit Deny (Règle d'Interdiction Implicite)**:
    *   The final, invisible rule at the very bottom of the policy list.
    *   If a packet does not match any user-defined policy, it matches the implicit deny rule and is **dropped**.
    *   By default, the implicit deny policy does not generate logs. To audit blocked traffic, logging on the implicit deny policy must be manually enabled.
*   **Policy Actions**:
    *   `ACCEPT`: Allows traffic and permits the application of UTM security profiles.
    *   `DENY`: Block the traffic immediately.
    *   `LEARN`: Monitors and logs traffic without blocking, used for testing and profiling traffic baseline before enforcement.

### 🛣️ Module 3: Routing
*   *(Study notes for static routing and dynamic routing will go here)*

### 🔑 Module 4: Firewall Authentication
*   *(Study notes for user directory integration, LDAP, and FSSO will go here)*

### 🔒 Module 5: SSL-VPN
*   *(Study notes for SSL-VPN web and tunnel modes will go here)*

### 🌐 Module 6: Web Filtering
*   *(Study notes for web filtering categories, profile settings, and inspection will go here)*

### 👾 Module 7: Application Control
*   *(Study notes for Application Control and Intrusion Prevention will go here)*

---

## 🏷️ Chapter 3: FortiGate Hardware Sizing & Selection

Selecting the correct FortiGate model requires evaluating target network requirements against datasheet performance metrics:

### 1. FortiGate Model Gamut

| Gamut | Models | Typical User Count | Characteristics |
| :--- | :--- | :--- | :--- |
| **Entry-Level** | 40F to 90G | 5 - 50 users | Compact desktop form-factor, often fanless (silent), utilizes SoC (System-on-a-Chip) processors (e.g., SoC4). |
| **Mid-Range** | 100F to 900G | 100 - 1000 users | Rackable (1U/2U), hot-swappable dual/redundant power supplies, SFP+ (10Gbps) and SFP28 (25Gbps) ports, dedicated NP/CP ASICs. |
| **High-End & Chassis** | 1000F to 7000F | 1000+ / Data Centers | Multi-slot chassis, blade architectures, 100G/400G interfaces, massive firewall throughput (>100 Gbps). |
| **FortiGate-VM** | Virtual Machines | Scaling based on vCPUs | Deployed in virtualized environments (ESXi, KVM, Cloud). Highly flexible, but lacks physical hardware ASIC acceleration (NP/CP offloading relies on host CPU virtualization). |

### 2. Critical Sizing Criteria
*   **Threat Protection / SSL Inspection Throughput**:
    *   *Crucial Guideline*: Never size a firewall based on its raw "Firewall Throughput" (L3/L4 stateful).
    *   *Performance Cost*: Enabling full Security Profiles (UTM) and Deep SSL Inspection reduces the nominal throughput by **70% to 85%**. Sizing must target the "Threat Protection" metric.
*   **Concurrent Sessions**: The maximum number of active connections in the FortiGate state table (governed by RAM).
*   **New Sessions Per Second**: Vitesse at which the firewall can validate and create new sessions in the state table (governed by CPU/ASIC performance).