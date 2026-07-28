---
title: FortiGate 7.0.9 Pre-Licensed Image
tags:
  - lab/fortigate
  - reference/images
---

# FortiGate 7.0.9 Pre-Licensed Image

A `fortios.qcow2` image (2 GB virtual, 74 MB compressed) with FortiOS 7.0.9 build 0444 (GA). Each booted instance gets a **valid eval license** automatically — no FortiCloud registration needed per VM.

## Comparison vs 7.4.12

| Feature | 7.0.9 (Pre-Licensed) | 7.4.12 (Official) |
|---|---|---|
| FortiOS version | 7.0.9 build 0444 | 7.4.12 build 2902 |
| License | Valid eval (auto-renews) | Valid eval (auto-renews) |
| Interface limit | 3 | 3 |
| Policy limit | 3 | 3 |
| Route limit | 3 | 3 |
| Signature DBs | 2015–2018 (old) | Newer |
| OSPF compatibility | Compatible with 7.4.12 | Compatible with 7.0.9 |
| Per-VM registration | Not needed | Required (FortiCloud) |

## Adding to GNS3 on Linux

```bash
cp fortios.qcow2 ~/GNS3/images/QEMU/
```

In GNS3 GUI: `Edit` → `Preferences` → `QEMU VMs` → `New` → Select `fortios.qcow2` → Set RAM 2048 MB, vCPU 1, adapters 8 (`virtio-net-pci`).

## Adding to GNS3 on Windows

1. Copy `fortios.qcow2` to the GNS3 VM's QEMU directory:
   - If using the GNS3 VM: `\\GNS3-VM\gns3\images\QEMU\` (SMB share)
   - Or SSH into the GNS3 VM and copy to `/opt/gns3/images/QEMU/`
2. In GNS3 GUI on Windows: `Edit` → `Preferences` → `QEMU VMs`
3. Click `New` → `New QEMU VM`
4. Name it `FortiGate-7.0.9-PreLicensed`
5. Select `fortios.qcow2` as the disk image
6. Set RAM to 2048 MB, vCPUs to 1
7. Set adapters to 8, adapter type `virtio-net-pci`
8. Console type: `telnet`
9. Finish — the template is ready to use

## Usage Ideas

- **Branch firewall** in multi-site topology
- **HA cluster test** (though 3-interface limit applies)
- **Extra isolated lab network**
- **Spare for experiments** without touching working FGTs
- **Mixed-version topology** with 7.4.12 (OSPF compatible)

## Verification

Boot the image and run:
```
get system status
get system license status
```

Look for `License Status: Valid` and the version string.
