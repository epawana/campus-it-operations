# Proxmox VE Infrastructure & Workload Configuration

## 1. Overview
This runbook details the setup of a bare-metal Proxmox Virtual Environment (PVE) host, network configuration, and workload provisioning. It covers the hypervisor initialization, physical disk passthrough for a Windows 11 virtual machine (with UEFI, TPM 2.0, and VirtIO optimizations), and infrastructure references for hosted container services.

---

## 2. Infrastructure Architecture

| Layer | Component | Description |
| :--- | :--- | :--- |
| **Bare-Metal** | Proxmox VE Host | Debian-based Type-1 Hypervisor |
| **Networking** | `vmbr0` | Bridged physical adapter (`eno1` / `eth0`) |
| **Storage** | Local ZFS / LVM-Thin | System images, container rootfs, and ISOs |
| **VM (ID: 100)** | Windows 11 Workstation | UEFI/OVMF, vTPM 2.0, Raw Storage Passthrough, VirtIO |
| **CT (ID: 200)** | MeshCentral Server | Unprivileged LXC (Reference: [MeshCentral Documentation](../services/meshcentral-lxc-deployment.md)) |

---

## 3. Host Network Configuration

The default network configuration uses a standard Linux Bridge (`vmbr0`) bound to the primary physical network interface.

File: `/etc/network/interfaces`

```bash
auto lo
iface lo inet loopback

iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.1.100/24
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
```

Verify networking status after applying changes:
```bash
systemctl restart networking
ip addr show vmbr0
```

---

## 4. Workload Provisioning: Windows 11 VM

### 4.1 Prerequisites & ISOs
Store the following images in `/var/lib/vz/template/iso`:
1. Official Windows 11 Installation ISO
2. Latest Fedora VirtIO Windows Driver ISO (`virtio-win.iso`)

### 4.2 VM Creation Parameters
Create the base VM structure adhering to Windows 11 hardware constraints:

- **BIOS**: OVMF (UEFI)
- **Machine Type**: `q35`
- **TPM**: Add vTPM 2.0 (`v2.0`, stored on local-lvm)
- **CPU**: Type = `host`, Cores = minimum 4
- **RAM**: Minimum 8192 MB (Ballooning enabled or pinned)
- **Display**: VirtIO GPU or default SPICE/QXL

### 4.3 Physical Disk Passthrough Configuration
To achieve bare-metal drive performance and allow the Windows guest direct physical disk access, route the disk using `/dev/disk/by-id/`.

1. **Locate the target disk unique ID:**
   ```bash
   ls -l /dev/disk/by-id/
   ```
   *Identify the persistent ID (e.g., `ata-Samsung_SSD_870_EVO_S6PENH0W...` or `nvme-...`). Do not use `/dev/sdX` as device paths can change on reboot.*

2. **Map the physical disk to the VM:**
   ```bash
   qm set 100 -scsi1 /dev/disk/by-id/ata-Samsung_SSD_870_EVO_XXXXX
   ```

3. **Enable SCSI Controller Optimization:**
   Ensure the controller type is set to **VirtIO SCSI Single** for proper TRIM/discard pass-through and maximum queue depth:
   ```bash
   qm set 100 --scsihw virtio-scsi-single
   ```

### 4.4 Driver Installation & Guest Agent
1. **Windows Setup Stage:**
   - Mount both the Windows 11 ISO and `virtio-win.iso`.
   - When the installer reports "No drives were found", click **Load Driver** and navigate to:
     - `virtio-win:\vioscsi\w11\amd64` (Storage controller)
     - `virtio-win:\NetKVM\w11\amd64` (VirtIO Network driver)
2. **Post-Installation:**
   - Run `virtio-win-guest-tools.exe` from the mounted driver ISO.
   - Confirm **QEMU Guest Agent** is running in Windows Services (`services.msc`).
   - Enable Guest Agent communication on Proxmox:
     ```bash
     qm set 100 --agent enabled=1
     ```

---

## 5. Hosted Services Integration

| Service ID | Type | Role | Documentation |
| :--- | :--- | :--- | :--- |
| **CT 200** | LXC (Debian/Ubuntu) | Centralized Device Management (MeshCentral) | [`meshcentral-lxc-deployment.md`](../services/meshcentral-lxc-deployment.md) |

*Note: Core service configurations are decoupled from hypervisor host storage to allow seamless portability and independent backup routines.*

---

## 6. Maintenance & Operational Commands

### VM Management via CLI
```bash
# Start Windows 11 VM
qm start 100

# Graceful shutdown via QEMU Guest Agent
qm shutdown 100

# Monitor live resource usage
qm status 100 --verbose
```

### Proxmox Host Health Checks
```bash
# Check physical disk SMART metrics
smartctl -H /dev/sdX

# Review hypervisor journal logs
journalctl -xeu pveproxy
```