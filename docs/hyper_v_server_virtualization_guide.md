# Windows Server Hyper-V Infrastructure & Workload Consolidation

## 1. Overview

This runbook documents the deployment, virtualization management, and workload isolation implemented on **Microsoft Hyper-V** (running on Windows Server). The platform was established to consolidate critical institutional internal services, provision virtual switches for LAN segregation, manage virtual storage controllers (VHDX), and ensure high-availability operational runtimes for local Windows/Linux services prior to bare-metal migration.

## 2. Infrastructure & Hypervisor Architecture

```
                      +-----------------------------------+
                      |     Physical Host (Bare-Metal)    |
                      |     Windows Server / Hyper-V      |
                      +-----------------------------------+
                                        |
                 +----------------------+----------------------+
                 |                                             |
     [ External Virtual Switch ]                    [ Internal / Private vSwitch ]
        (Bound to Physical NIC)                        (Host-Guest Isolated LAN)
                 |                                             |
       +---------+---------+                         +---------+---------+
       |                   |                         |                   |
+-------------+     +-------------+           +-------------+     +-------------+
|    VM 01    |     |    VM 02    |           |    VM 03    |     |    VM 04    |
| Internal DB |     | App / CBT   |           | Staging Lab |     | Core Auth   |
| (Gen 2 VHDX)|     | (Gen 2 VHDX)|           | (Gen 1/2)   |     | (Gen 2)     |
+-------------+     +-------------+           +-------------+     +-------------+

```

### 2.1 Host & Environment Specifications

| Component | Specification | Description | 
| ----- | ----- | ----- | 
| **Hypervisor Platform** | Windows Server Hyper-V Role | Enterprise Type-1 Microkernel Hypervisor | 
| **Management Tools** | Hyper-V Manager, PowerShell (`Hyper-V` Module) | Local and Remote Host Orchestration | 
| **Storage Architecture** | Fixed & Dynamic VHDX on RAID/SSD Pools | High-throughput virtual disk controllers | 
| **Networking** | Hyper-V Virtual Switch (`vSwitch`) | External (Bridged LAN), Internal, and Private isolation | 

## 3. Hypervisor Role Installation & Virtual Networking

### 3.1 Hyper-V Role Deployment via PowerShell

Initialize the Hyper-V role and administrative management tools using elevated PowerShell:

```powershell
# Install Hyper-V role and management tools
Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart
```

Verify virtualization hypervisor status:

```powershell
Get-WindowsFeature -Name Hyper-V*
```

### 3.2 Virtual Switch (`vSwitch`) Provisioning

Segregate administrative guest traffic and direct LAN-exposed services by configuring virtual network switches:

1. **External vSwitch (Direct Physical Uplink):**
   Allows guest VMs direct IP assignment from the institutional DHCP/gateway on the physical switch.

```powershell
# Create an External Virtual Switch bound to the physical adapter
Get-NetAdapter
New-VMSwitch -Name "vSwitch-External-LAN" `
             -NetAdapterName "Ethernet 1" `
             -AllowManagementOS $true
```

2. **Internal vSwitch (Isolated Inter-VM Network):**
   Used for backend services (e.g., database clustering or testing environments) unreachable from the physical LAN:

```powershell
New-VMSwitch -Name "vSwitch-Internal-Private" -SwitchType Internal
```

## 4. Virtual Machine Provisioning & Optimization

### 4.1 Generation 1 vs. Generation 2 Standards

Standardize modern workloads on **Generation 2** virtual machines to leverage native UEFI firmware, synthetic SCSI architecture, and hardware security enhancements:

* **Generation 2 Profiles:**
  * Firmware: UEFI with Secure Boot enabled
  * Boot Device: SCSI Virtual Hard Disk (`.vhdx`)
  * Emulated Devices: None (pure synthetic VMBus drivers)
  * Network: Synthetic Network Adapter (PXE over IPv4/IPv6 supported natively)

### 4.2 Automated VM Creation Script (PowerShell)

```powershell
# Define VM variables
$VMName      = "SRV-APP-PROD01"
$VMSwitch    = "vSwitch-External-LAN"
$VMPath      = "D:\Hyper-V\Virtual Machines"
$VHDPath     = "D:\Hyper-V\Virtual Hard Disks\$VMName.vhdx"
$ISOPath     = "D:\ISOs\WindowsServer2022.iso"

# 1. Create Generation 2 VM
New-VM -Name $VMName `
       -Generation 2 `
       -MemoryStartupBytes 4GB `
       -SwitchName $VMSwitch `
       -Path $VMPath `
       -NewVHDPath $VHDPath `
       -NewVHDSizeBytes 100GB

# 2. Configure Dynamic Memory & vCPU allocation
Set-VMMemory -VMName $VMName `
             -DynamicMemoryEnabled $true `
             -MinimumBytes 2GB `
             -MaximumBytes 8GB

Set-VMProcessor -VMName $VMName -Count 4

# 3. Attach Installation Media & Configure Boot Order
Add-VMDvdDrive -VMName $VMName -Path $ISOPath
$dvd = Get-VMDvdDrive -VMName $VMName
Set-VMFirmware -VMName $VMName -FirstBootDevice $dvd
```

## 5. Storage Tuning & Production Hardening

### 5.1 VHDX Optimization & Resizing

* **Dynamic Expansion vs. Fixed Size:**
  * Production database engines use **Fixed Size VHDX** to prevent fragmentation and zero-write latency overhead.
  * General application workloads utilize **Dynamically Expanding VHDX** with routine compaction maintenance.

```powershell
# Compact a dynamically expanding virtual disk (run offline or during maintenance)
Mount-VHD -Path "D:\Hyper-V\Virtual Hard Disks\SRV-APP-PROD01.vhdx" -ReadOnly
Optimize-VHD -Path "D:\Hyper-V\Virtual Hard Disks\SRV-APP-PROD01.vhdx" -Mode Full
Dismount-VHD -Path "D:\Hyper-V\Virtual Hard Disks\SRV-APP-PROD01.vhdx"
```

### 5.2 Guest Integration Services

Ensure all integration services are enabled for guest OS time synchronization, clean guest-initiated shutdowns, and heartbeat diagnostics:

```powershell
Enable-VMIntegrationService -VMName "SRV-APP-PROD01" -Name "Guest Service Interface"
Get-VMIntegrationService -VMName "SRV-APP-PROD01"
```

## 6. Operational Maintenance & Backup Strategy

### 6.1 Checkpoint (Snapshot) Discipline

```
[Operational Checkpoint Guidelines]
[!] Production Checkpoints: Use standard VSS (Volume Shadow Copy) checkpoints for application consistency.
[!] Standard Checkpoints: Restrict to non-production testing/staging only.
[!] Never use checkpoints as a replacement for cold or scheduled deduplicated backups.
[!] Delete/Merge checkpoints within 48-72 hours to prevent disk degradation and runaway host storage consumption.
```

### 6.2 Routine PowerShell Diagnostic Commands

```powershell
# Check running VMs and resource allocation
Get-VM | Select-Object Name, State, CPUUsage, MemoryAssigned, Uptime

# Check disk health and allocated paths
Get-VMHardDiskDrive -VMName * | Select-Object VMName, Path, ControllerType

# Gracefully power down virtual machine via Integration Services
Stop-VM -Name "SRV-APP-PROD01" -SaveState # Or use -Force for emergency shutdown
```

## 7. Troubleshooting Runbook

| Incident | Root Cause | Resolution Step | 
| ----- | ----- | ----- | 
| **VM fails to boot with PXE / network loop** | Boot order prioritized synthetic network over SCSI disk | Modify firmware order: `Set-VMFirmware -VMName <Name> -FirstBootDevice (Get-VMHardDiskDrive -VMName <Name>)` | 
| **Host NIC drops connection when vSwitch is created** | `AllowManagementOS` flag omitted during creation | Run `Set-VMSwitch -Name <SwitchName> -AllowManagementOS $true` or recreate vSwitch ensuring management traffic is preserved. | 
| **Checkpoint merge fails due to low disk space** | Host storage exhausted by differential AVHDX files | Evacuate ISO images/secondary disks to external storage; shut down VM to allow automatic background hypervisor merge. | 
