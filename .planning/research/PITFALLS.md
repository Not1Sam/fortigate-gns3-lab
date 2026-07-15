# Pitfalls Research: FortiGate GNS3 Lab

**Domain:** FortiGate SD-WAN / GNS3 virtual lab with OVS fabric and OCI cloud IPsec
**Researched:** 2026-07-15
**Confidence:** HIGH (multiple authoritative sources, verified against Fortinet documentation)

## Critical Pitfalls

### CP-01: FortiGate VM License — Assuming the 14-Day Evaluation License Still Works

**What goes wrong:**
The VM boots with "License Status: Invalid" and "Evaluation License Expires: 1970/01/01" immediately, before you can do anything useful. No traffic will pass (even basic forwarding) without a valid license. Older tutorials reference a built-in 15-day evaluation period that no longer exists in FortiOS 7.2.x and later — those VMs had an evaluation license baked into the image; new deployments do not.

**Why it happens:**
Starting with FortiOS 7.2.1, Fortinet removed the automatic 15-day evaluation period from fresh VM deployments. The license status now shows "Invalid" on first boot, even with correct 1 CPU / 2 GB RAM configuration. The VM *requires* explicit license activation via FortiCare credentials. This changed silently and many tutorials haven't been updated.

**How to avoid:**
Use Fortinet's **permanent trial (evaluation) license** instead. Register a FortiCloud account, deploy the VM with exactly 1 CPU and 2048 MB RAM, then run:

```
execute vm-license-options account-id your@email.com
execute vm-license-options account-password yourPassword
execute vm-license
```

The permanent trial is limited to 1 CPU, 2 GB RAM, 3 interfaces, 3 policies, 3 routes, and 2 VDOMs — plenty for this lab. It does not expire. Do NOT allocate more than 1 CPU / 2 GB RAM to the VM or the permanent trial will fail to activate.

**Warning signs:**
- `get system status` shows "License Status: Invalid"
- `get system status` shows "Evaluation License Expires: Wed Dec 31 17:00:01 1969" (epoch zero — means no valid time was ever set)
- GUI shows perpetual "Evaluation License" dialog with no option to proceed
- You can't create firewall policies past the first 3

**Phase to address:** Phase 1 (FortiGate VM Deployment & Licensing). Must be resolved before any SD-WAN, IPsec, or policy work.

---

### CP-02: Permanent Trial License Registration Failure — "Forticare Response Error 61" or "Invalid Serial Number"

**What goes wrong:**
Running `execute vm-license` returns "Forticare response error 61" or "Error downloading license: invalid serial number." The VM cannot activate and remains unlicensed.

**Why it happens:**
FortiCare allows only one permanent trial registration per FortiCare account. If you previously deployed a FortiGate VM (even a destroyed one) that registered under that account, the old serial number is still in FortiCare's asset management. The new VM generates a different serial number, and FortiCare rejects it because your account already has a trial claim. This is a permanent association — you can't just delete the old VM.

**How to avoid:**
1. Decommission the old VM license in FortiCare via **support.fortinet.com → Asset Management → Decommission** before deploying a new one.
2. Alternatively, create a second FortiCare account for this lab if you already used the permanent trial on your main account.
3. If you get "error 61," log into FortiCare, find the old trial entry, decommission it, wait 15 minutes, then retry `execute vm-license` on the fresh VM.
4. Use `diagnose debug vm-print-license` to verify your license state post-activation.

**Warning signs:**
- CLI returns "Forticare response error 61" after `execute vm-license`
- GUI license activation spins forever or returns "invalid serial number"

**Phase to address:** Phase 1 (FortiGate VM Deployment & Licensing). Save your FortiCare credentials in the project's secure notes.

---

### CP-03: OVS VLAN Trunking — FortiGate VLAN Sub-Interfaces Not Passing Traffic

**What goes wrong:**
VLAN sub-interfaces are created on the FortiGate side, the OVS bridge is configured with the correct VLAN trunks, but traffic between VLANs doesn't flow or passes in one direction only. The sniffer shows packets arriving on the FortiGate but never leaving the VLAN sub-interface.

**Why it happens:**
Three common causes compound:
1. **OVS trunk misconfiguration:** The OVS bridge port connecting to the FortiGate must be configured as a trunk (tagged) port carrying all required VLANs. If it's set as an access port, the FortiGate never sees the 802.1Q tags.
2. **Missing inter-VDOM or inter-zone policies:** In VDOM mode, VLAN sub-interfaces belong to specific VDOMs. Without explicit policies between the VDOMs (or within the same VDOM between VLANs), traffic is dropped by the implicit deny.
3. **FortiGate VLAN forwarding bug (FortiOS 7.2.x):** A known issue in early 7.2.x releases caused inter-VLAN routing to fail silently — packets arrived on one VLAN interface but were never forwarded to another despite correct policies. Rolling back to 6.4.9 resolved it. 7.4.x fixed this, but verify.

**How to avoid:**
1. Verify OVS port configuration: `ovs-vsctl list port <port-name>` and confirm `tag=[]` (trunk mode with no native VLAN) or `trunks=[10,20,30]`.
2. On the FortiGate, confirm VLAN sub-interfaces are up: `get system interface physical | grep vlan`.
3. Use `diag debug flow` to trace packets between VLANs:
   ```
   diagnose debug flow filter saddr <client-ip>
   diagnose debug flow filter daddr <target-ip>
   diagnose debug flow trace start 100
   diagnose debug enable
   ```
4. If using VDOMs, verify inter-VDOM link configuration and policies exist in both directions.
5. Test with a simple flat topology first (no VDOMs, no zones) to isolate the VLAN forwarding before adding complexity.

**Warning signs:**
- Packets arrive on ingress VLAN interface per sniffer but don't egress
- `diagnose debug flow` shows "deny by policy" even though policies exist
- Inter-VLAN ping works from the FortiGate CLI but not from clients

**Phase to address:** Phase 3 (OVS & VLAN Trunking). Must be verified before Phase 5 (IPsec tunnels).

---

### CP-04: IPsec Tunnel with NAT-T — "NAT Detected: ME PEER" Causes Tunnel Flap or No Traffic

**What goes wrong:**
IPsec Phase 1 establishes, then immediately disconnects. IKE debug shows "NAT detected: ME" and "NAT detected: PEER" on both ends. The tunnel goes up and down in a loop, or Phase 2 never establishes. Alternatively, the tunnel shows "up" but no traffic passes.

**Why it happens:**
The FortiGate VM behind GNS3's NAT node (or a cloud NAT) detects that both peers are behind NAT devices. NAT-T (NAT Traversal) auto-enables and wraps ESP in UDP 4500. However, when both sides detect NAT, the NAT-T negotiation can conflict with the NAT device's port mapping — especially with GNS3's NAT node which uses a simple iptables MASQUERADE rule. The NAT device assigns a dynamic source port, the peer sees traffic from an unexpected port, and the IKE negotiation fails to maintain state. In OCI, the security list rules for UDP 500/4500 may be too restrictive or conflicting.

**How to avoid:**
1. If both ends are behind NAT (GNS3 side behind your home router → OCI VPS with public IP), the OCI side should NOT see NAT. Verify OCI has a public IP directly attached (not through another NAT).
2. On the GNS3-side FortiGate, explicitly enable `set nat-traversal enable` in Phase 1 config.
3. On OCI, add security list / network security group rules for:
   - UDP 500 (IKE)
   - UDP 4500 (IPsec NAT-T)
   - IP protocol 50 (ESP — optional if NAT-T wraps it)
4. If OCI-side also shows NAT detected, your FortiGate VPS may have an extra NAT layer (OCI NAT gateway or public IP on a separate VNIC). Fix this by using a public IP directly on the OCI instance's primary VNIC.
5. As a fallback for stubborn cases: disable NAT-T (`set nat-traversal disable`) and rely on UDP 500 only if you control the intermediary NAT devices.

**Warning signs:**
- `diagnose vpn ike log` shows "NAT detected: ME PEER" or "detected NAT"
- `diagnose vpn tunnel list` shows status "down" with rekey attempts
- IPsec monitor shows tunnel status oscillating between up/down
- `get vpn ipsec tunnel details` shows "natt: mode=keepalive" with no traffic

**Phase to address:** Phase 5 (IPsec Tunnels). Must coordinate with Phase 6 (OCI Security Configuration).

---

### CP-05: SD-WAN Performance SLA — Probes Fail Because Routes Bypass the Tunnel

**What goes wrong:**
SD-WAN is configured with both IPsec tunnels as members, performance SLA health checks are configured, but all probes show 100% packet loss and both tunnels are marked "dead." The FortiGate removes the static routes associated with the tunnels, and all traffic black-holes.

**Why it happens:**
The performance SLA probe packets originate from the FortiGate itself (source = WAN interface IP). For the probes to go through the IPsec tunnel, the destination of the health check must match a route that points to the tunnel interface. Common causes:
1. The health check probe target's route exists but with a lower metric via a different interface (e.g., default route via WAN1 instead of via tunnel).
2. When using IPsec with SD-WAN, the gateway must be unset or set to the remote tunnel endpoint — NOT the tunnel interface IP itself, or routing loops occur.
3. The probe source IP isn't set to the FortiGate's LAN interface, so the return path doesn't match the tunnel's phase2 selectors.

**How to avoid:**
1. Set the SD-WAN member's source to the FortiGate's LAN/loopback interface IP, not the tunnel or WAN IP:
   ```
   config system sdwan
     config members
       edit <id>
         set source <lan-ip>
         set gateway <remote-tunnel-ip>  (or unset)
   ```
2. Verify the probe target (e.g., 8.8.8.8) routes through the tunnel by checking `get router info routing-table details 8.8.8.8`. If it doesn't, add a static route for the probe target pointing to the tunnel.
3. Set probe-timeout to 2000ms (default 500ms is too low for tunneled probes over high-latency links):
   ```
   config system sdwan
     config health-check
       edit <name>
         set probe-timeout 2000
   ```
4. For cloud probe targets that need to go through the tunnel, ensure OCI-side has a route back and a security list rule permitting ICMP/HTTP from the FortiGate's tunnel IPs.
5. Use `diagnose sys sdwan health-check` to monitor real-time probe results and identify which member is failing.

**Warning signs:**
- `diagnose sys sdwan health-check` shows "state(dead)" or "state(unknown)"
- SD-WAN dashboard shows "link-failure" on tunnel members
- Tunnel is up (per IPsec monitor) but SD-WAN marks it down
- Routes disappear from routing table after SLA failure

**Phase to address:** Phase 7 (SD-WAN Configuration). Requires Phase 5 (IPsec) to be stable first.

---

### CP-06: SSL Deep Inspection — Certificate Not Trusted by Browsers, Breaking All HTTPS

**What goes wrong:**
Full SSL inspection is enabled, traffic is decrypted and re-encrypted, but every HTTPS site shows certificate errors. Users (including GNS3 VPCS and browser hosts) cannot access any HTTPS site. The FortiGate's CA certificate is not trusted by client machines.

**Why it happens:**
FortiGate's default SSL inspection CA certificate ("Fortinet_CA_Untrusted" — the name itself is a hint) is not trusted by any OS or browser. For SSL inspection to work transparently, every client that goes through the inspection must trust the FortiGate's CA certificate. In a GNS3 lab:
1. Lab PCs (VPCS, Linux containers) don't have the FortiGate CA installed.
2. Even if you export the CA, VPCS has no mechanism to trust custom certificates.
3. Browsers on the host machine accessing the internet through the lab also need the CA installed.
4. Without client-side trust, every HTTPS connection shows a warning or fails entirely.

**How to avoid:**
1. Generate a proper CA certificate on the FortiGate (not the default):
   ```
   config vpn certificate ca
     edit "MyLab-CA"
       set private-key <rsa-key>
       set certificate <ca-cert>
   ```
2. Export the CA certificate from **System → Certificates → CA Certificates → Export**.
3. Distribute the CA to every client that needs SSL inspection. This is straightforward for the GNS3 host machine (import to system trust store) but nearly impossible for VPCS nodes.
4. **For lab purposes:** Use "SSL Certificate Inspection" instead of "Full SSL Inspection" unless you specifically need to inspect HTTPS content. Certificate inspection checks the certificate validity without decrypting the payload and doesn't require client-side CA trust.
5. If you must test full inspection, use a browser on the GNS3 host (which can have the CA installed) and access sites that don't use certificate pinning (HPKP).
6. In FortiOS 7.4, note that flow-based SSL inspection with TLS 1.3 Kyber (post-quantum key exchange) can break — switch to proxy-based mode for problematic sites.

**Warning signs:**
- Browser shows "NET::ERR_CERT_AUTHORITY_INVALID" or similar
- WAD debug shows "SSL proxy failure" with CA validation errors
- Only HTTP (not HTTPS) sites load correctly

**Phase to address:** Phase 8 (NGFW & SSL Inspection). Requires client-side trust documentation in the lab guide.

---

### CP-07: VDOM Mode — Management Access Breaks After Enabling VDOMs

**What goes wrong:**
You enable VDOMs with `set vdom-admin enable` and the FortiGate restarts. After reboot, you cannot access the GUI or SSH. The FortiGate appears to be running (you can ping its management IP) but all management connections are refused.

**Why it happens:**
Enabling VDOMs moves the FortiGate into multi-VDOM mode. Interface configurations, administrative access permissions, and admin accounts are now VDOM-scoped by default. The root VDOM retains global management capabilities, but:
1. If the management interface wasn't explicitly assigned to the root VDOM, it may become "unmanaged."
2. Administrative access protocols (HTTPS, SSH, PING) are reset per-interface when VDOMs are enabled and need to be re-configured.
3. Trusted host restrictions may be too narrow for the lab network.
4. If the admin account is a "super_admin" within a non-root VDOM, it cannot see global settings or other VDOMs.

**How to avoid:**
Before enabling VDOMs:
1. Ensure your management interface is in the root VDOM and has `set allowaccess ping https ssh` enabled.
2. Remove any trusted-host restrictions temporarily until VDOMs are stable.
3. After enabling VDOMs, log into the root VDOM (select "Global" from the VDOM dropdown in GUI, or use `config global` in CLI).
4. Assign interfaces to VDOMs explicitly — don't leave any interface unattached.
5. Create at minimum: root (admin VDOM), Edge-VDOM, and Internal-VDOM.
6. Verify management access from the root VDOM before switching context to traffic VDOMs.
7. Keep a console session open (GNS3 console) while testing management access — it's your escape hatch if you lock yourself out.

**Warning signs:**
- GUI login returns "Connection refused" after VDOM enable
- SSH times out despite FortiGate being pingable
- "Administrator access is denied" errors in logs
- Cannot access VDOM-specific interfaces from management network

**Phase to address:** Phase 9 (VDOM Implementation). Must occur after Phase 1 and before Phase 3 (interface assignments).

---

### CP-08: OCI Security List — IPsec Tunnel Establishes (Phase 1) But No Traffic Passes

**What goes wrong:**
IPsec Phase 1 and Phase 2 establish successfully. The tunnel shows "up" on both ends. But no application traffic passes through the tunnel — pings fail, HTTP doesn't work, everything times out.

**Why it happens:**
This is almost always a **security list** or **route table** issue on the OCI side, not an IPsec problem:
1. OCI security lists are **stateless** by default. Outbound traffic rules must explicitly match return traffic's source/destination. ESP (protocol 50) and UDP 4500 rules are often incomplete.
2. OCI route tables: The subnet hosting the FortiGate VPS must have a route rule pointing the lab's private subnet (e.g., 192.168.x.x) back to the FortiGate VPS's private IP — not its public IP. Without this, return traffic from the OCI instance goes through the internet gateway instead of the tunnel.
3. If the OCI instance has OS-level firewall (iptables/nftables), it may block forwarded traffic even if OCI security lists permit it.
4. NAT on the tunnel: The GNS3 FortiGate may be NATting the tunnel traffic, which OCI's security list doesn't expect, breaking return packets.

**How to avoid:**
1. OCI security list **egress rules** for the VPS subnet **must** allow:
   - Destination: 192.168.0.0/16, Protocol: All (or ICMP + UDP 500/4500 + ESP)
   - Without this, response traffic from OCI services back to your lab is dropped before it enters the tunnel.
2. Add a **route table rule** on the OCI VCN for the lab subnet: Destination = 192.168.x.x/16, Route Target = the FortiGate VPS's **private** IP (not the public IP).
3. On the OCI FortiGate-side, create firewall policies that permit the return traffic. If using a simple iptables-based VPS, add:
   ```
   iptables -A FORWARD -i <tunnel-if> -j ACCEPT
   iptables -A FORWARD -o <tunnel-if> -j ACCEPT
   iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
   ```
4. Avoid NATting traffic that goes through the IPsec tunnel on the GNS3 side — the tunnel should transport the original private IPs so OCI routes return traffic correctly.
5. Test with `ping` and `traceroute` from the FortiGate CLI to the OCI VPS's tunnel IP before testing from lab clients.

**Warning signs:**
- Tunnel is "up" from `get vpn ipsec tunnel details` but `ping <oci-tunnel-ip>` from FortiGate times out
- `diagnose sniffer packet` shows packets leaving the tunnel interface but no return packets
- OCI VPS shows ESP/UDP 4500 packets arriving in tcpdump but no response sent
- OCI security list flow logs show "DENY" for return traffic

**Phase to address:** Phase 6 (OCI Security Configuration). Must be verified before Phase 7 (SD-WAN).

---

### CP-09: GNS3 NAT Node — Connectivity Is Unidirectional (Can Ping Out, But Not In)

**What goes wrong:**
The FortiGate can ping the internet (8.8.8.8) from its CLI, and the GNS3 NAT node appears operational. But LAN-side clients behind the FortiGate cannot reach the internet. Traffic reaches the FortiGate (verified by sniffer) but no response comes back.

**Why it happens:**
The GNS3 NAT node (or the underlying GNS3 VM's libvirt virbr0) provides a simple outbound NAT. It works for traffic that originates from the NAT node's directly connected network. But when the FortiGate's WAN interface is behind the NAT node, and the FortiGate is doing its own NAT for LAN clients:
1. **Double NAT:** The FortiGate NATs 192.168.x.x → 192.168.122.x (NAT node subnet), then the NAT node NATs 192.168.122.x → host IP. The return path has two layers of address translation to unwind, and state tracking breaks at one of the hops.
2. **Default route missing:** The FortiGate needs a default route pointing to the NAT node's IP (typically 192.168.122.1) on the WAN interface.
3. **Policy misconfiguration:** The LAN→WAN policy doesn't have NAT enabled, or enables NAT on the wrong interface.
4. **GNS3 server firewall:** The host machine's firewall blocks forwarded traffic between GNS3 VM virtual networks and the physical network.

**How to avoid:**
1. Check the FortiGate's default route: `get router info routing-table all`. It must have `S* 0.0.0.0/0 [10/0] via 192.168.122.1, portX`.
2. Verify the LAN→WAN policy has NAT enabled (Outgoing Interface Address or IP Pool).
3. Simplify: Use the GNS3 Cloud node instead of the NAT node when possible, connecting directly to the host's physical NIC or a bridge. This eliminates one layer of NAT.
4. If using the NAT node, confirm the GNS3 VM has libvirt installed and `virbr0` is up (`ip addr show virbr0`).
5. Check the host firewall isn't blocking GNS3 traffic: `iptables -L -n` for FORWARD chain rules.
6. Test incrementally: FortiGate CLI ping → WAN interface ping → LAN client ping via diagnostic tools.

**Warning signs:**
- FortiGate can ping 8.8.8.8 from CLI but LAN clients cannot
- `diagnose debug flow` shows traffic entering FortiGate on LAN interface, leaving on WAN interface, but no return traffic
- `diagnose sniffer packet` shows ICMP echo request on WAN interface but no echo reply
- GNS3 NAT node shows no forwarded packets in its status

**Phase to address:** Phase 2 (Basic GNS3 Topology & Internet Connectivity). Must work before any cloud-dependent features.

---

### CP-10: FortiOS 7.4.x — Inter-VLAN Routing Broken on Specific Builds

**What goes wrong:**
Inter-VLAN routing stops working after a FortiOS upgrade or on a fresh 7.4.x deployment. Clients in VLAN 10 can ping the VLAN 10 gateway but cannot reach clients in VLAN 20, despite correct firewall policies allowing the traffic. The sniffer shows packets arriving on VLAN 10 interface but never being forwarded to VLAN 20.

**Why it happens:**
FortiOS 7.2.0 through early 7.4.x had a confirmed bug (documented in Fortinet community discussions) where inter-VLAN routing silently failed. Packet flow debugging showed the FortiGate received the packet, matched the correct policy, but never forwarded it out the destination VLAN interface. Rolling back to 6.4.9 resolved the issue in multiple reports. This was fixed in later 7.4.x patches, but specific builds (7.2.0, 7.4.2) are known to be affected. The exact root cause was related to the kernel routing table not properly binding VLAN sub-interfaces.

**How to avoid:**
1. Use FortiOS **7.4.12** (the version specified in PROJECT.md) — this is a mature patch level where the inter-VLAN bug is confirmed resolved. Do NOT use 7.4.0, 7.4.1, or 7.4.2.
2. If you encounter the issue despite using a recent build, verify with `diagnose debug flow` that packets are matching the correct policy but not being forwarded. If so, check FortiOS release notes for known unresolved issues.
3. Workaround if encountered: Use a dedicated VLAN interface on the OVS switch instead of FortiGate VLAN sub-interfaces for inter-VLAN routing, or add explicit static routes for each VLAN subnet.
4. Before upgrading FortiOS, always export a full configuration backup and keep the previous image file accessible.

**Warning signs:**
- Inter-VLAN pings fail despite correct policies and interface configuration
- `diagnose debug flow` shows "deny by policy" with no matching deny rule, or shows "forward" but packet never appears on egress
- Same configuration works after downgrading to 6.4.9
- The bug is specific to certain build numbers — check `get system status` for build number

**Phase to address:** Phase 3 (OVS & VLAN Trunking), but verified during Phase 1 (choose correct FortiOS version: 7.4.12).

---

### CP-11: FortiGate VM Resource Constraints — Permanent Trial Limitations That Break the Lab

**What goes wrong:**
After activating the permanent trial license, you discover you cannot:
- Create a 4th firewall policy (required for more complex routing)
- Add a 4th static route
- Use more than 3 network interfaces
- Enable FortiGuard services for IPS/Web Filtering/App Control
- Allocate more than 1 vCPU or 2 GB RAM

**Why it happens:**
The permanent trial license explicitly limits: 1 CPU, 2 GB RAM, 3 interfaces, 3 firewall policies, 3 routes, no FortiGuard services. This is sufficient for basic connectivity but may be too restrictive for a full SD-WAN + IPsec + NGFW lab that requires multiple policies (LAN→WAN, LAN→DMZ, WAN→DMZ, IPsec→LAN, etc.) and routes (default, IPsec tunnel 1, IPsec tunnel 2, OCI subnet).

**How to avoid:**
1. Design your policy and route budget upfront:
   - Policy 1: LAN→WAN (internet)
   - Policy 2: LAN→IPsec (to OCI)
   - Policy 3: IPsec→LAN (return traffic)
   - ⚠️ No room for a separate DMZ policy, SSL inspection exemption, or management access policy.
2. **Workaround:** Use firewall address groups and consolidated policies to combine rules. For example, put both IPsec tunnels in a single policy using a service-based approach.
3. **Route budget:** Route 1 = default via WAN, Route 2 = OCI subnet via Tunnel 1, Route 3 = OCI subnet via Tunnel 2 (floating metric) — that exhausts your budget. No room for a management subnet route.
4. If you absolutely need more resources, consider purchasing a VM license or using the full evaluation (14-day) from an older FortiOS version (6.4.x) that didn't have these restrictions — but 6.4.x won't support all SD-WAN features.
5. Alternatively, use a single VDOM (no VDOM separation) to save policy overhead.
6. For FortiGuard-dependent features (IPS, Web Filtering), you cannot test them on the permanent trial. These require a paid subscription. Plan your lab scope accordingly — or use a second FortiGate with a 14-day trial for these features.

**Warning signs:**
- `get system status` shows "License Status: Valid" but you cannot create a 4th policy
- "Maximum number of routes reached" error when adding a static route
- "Cannot attach more interfaces" error when connecting OVS to a 4th port

**Phase to address:** Phase 1 (FortiGate VM Deployment & Licensing). Design policy budget before Phase 4 (Firewall Policies).

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Use the default self-signed CA for SSL inspection | Saves 5 minutes generating a proper CA | All HTTPS breaks on clients without the CA; training/remediation later | Never — always generate a proper CA and distribute it |
| Skip static routes on OCI for return traffic | Tunnel comes up faster | No traffic passes; debugging takes hours | Never — routing must be bidirectional for IPsec to work |
| Use 0.0.0.0/0 everywhere (any-any policies) | Quick connectivity test | Loses all firewall benefit; hard to troubleshoot later | Only as a *temporary* test, clearly labeled, with a planned expiration |
| Allocate 4 GB RAM to the FortiGate VM | Faster boot time | Permanent trial license activation fails if > 2 GB | Never for trial-licensed VMs |
| Use NAT node instead of Cloud node | Simpler setup, no bridging | Double NAT breaks complex topologies; no inbound access | Only if the host has only one NIC and bridging isn't possible |
| Skip configuration backups | No overhead in early stages | Hours lost rebuilding if VM corrupts or license needs re-deployment | Never — always export config after each milestone |
| Enable VDOMs on Day 1 | Segmentation from the start | Management lockout risk; debugging complexity increases | Wait until Phase 9 after basic connectivity is stable |
| Use `set asymroute enable` for traffic issues | Quick fix for asymmetric routing | Disables all UTM features; FortiGate becomes stateless | Never in production; use only as diagnostic, then fix routing |
| Disable NAT-T for IPsec | Simple tunnel setup | Breaks if any NAT device exists between peers | Only if you control all devices between peers and know there's no NAT |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| **GNS3 ↔ FortiGate VM** | Using default QEMU settings without adjusting for KVM acceleration | Enable KVM (if available on host): set `qemu.kvm=enable` in GNS3 VM configuration; verify `/dev/kvm` exists and is accessible |
| **OVS ↔ FortiGate** | Connecting FortiGate directly to OVS bridge without matching port configurations | Both sides must agree on trunk mode. OVS: `set port <name> trunks=10,20,30`. FortiGate: ensure parent interface is the OVS-connected port, VLAN ID matches. |
| **GNS3 ↔ Host network** | Using Cloud node with the wrong VMnet/virtual interface | Match the GNS3 Cloud node to the specific host interface or bridge (e.g., `vmnet8` for NAT, `br0` for bridged). Test with `ping` from a simple VPCS first. |
| **FortiGate ↔ OCI IPsec** | Configuring the OCI endpoint with the FortiGate's public IP when it's behind NAT | The OCI peer must point to the GNS3 side's *public* IP (your home router's WAN IP). If dynamic, use DDNS. For the local FortiGate, use the LAN IP of the NAT gateway as the "local gateway." |
| **GNS3 ↔ OCI API** | Opening OCI API ports (e.g., 443) in security list but not checking instance-level firewall | OCI instances have both security lists AND OS firewalls (iptables). Open both. Check `iptables -L -n` on the OCI instance. |
| **FortiGate ↔ Browser (GUI access)** | Trying to use HTTPS with the default self-signed cert | Evaluation license only supports low encryption. Use `set allowaccess http` on the management interface and access via HTTP instead of HTTPS. Or generate a proper self-signed cert. |
| **VPCS ↔ Anything** | VPCS has no CA store and no browser | VPCS cannot test SSL inspection. Use a Linux container (WebTerm) with the FortiGate CA installed for HTTPS testing. |
| **FortiGate ↔ FortiGuard** | Expecting FortiGuard services to work on the permanent trial | They won't. The permanent trial explicitly disables FortiGuard. You cannot test IPS, Web Filtering, or App Control beyond the built-in signatures without a paid subscription. |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| **GNS3 VM RAM exhaustion** | FortiGate boots slowly, console hangs, QEMU processes get killed | Allocate at least 8 GB RAM to GNS3 VM for 2 FortiGates + OVS + Docker. 4 GB will OOM. | With 2 FortiGates and any additional nodes |
| **Single vCPU on FortiGate** | High CPU usage during IPsec encryption, slow management response | Permanent trial caps at 1 vCPU — design around this; avoid running heavy processes concurrently (e.g., don't run IDS traffic and VPN rekey simultaneously) | Under IPsec traffic load with multiple tunnels |
| **3-policy budget** | Cannot add a 4th essential policy | Consolidate: combine IPsec tunnels into one policy; use address groups | Halfway through lab when you need policy for DMZ, management, or SSL exemption |
| **NAT node throughput** | Throughput drops significantly, packet loss > 5% on any traffic > 10 Mbps | Use Cloud node with bridging instead of NAT node for higher throughput | When transferring files or running speed tests |
| **OVS with many VLANs** | CPU usage spikes on GNS3 VM, STP convergence issues | Keep VLAN count ≤ 5 for a lab; don't enable STP unless testing it specifically | Beyond 10 VLANs in a single OVS bridge |
| **SSL Deep Inspection on low-resource VM** | WAD process consumes 50%+ CPU, proxy mode hangs | Use "certificate inspection only" unless specifically testing deep inspection; switch to flow-based mode | On the permanent trial's single vCPU with > 50 concurrent HTTPS sessions |
| **Multiple IPsec tunnels to same peer** | Phase 2 selectors conflict, tunnel instability | Use unique phase2 selectors per tunnel, or use a single tunnel with multiple phase2 entries | When both tunnels have the same local/remote subnet selectors |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| **Leaving admin password empty on first boot** | Anyone with console access owns the lab | Set a strong admin password immediately. The default prompt forces this, but don't bypass it. |
| **Enabling HTTP admin access without restriction** | Credentials sent in cleartext across the OVS network | Use HTTPS for GUI access. Only enable HTTP temporarily for initial setup when the self-signed cert causes browser issues. |
| **Using the same VLAN ID for management and data traffic** | Management traffic exposed on data VLANs | Use a dedicated management VLAN or at minimum separate VLANs by trust zone. |
| **Not restricting trusted hosts for admin access** | Any device on the network can attempt to brute-force admin credentials | Configure `set trustedhost <mgmt-subnet>` on admin accounts. |
| **Publishing FortiCare credentials in scripts or config backups** | Account compromise, permanent trial license lost | Store credentials in a secure note, never in Git-tracked files. Use `.gitignore` for config exports. |
| **Exposing OCI API credentials in GNS3 scripts** | Cloud account compromise possible | Use OCI API keys or instance principals, not root credentials. Store in environment variables. |
| **Disabling security features for convenience** | Lab becomes non-representative | Document what you disabled and why. The lab should mirror real deployment constraints where possible. |

---

## "Looks Done But Isn't" Checklist

- **[IPsec Tunnel]:** Tunnel Phase 1 and Phase 2 show "up," but verify `ping <oci-private-ip>` works from the FortiGate CLI. Many people stop at "tunnel up" and don't test actual through-tunnel connectivity.
- **[SD-WAN]:** SD-WAN is configured and members are added, but check `diagnose sys sdwan health-check` — a tunnel showing "dead" because the SLA probe doesn't traverse it will silently break SD-WAN failover.
- **[VLANs]:** VLAN sub-interfaces are created and IPs are assigned, but test **bidirectional** traffic between VLANs. A common failure mode is traffic flowing one way only (VLAN10→VLAN20 works but return doesn't).
- **[SSL Inspection]:** SSL inspection policy exists and HTTPS seems to work, but check that certificate inspection vs. full inspection is configured correctly. If clients see certificate errors, inspection isn't transparent.
- **[NAT Policy]:** LAN→WAN policy has NAT enabled, but verify with `diagnose sniffer packet` that the source IP is actually being translated. A missing NAT checkbox is the #1 cause of "no internet from LAN."
- **[VDOM]:** VDOMs are created and interfaces are assigned, but verify management access from each VDOM separately. A VDOM without admin access enabled on its interfaces is isolated.
- **[OVS Trunk]:** The OVS trunk port is configured, but verify with `ovs-vsctl list port <name>` that the trunks list includes ALL required VLANs. A missing VLAN ID silently drops all traffic for that VLAN.
- **[Config Backup]:** You've configured many features, but have you exported the config? `execute backup config tftp <server> <filename>` — do this after every milestone.

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| **Permanent trial expired/** | HIGH (must rebuild) | 1. Decommission old license in FortiCare Asset Management. 2. Delete VM's disk image. 3. Deploy fresh VM. 4. Re-register. 5. Restore config backup. |
| **VDOM lockout** | MEDIUM | 1. Console into FortiGate (GNS3 serial console always works). 2. Enter `config global`. 3. Re-enable admin access on management interface. 4. Verify trusted hosts. |
| **IPsec tunnel flapping due to NAT** | LOW | 1. Check NAT-T settings on both peers. 2. Ensure OCI has public IP directly on VNIC. 3. Adjust security lists for UDP 500/4500. |
| **SD-WAN marks all tunnels dead** | LOW | 1. Check probe target is routable through tunnel. 2. Verify SD-WAN member source/gateway config. 3. Increase probe-timeout. |
| **OVS VLAN traffic not passing** | LOW | 1. Verify OVS port trunk config. 2. Verify FortiGate VLAN interface status. 3. Test with flat (non-VLAN) topology first. |
| **SSL inspection breaking everything** | LOW | 1. Temporarily switch to certificate inspection. 2. Export and distribute CA cert. 3. Re-enable full inspection. |
| **GNS3 node corruption** | MEDIUM | 1. Stop the project. 2. Delete the specific node (not all project files). 3. Re-add from template. 4. Restore config from backup. |
| **FortiOS upgrade failure** | HIGH | 1. Keep previous firmware image. 2. Boot from old image. 3. Restore pre-upgrade config backup. 4. Follow recommended upgrade path for next attempt. |

---

## Pitfall-to-Phase Mapping

| Pitfall | Key Prevention Phase | Verification |
|---------|---------------------|--------------|
| CP-01: License assumption | Phase 1: FortiGate VM & Licensing | Run `get system status`, verify "License Status: Valid" |
| CP-02: License registration failure | Phase 1: FortiGate VM & Licensing | `execute vm-license` returns success, not error 61 |
| CP-03: OVS VLAN trunking | Phase 3: OVS & VLAN Trunking | Bidirectional ping test between VLANs passes |
| CP-04: IPsec NAT-T | Phase 5: IPsec Tunnels (verify with Phase 6: OCI) | `diagnose vpn tunnel list` shows both tunnels UP with traffic |
| CP-05: SD-WAN SLA probes | Phase 7: SD-WAN Configuration | `diagnose sys sdwan health-check` shows "state(alive)" for both tunnels |
| CP-06: SSL inspection cert | Phase 8: NGFW & SSL Inspection | HTTPS sites load without cert errors from CA-trusted client |
| CP-07: VDOM management access | Phase 9: VDOM Implementation | SSH/GUI access works from each VDOM after VDOM enable |
| CP-08: OCI security lists | Phase 6: OCI Security Configuration | Ping from GNS3 to OCI private IP through tunnel succeeds |
| CP-09: GNS3 NAT node | Phase 2: Basic GNS3 Topology | LAN client can browse internet through FortiGate |
| CP-10: FortiOS 7.4 inter-VLAN bug | Phase 1 (version choice) + Phase 3 | Inter-VLAN ping test works (see CP-03 verification) |
| CP-11: Trial resource limits | Phase 1 (planning) | Policy/route budget stays within 3/3 limit |

---

## Sources

- **Fortinet Documentation:** Permanent Trial License Limitations, FortiOS 7.4.12 Release Notes, IPsec NAT-T KB, SD-WAN Performance SLA Guide, SSL Inspection Guide
- **Fortinet Community:** "FortiOS 7.4.2 Bug Causes IPsec VPN Tunnel Phase 2 Instability" (andybarker, Jan 2024), "Inter-VLAN routing issues" (chethan, May 2022), "Issues with GNS3 and FortiOS" (rarac, Dec 2023), "Technical Tip: How to add a FortiGate VM into the GNS3" (kajlasunil, Jan 2023)
- **GNS3 Documentation:** NAT Node Guide, Connect GNS3 to Internet, Troubleshooting FAQ
- **GetLabsDone:** "Build a FortiGate lab using GNS3" (Saifudheen Sidheeq, Nov 2023), "GNS3 Common Errors" (Mar 2021)
- **Grandmetric:** "FortiGate Configuration: Common Mistakes" (Feb 2026)
- **Community Discussion Threads:** GNS3 FortiGate evaluation license bypass, GNS3 Cloud node connectivity, OCI IPsec routing, Twin connections in NAT-T
- **Verified Against:** Fortinet KB articles for VM licensing (updated Dec 2024), OCI networking documentation, Open vSwitch documentation

---

*Pitfalls research for: FortiGate SD-WAN / GNS3 Lab*
*Researched: 2026-07-15*
