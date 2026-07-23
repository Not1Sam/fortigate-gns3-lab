# FortiGate 7.4.12 Reference

Living document — updated whenever new commands or info are discovered.

---

## Eval License (Permanent Trial)

Activated via FortiCloud/FortiCare account. One per account, can reuse after decommissioning old VM.

**Limits:**
- 1 vCPU, 2 GB RAM
- Max 3 interfaces (incl. disabled, loopback)
- Max 3 firewall policies
- Max 3 static routes
- Max 2 VDOMs
- Low encryption only (GUI/HTTPS still normal)
- No FortiGuard, no FortiCare support

**CLI license check:**
```
get system status
execute vm-license
execute vm-license-options account-id <email>
execute vm-license-options account-password <pass>
execute vm-license
```

---

## System Commands

| Command | What it does |
|---------|-------------|
| `get system status` | Version, serial, license, uptime |
| `get system performance status` | CPU/memory usage |
| `get system interface physical` | All physical interfaces |
| `get hardware nic <int>` | Interface details |
| `diagnose sys top` | Process list (P=sort CPU, M=sort mem) |
| `diagnose sys top-summary` | Easier top with bars |
| `execute reboot` | Reboot |
| `execute tac report` | Generate support report |
| `diagnose debug crashlog read` | Read crash logs |
| `diagnose sys process pidof <daemon>` | Find PID of a daemon |

---

## Interface Commands

```
config system interface
  edit "port1"
    set mode static
    set ip 192.168.1.1/24
    set allowaccess ping https ssh
  next
end

show system interface
```

**VLAN subinterface** (needed to stretch 3-intf limit):
```
config system interface
  edit "port3.10"
    set vdom "root"
    set type vlan
    set vlanid 10
    set interface "port3"
    set ip 192.168.10.1/24
    set allowaccess ping
  next
end
```

**Diagnose interfaces:**
```
diagnose ip address list
diagnose netlink interface list
get system arp
diagnose ip arp list
fnsysctl ifconfig port1
```

---

## Routing

| Command | What it does |
|---------|-------------|
| `get router info routing-table all` | Full routing table |
| `get router info routing-table static` | Static routes only |
| `get router info routing-table ospf` | OSPF routes |
| `diagnose ip route list` | Kernel routing table |
| `diagnose ip rtcache list` | Route cache |

**Static route:**
```
config router static
  edit 1
    set device "port1"
    set gateway 192.168.1.254
    set dst 0.0.0.0/0
  next
end
```

**OSPF:**
```
config router ospf
  set router-id 1.1.1.1
  config area
    edit 0.0.0.0
    next
  end
  config network
    edit 1
      set prefix 192.168.10.0/24
      set area 0.0.0.0
    next
  end
end
```

---

## Firewall Policies

**Max 3 on eval license.** Must consolidate aggressively.

```
config firewall policy
  edit 1
    set name "LAN-to-WAN"
    set srcintf "port3.10"
    set dstintf "port1"
    set srcaddr "LAN-10"
    set dstaddr "all"
    set action accept
    set schedule "always"
    set service "ALL"
    set nat enable
  next
end
```

**Show policies:**
```
show firewall policy
get firewall policy
diagnose firewall iprope list
```

**Session table:**
```
diagnose sys session filter
diagnose sys session list
diagnose sys session clear
```

---

## IPsec VPN

**Phase 1 (IKE):**
```
config vpn ipsec phase1-interface
  edit "to-oci-1"
    set interface "port1"
    set ike-version 2
    set keylife 86400
    set peertype any
    set net-device disable
    set proposal aes256-sha256
    set dhgrp 14
    set remote-gw <OCI-public-IP>
    set psksecret <pre-shared-key>
  next
end
```

**Phase 2 (IPsec):**
```
config vpn ipsec phase2-interface
  edit "to-oci-1-p2"
    set phase1name "to-oci-1"
    set proposal aes256-sha256
    set pfs enable
    set dhgrp 14
    set auto-negotiate enable
    set src-subnet 192.168.10.0/24
    set dst-subnet 172.16.0.0/16
  next
end
```

**Diagnose IPsec:**
| Command | What it does |
|---------|-------------|
| `diagnose vpn ike gateway list` | IKE gateways |
| `diagnose vpn tunnel list` | Tunnel status |
| `diagnose vpn ike log` | IKE debug |
| `diagnose debug application ike -1` | Full IKE debug |
| `diagnose debug enable` | Start debug output |
| `diagnose debug reset` | Stop all debug |
| `execute ping-options source <ip>` | Ping from specific IP |

---

## SD-WAN

```
config system sdwan
  set status enable
  config members
    edit 1
      set interface "to-oci-1"
      set gateway <peer-tunnel-IP>
    next
    edit 2
      set interface "to-oci-2"
      set gateway <peer-tunnel-IP>
    next
  end
  config health-check
    edit "oci-probe"
      set server <OCI-internal-IP>
      set interval 500
      set members 1 2
    next
  end
end
```

**Diagnose SD-WAN:**
| Command | What it does |
|---------|-------------|
| `diagnose sys sdwan health-check` | SLA probe results |
| `diagnose sys sdwan members` | Member status |
| `diagnose sys sdwan service list` | SD-WAN rules |

---

## VDOM

```
config system global
  set vdom-mode multi-vdom
end
<reboots>

config vdom
  edit "Edge-VDOM"
  next
  edit "Internal-VDOM"
  next
end

config system vdom-link
  edit "to-internal"
    set vdom "Internal-VDOM"
  next
  edit "to-edge"
    set vdom "Edge-VDOM"
  next
end
```

**Diagnose VDOM:**
| Command | What it does |
|---------|-------------|
| `get system status` | Shows current VDOM |
| `config vdom` | List/switch VDOMs |
| `diagnose vdom list` | VDOM list |

---

## SSL Inspection

```
config firewall ssl-ssh-profile
  edit "deep-inspection"
    config https
      set ports 443
      set status deep-inspection
    end
  next
end

config vpn certificate ca
  edit "Fortinet_CA"
    set certificate <ca-cert-in-pem>
  next
end
```

---

## NGFW Security Profiles

```
config application list
  edit "app-default"
    config entries
      edit 1
        set application 15892 16160 16191 17363
        set log enable
      next
    end
  next
end

config webfilter profile
  edit "web-default"
    config web
      edit "default"
        set enable-log all-url
      next
    end
  next
end

config ips sensor
  edit "ips-default"
    config entries
      edit 1
        set severity critical high medium
        set log enable
      next
    end
  next
end

config firewall policy
  edit 1
    set utm-status enable
    set application-list "app-default"
    set webfilter-profile "web-default"
    set ips-sensor "ips-default"
    set ssl-ssh-profile "deep-inspection"
  next
end
```

---

## Debug & Troubleshooting

### Packet capture
```
diagnose sniffer packet any "host 8.8.8.8" 4 100
diagnose sniffer packet port1 "icmp" 3 10
```

### Flow debug
```
diagnose debug flow filter addr <ip>
diagnose debug flow show function-name enable
diagnose debug enable
diagnose debug flow trace start 100
```

### CPU/Memory
```
diagnose hardware sysinfo memory
diagnose hardware sysinfo cpu
diagnose sys top 5 10
diagnose sys top-summary 5 10
```

### General
```
diagnose debug enable
diagnose debug disable
diagnose debug reset
diagnose autoupdate versions
diagnose log test
```

---

## GNS3 Integration Notes

- VM image: `FGT_VM64_KVM-v7.4.12` → saved as `fgt-v7.4.12.qcow2`
- Template in GNS3 uses QEMU, e1000 adapters, virtio disk, 2048 MB RAM
- Console: telnet
- Serial output shows boot process and CLI
- First login: `admin` / no password, set new password on first login

**Interface mapping in GNS3:** GNS3's port numbering maps 1:1 to FortiGate's port1-port10. The eval license limits to 3 usable interfaces — use VLAN subinterfaces on one trunk port to stretch this.

---

## FortiManager VM Trial

**Not currently deployed** — deferred idea from discussion.

Limits if added later:
- Free trial via FortiCloud/FortiCare account
- Max 3 managed devices/VDOMs
- Max 2 ADOMs
- No FortiAnalyzer features
- No FortiGuard

---

*Last updated: 2026-07-15*
*Next update: Add commands/info as discovered during lab work*
