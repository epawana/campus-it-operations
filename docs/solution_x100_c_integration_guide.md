# Solution X100-C Biometric Attendance Integration & Troubleshooting

## 1. Overview

This technical runbook details the network integration, communication parameter configuration, server synchronization, and hardware troubleshooting for the **Solution X100-C** standalone biometric fingerprint time & attendance terminal. The system interfaces with centralized HR/academic attendance databases over TCP/IP LAN infrastructure to automate log collection and personnel record synchronization.

## 2. Hardware Architecture & Network Placement

```
                  [ Institutional Core Network / Admin VLAN 10 ]
                                         |
                       +-----------------+-----------------+
                       |                                   |
              +-----------------+                 +-----------------+
              | Attendance SRV  |                 | Solution X100-C |
              | IP: 192.168.10.5|                 | IP: 192.168.10.220
              | App: Att2008 /  |                 | Port: 4370 TCP  |
              | AttendancePlus  |                 | COM Key: ****** |
              +-----------------+                 +-----------------+
                       |                                   |
                       +====== Biometric Sync (TCP/IP) ====+= (CAT6 Drop)
```

### 2.1 Technical Specifications

| Parameter | Specification | Notes |
| :--- | :--- | :--- |
| **Terminal Model** | Solution X100-C | Standalone Optical Sensor (ZMM platform) |
| **Interface Options** | TCP/IP (Ethernet RJ45), USB Host/Client | RJ45 10/100 Mbps primary |
| **Default Port** | `4370` (TCP/UDP) | Standard protocol communication port |
| **Authentication Security** | 6-digit Communication Key (`CommKey`) | Encrypts and authorizes socket handshake |
| **Capacity** | ~10,000 Fingerprints / 100,000 Log Events | Offline non-volatile flash storage |

---

## 3. Network Configuration & Device Provisioning

### 3.1 On-Device Menu Network Setup

Configure static addressing on the physical terminal interface to avoid lease expiration issues:

1. Press **[Menu/OK]** -> Authenticate as Super Admin.
2. Navigate to **Comm. (Komunikasi)** -> **Network (Jaringan)**.
3. Configure static addressing:
   - **IP Address**: `192.168.10.220` (Reserved outside dynamic DHCP pool)
   - **Subnet Mask**: `255.255.255.0`
   - **Gateway**: `192.168.10.1`
   - **DNS**: `1.1.1.1` (Optional)
4. Navigate to **Comm. Options (Pengaturan Komunikasi)**:
   - **Device ID**: `1` (Unique integer across the fleet)
   - **Comm Key / COM Key**: `******` *(Masked; must match application sync profile)*
   - **Port**: `4370`

---

## 4. Software Integration & Server Configuration

### 4.1 Attendance Management Client Setup (Att2008 / Attendance Management)

1. Launch the Attendance Management software on the central workstation/server.
2. Go to **Device Management** -> **Add New Device**:
   - **Device Name**: `ATT-MAIN-LOBBY`
   - **Communication Type**: `Ethernet (TCP/IP)`
   - **IP Address**: `192.168.10.220`
   - **Port**: `4370`
   - **Device ID**: `1`
   - **Communication Password / CommKey**: `[REDACTED_COMM_KEY]`
3. Click **Connect**. Verify status indicator transitions from disconnected to connected.

### 4.2 Automated Log Retrieval SOP

To maintain database freshness and clear terminal flash memory before capacity thresholds:

```
[Attendance Daily Automation Routine]
1. Connect to terminal daily at scheduled maintenance window (e.g., 23:00).
2. Download General Log Data (Presensi / Event Log).
3. Append records into central MS Access / MySQL / SQL Server database.
4. Synchronize device clock (NTP alignment with host system time).
5. [Monthly Maintenance]: Clear obsolete attendance logs from device flash after verified backup.
```

---

## 5. Network & Firewall Hardening

To safeguard the terminal against unauthorized polling or network-level disruptions:

1. **Firewall Inbound/Outbound Rules (Server Host):**
   ```powershell
   # Allow outbound/inbound attendance polling on TCP 4370
   New-NetFirewallRule -DisplayName "Attendance Terminal 4370" `
                       -Direction Inbound `
                       -LocalPort 4370 `
                       -Protocol TCP `
                       -Action Allow
   ```
2. **VLAN Segregation:** Isolate attendance units inside the management/administrative VLAN; drop inter-VLAN routing originating from guest or student subnets.

---

## 6. Comprehensive Troubleshooting Matrix

| Incident Symptom | Root Cause | Diagnosis & Resolution Steps |
| :--- | :--- | :--- |
| **"Can't connect to device" via software** | IP conflict or mismatched Comm Key | 1. Ping `192.168.10.220`. If no response, verify RJ45 link lights and switch port VLAN.<br>2. Check if another host hijacked the IP: `arp -a`.<br>3. Verify the **COM Key** on the device matches the software password. |
| **Ping replies, but software connection fails (Socket Error)** | Port `4370` blocked or hung device daemon | 1. Test socket open: `Test-NetConnection 192.168.10.220 -Port 4370`.<br>2. If failed, power cycle the X100-C (unplug power barrel jack for 15s) to flush frozen TCP listeners. |
| **Terminal clock drifts / logs show wrong timestamps** | Internal RTC drift; lack of NTP sync | 1. Connect via software -> select device -> click **Synchronize Time**.<br>2. Replace internal CR2032 button battery on the mainboard if time resets after power loss. |
| **Fingerprint reading delay / High rejection rate** | Dirty optical prism or dry skin conditions | 1. Clean sensor glass using lint-free microfiber dampened with ethanol.<br>2. Lower recognition threshold (1:N Match) from strict to standard in device firmware settings. |
| **Terminal locked: Admin lost access** | Administrator locked out or staff turnover | 1. Connect terminal to software via USB or verified Ethernet IP.<br>2. Execute **Clear Administrator Privilege** command from the software interface.<br>3. Alternative: Generate dynamic manufacturer backdoor unlock code based on machine serial number and system date. |
| **USB Flash Drive export fails** | Unsupported filesystem or oversized drive | Format USB flash drive strictly to **FAT32** with **MBR partition table** (NTFS, exFAT, and GPT partitions are not recognized by the terminal firmware). Ensure capacity is $\le 32\text{ GB}$. |

---

## 7. Operational Backup & Recovery SOP

### 7.1 Offline Emergency Log Extraction (USB Method)

If the Ethernet interface fails during network outages:

1. Insert a FAT32-formatted USB drive into the USB-A host port on the X100-C.
2. Press **[Menu/OK]** -> **PenDrive (U-Disk Management)**.
3. Select **Download AttLog (Unduh Data Presensi)**.
4. Extract `*.dat` file on administrative PC and import manually into the attendance software via **Data -> Import from USB Disk**.

### 7.2 Biometric Template Backup

Export registered fingerprint templates once every semester or after mass staff enrollments:
```
Software -> Device Management -> Download User Information & Fingerprint Templates
-> Save Backup to Encrypted Storage (\backup\attendance\templates_YYYYMM.dat)
```
