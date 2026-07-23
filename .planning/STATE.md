---
gsd_state_version: '1.0'
status: planning
progress:
  total_phases: 5
  completed_phases: 0
  total_plans: 3
  completed_plans: 0
  percent: 0
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-07-15)

**Core value:** Deliver a working FortiGate topology demonstrating SD-WAN, IPsec, VDOMs, and NGFW — proving enterprise firewall administration competency for the internship.
**Current focus:** Phase 1 — GNS3 Lab Foundation

## Current Position

Phase: 1 of 5 (GNS3 Lab Foundation)
Plans: 01 / 02 / 03 (2 waves)
Status: Ready to execute
Last activity: 2026-07-17 — Phase 1 planned (3 plans, 2 waves)

Progress: [██░░░░░░░░] 20% (planned)

## Performance Metrics

**Velocity:**
- Total plans completed: 0
- Average duration: —
- Total execution time: — hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 1. GNS3 Lab Foundation | 3 | Planned | — |
| 2. VLAN Fabric & Dual WAN | 0 | — | — |
| 3. OCI Cloud & IPsec Tunnels | 0 | — | — |
| 4. SD-WAN & VDOM Segmentation | 0 | — | — |
| 5. NGFW Security Profiles | 0 | — | — |

**Recent Trend:**
- Last 5 plans: —
- Trend: —

## Accumulated Context

### Decisions

Recent decisions affecting current work:

- **Trial license strategy:** Use permanent trial (no 14-day eval) — limited to 3 interfaces, 3 policies, 3 routes, 1 vCPU, 2 GB RAM per FortiGate. VLAN subinterfaces workaround for interface limit. One trial per FortiCare account — need two accounts.
- **Coarse granularity:** 5 phases delivering working increments. Each phase is an MVP — testable at completion.
- **Phase order:** Infrastructure → Fabric → Cloud/IPsec → SD-WAN/VDOM → NGFW — follows dependency chain. VDOMs before SD-WAN for cleaner interface ownership.
- **Policy budget:** Must design all policies within 3-policy limit using address groups. Policy 1: LAN→WAN (internet), Policy 2: LAN→IPsec (cloud), Policy 3: Inter-VDOM (inspection).
- **Phase 1 plan split:** 3 plans across 2 waves — Wave 1: topology build + license activation (01, 02), Wave 2: WAN internet + config backup (03).

### Pending Todos

None yet.

### Blockers/Concerns

- **WAN2 physical bridge (GNS3-04):** Feasibility of physical NIC passthrough for second WAN on primary FortiGate is TBD. If host has only one NIC, fallback to NAT cloud on separate VMnet adapter. Decision needed during Phase 1/2.
- **FortiGuard on permanent trial:** IPS signature updates and FortiGuard lookups require paid subscription. Built-in signatures work for profile configuration but current threat intelligence won't be available. Lab acknowledges this limitation.
- **SSL inspection with VPCS:** VPCS nodes cannot install custom CA certificates. SSL deep inspection testing will need a Linux VM or certificate-only inspection mode as workaround.

## Deferred Items

| Category | Item | Status | Deferred At |
|----------|------|--------|-------------|
| *(none)* | | | |

## Session Continuity

Last session: 2026-07-17 — Phase 1 planned (3 plans, 2 waves)
Stopped at: Plans presented for approval
Resume file: None
