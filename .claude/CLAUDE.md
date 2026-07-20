<!-- GSD:project-start source:PROJECT.md -->

## Project

**FortiGate & Cybersecurity Internship Lab**

An Obsidian-managed learning vault and GNS3 laboratory playground for a 1-month cybersecurity internship focused on the FortiGate Next-Generation Firewall (NGFW) solution. The project integrates Fortinet theoretical study, Coursera certification courses, practical lab implementations, and security robustness testing.

**Core Value:** Design, configure, and validate a secure enterprise network topology on GNS3 using FortiGate firewalls to master all key features required by the internship supervisor (Policies, NAT, Routing, VPNs, HA, UTM, Authentication, Logging) and test its security posture.

### Constraints

- **Licensing**: Free evaluation licenses permit only 2 active FortiGate VMs simultaneously.
- **Eval limits per FGT**: 3 interfaces (ports 1-3), 3 firewall policies, 3 static routes, 1 vCPU, 2 GB RAM.
- **No FortiGuard/FortiCare**: UTM uses factory signatures (EICAR, SQLi/XSS) + static URL/domain block lists.
- **No FortiAnalyzer/FortiManager**: Syslog via Alpine + socat instead.
- **Resource limit**: Evaluation VMs are restricted to 1 vCPU and 2048 MB RAM.
- **Hardware Sizing**: FortiOS features require offloading or sizing; in GNS3, UTM inspection modes (Flow vs Proxy) significantly impact memory and CPU.

<!-- GSD:project-end -->

<!-- GSD:stack-start source:STACK.md -->

## Technology Stack

Technology stack not yet documented. Will populate after codebase mapping or first phase.
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
