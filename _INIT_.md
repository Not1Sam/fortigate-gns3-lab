---
title: Agent Init File
tags:
  - agent/init
  - meta/guide
---

> [!warning] Read this first
> If you are an AI agent assisting with this project, your entry sequence is below.

## 1. Orient

This vault documents a **FortiGate NGFW hybrid-cloud lab** built on GNS3 — the practical deliverable for a cybersecurity internship. The lab has 15 nodes: dual FortiGate HA pair, OVS fabric, Ubuntu/Docker clients, and an OCI cloud threat simulator.

Read these files **before any work**:

| File | Why |
|---|---|
| `[[Full-Topology-Spec]]` | Complete blueprint — architecture, IPs, node specs, wiring, demos |
| `[[Nodes-Reference]]` | Quick reference — node names, ports, images, credentials |
| `[[Topology-Setup-Guide]]` | Step-by-step setup sequence for any agent to follow |
| `[[memory/facts]]` | Technical facts — eval limits, IP scheme, port maps, OCI endpoints |
| `[[memory/decisions]]` | Architectural decisions with rationale |
| `[[memory/progress]]` | Current state — what's done, what's next |

## 2. Key Facts

| Item | Value |
|---|---|
| Hypervisor | GNS3 on Arch Linux (QEMU/KVM + Podman) |
| FortiOS | 7.4.12 KVM, permanent eval (3 ports, 3 policies, 3 routes per FGT) |
| Ubuntu VM user/pass | `ubuntu` / `gns3` (NOPASSWD sudo, pre-provisioned in base image) |
| Root password | `gns3` |
| WAN subnet | `192.168.122.0/24` (NAT1 / virbr0) |
| LAN1 | `192.168.10.0/24` (FGT-Primary port2) |
| LAN2 | `192.168.20.0/24` (FGT-Secondary port2) |
| HA link | `169.254.0.0/30` (port3 on both FGTs) |
| GNS3 project | `/home/bingus/GNS3/projects/d311a72f-2416-4426-9138-96ccd23fe8fd/Internship.gns3` |
| Vault path | `/home/bingus/obsidian-vaults/fortigate-lab` |
| Git repo (public) | `https://github.com/Not1Sam/fortigate-gns3-lab` |

## 3. Workflow

1. **Read** the setup guide (`[[Topology-Setup-Guide]]`)
2. **Check** current state in `[[memory/progress]]` and `.planning/STATE.md`
3. **Ask** the user what step they want to tackle next
4. **Proceed step-by-step** — present each action, let the user confirm
5. **Update** `[[memory/progress]]` and `[[memory/log]]` after completing work

> [!tip] Credentials
> All Docker containers use default credentials. Ubuntu VM: `ubuntu` / `gns3`. FGT web: `admin` / no password (set at first boot).

