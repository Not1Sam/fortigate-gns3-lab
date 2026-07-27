# 🚀 Agent Initialization Guide

Welcome, Agent! You have been invoked to assist the user with their FortiGate and Cybersecurity internship. 

To ensure continuity and maximum effectiveness, follow these guidelines strictly:

## 📋 Required Entry Sequence (Do these first!)

1.  **Read the Memory**: Before doing any work, read the files in the `memory/` folder:
    *   `[[facts]]` (Technical details, IP schemes, versions)
    *   `[[decisions]]` (Historical choices and constraints)
    *   `[[progress]]` (Completed tasks and next steps)
    *   `[[daily_objectives]]` (Daily study and lab task lists)
2.  **Log Your Session**: Open `[[log]]` (located in the `memory/` directory) and append a log entry recording your name, the date/time, and your objectives for this session.
3.  **Detect OS & Select Guide**: Determine the user's operating system, then read the correct setup guide.
    *   **On Linux/macOS** (native GNS3): Read `03_GNS3_Labs/Setup-Guide-Linux.md`. Rename `Setup-Guide-Windows.md` to `Setup-Guide-Windows.md.ignore` so future agents don't get confused.
    *   **On Windows** (GNS3 VM): Read `03_GNS3_Labs/Setup-Guide-Windows.md`. Rename `Setup-Guide-Linux.md` to `Setup-Guide-Linux.md.ignore`.
    *   Also read `03_GNS3_Labs/Device-Setup-Guide.md` for per-node config reference.
    *   To detect: run `uname` on Linux/Mac, or check for `SystemRoot` / `COMSPEC` env vars on Windows.
4.  **Verify GNS3 Virtual Environment**: Verify the status of the emulated lab backend.
    *   **Linux**: Check status using `gns3-control status` or `systemctl status gns3`.
    *   **Windows**: Verify GNS3 VM is running in VMware/VirtualBox and is reachable.
    *   If services are dead, start them.
    *   If NAT forwarding is inactive, enable it.
5.  **Enforce Gating Rules**: Check the progress of tasks in `[[daily_objectives]]`.
    *   **Rule 1 (Previous Work Gating)**: If any task from a previous day is unchecked, you **must not** allow the user to start today's tasks. Block new task setups until yesterday's objectives are marked done.
    *   **Rule 2 (No Rushing Ahead)**: You **must not** check off or allow execution of future days' tasks early. Keep progression to one day at a time.

---

## 🛠️ Operational Guidelines

### 1. Maintain the Vault Using Obsidian Skills
*   You must prioritize using Obsidian tools/skills (`obsidian-cli`, `obsidian-markdown`, `obsidian-bases`) when interacting with the vault.
*   Do not overwrite configuration files or user notes directly with raw terminal commands if you can use the CLI or standard note creation interfaces.
*   Keep files linked using Obsidian Wikilinks (`[[Note Name]]`) to preserve relational search and vault mapping.

### 2. Prioritize Visual & Graphical Representations
*   **Default Mode**: If a concept, timeline, topology, or workflow can be represented graphically (using Mermaid flowcharts, state diagrams, tables, Gantt charts, or Canvas layouts), you **must** opt for that visual representation instead of raw text.
*   **Exceptions**: Use raw text only if:
    *   A visual representation is technically unavailable or not optimal for the information.
    *   The user explicitly requests text formatting.

### 3. Update the Memory Post-Action
*   After completing any milestone or significant change, update the following files:
    *   `[[progress]]` (Check off completed tasks, update active goal, define next steps).
    *   `[[facts]]` (If new subnets, devices, credentials, or commands are introduced).
    *   `[[decisions]]` (If you make design choices or troubleshooting changes).
*   Add a detailed summary of what you accomplished in `[[log]]` before finishing your turn.

---

## 📂 Vault Structure Quick Reference
*   `01_Coursera_Study/`: Structured templates for CompTIA Security+, FortiGate Admin, Enterprise Admin, and FortiAnalyzer notes.
*   `02_French_Study_Notes/`: Theoretical reference documents matching the supervisor's curriculum.
*   `03_GNS3_Labs/`: Practical configuration commands and CLI dumps for each lab phase.
*   `memory/`: Session logs, technical facts, and roadmap tracking.
