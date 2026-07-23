---
phase: "01"
plan: "01"
type: "infrastructure"
wave: 1
depends_on: []
files_modified:
  - GNS3 topology file (topology.gns3project or .gns3)
autonomous: false
requirements: [GNS3-03]
---

<objective>
Build GNS3 topology with both FortiGate VMs, OVS Docker appliance, WAN1 NAT cloud, and VPCS test nodes connected by Ethernet links.
</objective>

<read_first>
- FORTIGATE-REFERENCE.md — interface mapping, first boot procedure
</read_first>

<tasks>
<task>
<type>infrastructure</type>
<action>Create GNS3 topology project and place all nodes

1. Create new GNS3 project named "FortiGate-SDWAN-Lab"
2. Add from templates:
   - FGT-Primary (FortiGate 7.4.12 KVM, 2048 MB RAM, 1 vCPU)
   - FGT-Secondary (FortiGate 7.4.12 KVM, 2048 MB RAM, 1 vCPU)
   - OVS-Core (gns3/openvswitch:latest)
   - VPCS-LAN-10 (VPCS template)
   - WAN1-Cloud (GNS3 NAT cloud)
3. Connect nodes by Ethernet cables:
   - FGT-Primary port1 → WAN1-Cloud
   - FGT-Primary port2 → OVS-Core port1
   - FGT-Secondary port1 → OVS-Core port2
   - FGT-Secondary port2 → (unused in Phase 1)
   - VPCS-LAN-10 → OVS-Core port3
4. Set all link types to Ethernet (not serial)
</action>
<acceptance_criteria>
- GNS3 project file exists with all 5 nodes placed
- All required cable connections shown in GNS3 topology
- Each FortiGate VM has exactly 2048 MB RAM and 1 vCPU
</acceptance_criteria>
</task>

<task>
<type>infrastructure</type>
<action>Start all nodes and verify console access

1. Start nodes in order: WAN1-Cloud → OVS-Core → FGT-Primary → FGT-Secondary → VPCS-LAN-10
2. Open console for each FortiGate, observe boot sequence
3. Accept EULA when prompted on first boot
4. Set admin password on each FortiGate when prompted
5. Verify login works: admin + chosen password
6. VPCS: open console, verify prompt shows PC1>
</action>
<acceptance_criteria>
- FortiGate console shows login prompt after boot completes
- `get system status` shows FortiGate-7.4.12, uptime increasing
- VPCS console shows `PC1>` prompt
- OVS console shows Open vSwitch banner
</acceptance_criteria>
</task>
</tasks>

<verification>
1. All 5 nodes boot and show operational status (green) in GNS3
2. Console access confirmed for each node type
3. FGT-Primary `get system interface physical` shows port1 and port2 with link UP
</verification>

<success_criteria>
- [ ] GNS3 project created with all nodes connected
- [ ] All 5 nodes start without errors
- [ ] Console accessible on all nodes
</success_criteria>

<must_haves>
  <truths>
    - FGT-Primary and FGT-Secondary are both placed in the GNS3 topology
    - OVS-Core is placed and connected to both FortiGates
    - WAN1-Cloud is placed and connected to FGT-Primary port1
    - VPCS-LAN-10 is placed and connected to OVS-Core
    - All nodes boot and show console access
  </truths>
</must_haves>

<artifacts_this_phase_produces>
- GNS3 project directory (FortiGate-SDWAN-Lab/)
- GNS3 topology file
</artifacts_this_phase_produces>
---
phase: "01"
plan: "02"
type: "configuration"
wave: 1
depends_on: [01-PLAN]
files_modified: []
autonomous: true
requirements: [GNS3-01, GNS3-02]
---

<objective>
Activate permanent trial licenses on both FortiGate VMs and verify OVS Docker appliance is operational with all ports on br0.
</objective>

<read_first>
- FORTIGATE-REFERENCE.md — license activation commands, OVS verification
</read_first>

<tasks>
<task>
<type>configuration</type>
<action>Activate permanent trial license on FGT-Primary

1. Console into FGT-Primary
2. Run: `execute vm-license`
3. When prompted, enter FortiCloud account email (Account 1)
4. Enter FortiCloud account password
5. Wait for license activation confirmation
6. Reboot: `execute reboot`
7. After reboot, verify license: `get system license | grep -i "permanent"`
</action>
<acceptance_criteria>
- `get system license` shows `permanent` trial type (not 14-day)
- License shows no expiry date
- VM resource limits shown: 1 vCPU, 2048 MB, 3 interfaces, 3 policies, 3 routes
</acceptance_criteria>
</task>

<task>
<type>configuration</type>
<action>Activate permanent trial license on FGT-Secondary

1. Console into FGT-Secondary
2. Run: `execute vm-license`
3. When prompted, enter FortiCloud account email (Account 2 — must be different account)
4. Enter FortiCloud account password
5. Wait for license activation confirmation
6. Reboot: `execute reboot`
7. After reboot, verify license: `get system license | grep -i "permanent"`
</action>
<acceptance_criteria>
- Same as FGT-Primary — permanent trial license confirmed
- Uses a different FortiCloud account than FGT-Primary
</acceptance_criteria>
</task>

<task>
<type>verification</type>
<action>Verify OVS Docker appliance is operational

1. Console into OVS-Core
2. Run: `ovs-vsctl show`
3. Verify output shows a bridge named `br0`
4. Verify `br0` has 16 ports listed
5. Run: `ovs-ofctl dump-ports br0`
6. Verify all 16 ports show (even if RX/TX counts are zero)
</action>
<acceptance_criteria>
- `ovs-vsctl show` output includes `Bridge br0` with `Port br0` and 15 additional `Port` entries (total 16)
- No error messages in OVS output
</acceptance_criteria>
</task>
</tasks>

<verification>
1. FGT-Primary: `get system license` shows permanent trial
2. FGT-Secondary: `get system license` shows permanent trial from separate account
3. OVS-Core: `ovs-vsctl show` confirms 16-port br0
</verification>

<success_criteria>
- [ ] Both FortiGates show valid permanent trial license
- [ ] OVS br0 with 16 ports operational
- [ ] Different FortiCloud accounts used for each FortiGate
</success_criteria>

<must_haves>
  <truths>
    - FGT-Primary permanent trial license activated and verified
    - FGT-Secondary permanent trial license activated and verified
    - OVS br0 bridge with 16 ports confirmed operational
  </truths>
</must_haves>

<artifacts_this_phase_produces>
- FortiCloud account: Account 1 (FGT-Primary)
- FortiCloud account: Account 2 (FGT-Secondary)
</artifacts_this_phase_produces>
---
phase: "01"
plan: "03"
type: "configuration"
wave: 2
depends_on: [02-PLAN]
files_modified: []
autonomous: true
requirements: [GNS3-03, GNS3-01]
---

<objective>
Configure WAN1 internet access and verify connectivity, then document and test config backup/restore workflow.
</objective>

<read_first>
- FORTIGATE-REFERENCE.md — static route, ping, config backup/restore
</read_first>

<tasks>
<task>
<type>configuration</type>
<action>Configure WAN1 interface and default route on FGT-Primary

1. Console into FGT-Primary
2. Configure port1 as WAN interface:
   ```
   config system interface
     edit "port1"
       set mode dhcp
       set allowaccess ping
     next
   end
   ```
3. Wait for DHCP lease from GNS3 NAT cloud
4. Verify IP: `show system interface port1`
5. Check default route was auto-added (DHCP gateway):
   ```
   get router info routing-table all
   ```
6. If no default route, add:
   ```
   config router static
     edit 1
       set device "port1"
       set gateway <NAT-gateway-IP>
       set dst 0.0.0.0/0
     next
   end
   ```
7. Test internet: `execute ping 8.8.8.8`
</action>
<acceptance_criteria>
- `show system interface port1` shows an IP in 192.168.122.0/24 range
- `get router info routing-table all` shows S* 0.0.0.0/0 via port1 gateway
- `execute ping 8.8.8.8` returns successful ICMP replies
</acceptance_criteria>
</task>

<task>
<type>documentation</type>
<action>Document and test config backup/restore workflow

1. On FGT-Primary, backup config via TFTP to host:
   ```
   execute backup config tftp config-fgt-primary.conf <host-IP>
   ```
2. Verify file exists on GNS3 host filesystem
3. Make a trivial config change (e.g., set hostname):
   ```
   config system global
     set hostname "FGT-Primary"
   end
   ```
4. Restore backup:
   ```
   execute restore config tftp config-fgt-primary.conf <host-IP>
   ```
5. Verify hostname reverted (confirm restore worked)
6. Repeat steps 1-5 for FGT-Secondary
7. Add backup commands to FORTIGATE-REFERENCE.md if not already present
</action>
<acceptance_criteria>
- Backup file saved to GNS3 host: `config-fgt-primary.conf` and `config-fgt-secondary.conf`
- Restore reverts config change (hostname goes back to default)
- Both backup and restore commands documented in FORTIGATE-REFERENCE.md
</acceptance_criteria>
</task>

<task>
<type>verification</type>
<action>End-to-end connectivity test

1. From FGT-Primary: `execute ping 8.8.8.8` — internet reachable
2. From FGT-Primary: `execute ping <FGT-Secondary-port1-IP>` — local reachable (if connected)
3. `get system arp` shows at least one entry (NAT gateway)
4. `diagnose sniffer packet port1 "icmp" 3 10` captures a ping to 8.8.8.8
</action>
<acceptance_criteria>
- Internet ping succeeds (8.8.8.8)
- Sniffer capture shows ICMP echo request/reply
- ARP table populated with gateway MAC
</acceptance_criteria>
</task>
</tasks>

<verification>
1. FGT-Primary reaches internet via WAN1
2. Config backup file exists on GNS3 host
3. Config restore successfully reverts changes
4. Both FortiGates have documented backup workflows
</verification>

<success_criteria>
- [ ] FGT-Primary pings 8.8.8.8 successfully
- [ ] Config backup written to GNS3 host
- [ ] Config restore verified working
- [ ] Backup/restore commands documented
</success_criteria>

<must_haves>
  <truths>
    - FGT-Primary has internet connectivity via WAN1 NAT cloud
    - Config backup file `config-fgt-primary.conf` exists on host
    - Config restore verified functional
  </truths>
</must_haves>

<artifacts_this_phase_produces>
- config-fgt-primary.conf (backup on host filesystem)
- config-fgt-secondary.conf (backup on host filesystem)
</artifacts_this_phase_produces>
