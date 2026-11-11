# NetSupport School Lab Management & Proctoring Deployment

## 1. Overview

This document details the architecture, rollout, configuration, and end-user enablement for **NetSupport School** across the institutional computer laboratory. The deployment establishes centralized lab orchestration, live screen monitoring, student device restriction, and exam proctoring workflows, complemented by a teacher enablement program.

---

## 2. System Architecture & Topology

```
                       [ Institutional LAN / VLAN 30 ]
                                      |
         +----------------------------+----------------------------+
         |                                                         |
+-------------------+                                     +-------------------+
|  Teacher Station  | (Tutor Console)                     |  Student Stations | (Student Client)
|  IP: 192.168.30.10|                                     |  IP Pool: .101-.140|
+-------------------+                                     +-------------------+
         |                                                         |
         +==== Multicast / TCP Port 5405-5420 / UDP Broadcast =====+
```

### 2.1 Environmental Matrix

| Parameter | Specification | Purpose |
| :--- | :--- | :--- |
| **Lab Network Segment** | `192.168.30.0/24` (Lab VLAN) | Segregates classroom broadcast traffic from admin networks |
| **Discovery Protocol** | IP Subnet Broadcast / Direct Connect | Reliable dynamic client discovery across the lab |
| **Communication Ports** | TCP/UDP `5405`, `5421` | Command channel, screen streaming, and file transfers |
| **Host Workstations** | Windows 10/11 Pro (64-bit) | Standardized client and tutor operating environment |
| **Security Mechanism** | Encrypted Security Key (`NS_KEY`) | Restricts client takeover to authorized teacher consoles |

---

## 3. Tutor (Server) & Student (Client) Installation

### 3.1 Silent / Unattended Installation (Student Clients)

To deploy across multiple client stations consistently, use an unattended installer script executed via elevated command shell or local imaging tools.

1. Prepare an answers configuration template (`client_install.iss`) or execute using standard silent MSI parameters:

```cmd
msiexec /i "\\DEPLOY-SRV\Share\NetSupport\NetSupportSchool.msi" /qn ^
  INSTALLDIR="C:\Program Files (x86)\NetSupport\NetSupport School" ^
  LICENCETYPE="School" ^
  CLIENT=1 ^
  TUTOR=0 ^
  SECURITYKEY="[REDACTED_SECURITY_KEY]" ^
  CLASSROOM="COMPUTER_LAB_A"
```

2. Verify client service registration:

```cmd
sc query "NetSupport Client"
```

### 3.2 Tutor (Teacher Console) Installation

1. Execute `setup.exe` and select **Custom Installation**.
2. Select **Tutor Console** only (disable Client component on the instructor workstation to prevent recursive loop captures).
3. Import the institutional licensing file:
   * **License Serial Number**: `NSS-******-******` *(Masked)*
   * **Licensed Workstations**: `40 Devices`
   * **Organization Name**: `[REDACTED_INSTITUTION_NAME]`
4. Input the identical **Security Key** assigned to the student installations:
   * **Security Key**: `********` *(Masked)*

---

## 4. Operational Application Configuration

### 4.1 Room & Connectivity Configuration

To prevent client stations from picking up random consoles across surrounding rooms:

1. Open **NetSupport School Configuration Utility** (`C:\Program Files (x86)\NetSupport\NetSupport School\config.exe`).
2. Navigate to **Room Setup**:
   * Mode: `This computer is located in a specific room`
   * Primary Room ID: `LAB_A`
3. Network Settings:
   * Transport: `TCP/IP`
   * Broadcast Address: `192.168.30.255`
   * Enable **Direct Connect / IP Address List** fallback for subnets with broadcast suppression enabled.

### 4.2 Security & Integrity Safeguards

To prevent student tampering or unauthorized task termination:

```
[Security Policy Checklist]
[x] Protect Configuration Utility with Master Password: [REDACTED_ADMIN_PASSWORD]
[x] Hide System Tray Icon on Client Workstations (Silent Mode)
[x] Enable "Prevent Student from Terminating NetSupport Service via Task Manager"
[x] Enforce User Confirmation on First-time Tutor Connection: Disabled (Mandatory for Proctoring)
```

---

## 5. Proctoring & Classroom Control Features

```
+-------------------------------------------------------------------------+
|                  NETSUPPORT TUTOR CLASSROOM WORKSPACE                   |
+-------------------------------------------------------------------------+
| [ Monitor All ]  [ Blank Screen ]  [ Lock All ]  [ Web Control ] [ File ]|
+-------------------------------------------------------------------------+
| +-------------+  +-------------+  +-------------+  +-------------+       |
| |  [PC-01]    |  |  [PC-02]    |  |  [PC-03]    |  |  [PC-04]    |  ...  |
| | [Thumbnail] |  | [Thumbnail] |  | [Thumbnail] |  | [Thumbnail] |       |
| | User: Std01 |  | User: Std02 |  | User: Std03 |  | User: Std04 |       |
| +-------------+  +-------------+  +-------------+  +-------------+       |
+-------------------------------------------------------------------------+
```

### 5.1 Real-Time Monitoring & Screen Control
* **Mosaic Thumbnail View**: Displays real-time 30-fps thumbnails of all connected student monitors.
* **Dual-Monitor Tracking**: Automatically detects and switches between primary and secondary display outputs on client stations.
* **1-to-1 Remote Assistance**: Right-click target student thumbnail -> **Watch / Share / Control** to take mouse/keyboard control with user notification suppressed or flagged.

### 5.2 Exam Lockdown & Restriction Profiles

1. **Restricted Web Browsing (Whitelisting):**
   * Path: **Web Control** -> **Approved Sites Only**.
   * Policy: Block all outbound traffic except designated exam URLs:
     * `https://cbt.[domain_masked].sch.id/*`
     * `https://exam-engine.local/*`
2. **Application Restriction:**
   * Path: **Application Control** -> **Restricted List**.
   * Active Deny List: `calc.exe`, `notepad.exe`, `chrome.exe` (if local test app used), `anydesk.exe`, `teamviewer.exe`.
3. **USB & Peripherals Blocker:**
   * Disable external USB flash drives across all stations in a single command prior to distributing exam material.

---

## 6. Teacher Training & Onboarding Module

A comprehensive 3-stage training syllabus designed for instructional staff to ensure frictionless classroom adoption.

### Stage 1: The Basics (Classroom Session Lifecycle)
* **Starting the Session**: Launch the **Tutor Console** icon from the desktop; verify all student nodes change from gray (offline) to green (connected).
* **Attention Management**:
  * Using **Show Teacher** (transmits the instructor's screen directly to every student monitor, disabling local student input).
  * Using **Blank Screen** / **Lock** to pause all student keyboards and mice during lecture transitions.

### Stage 2: Assessment & Distribution Workflows
* **Quick Question (Instant Polling)**: Push instant true/false or multiple-choice questions to all stations to gauge subject retention without grading software.
* **File Distribution & Collection**:
  * Navigate to **File Transfer** -> **Distribute Work**.
  * Select target directory `C:\Users\Student\Documents\Assignment1`.
  * Click **Collect Work** at session conclusion; files are automatically aggregated and renamed by student seat/username into the teacher's local folder.

### Stage 3: CBT Exam Proctoring Protocol
* **Step 1: Lockdown**: Click **Restrict Web** (Approved list only) and **Lock Application Launching**.
* **Step 2: Monitoring**: Switch to **Full Screen Thumbnail Mode** (F7).
* **Step 3: Intervention**: Use **Send Message** or private chat to warn offending students without causing classroom disruptions.

---

## 7. Troubleshooting & Maintenance Runbook

| Incident | Probable Root Cause | Resolution Step |
| :--- | :--- | :--- |
| **Student nodes do not appear in Tutor Console** | Network subnet mismatch or local Windows Firewall block | Confirm client IP is in `192.168.30.0/24`. Run: `netsh advfirewall firewall add rule name="NSS" dir=in action=allow program="C:\Program Files (x86)\NetSupport\NetSupport School\client32.exe" enable=yes` |
| **"Security Key Mismatch" Error** | Client installed with an outdated or typo-laden key | Re-run silent installer script with updated `SECURITYKEY="[REDACTED]"` parameter or modify via `config.exe`. |
| **High latency / Choppy screen monitoring** | Excessive multicast traffic or switch broadcast storm | Lower thumbnail refresh rate from `Continuous` to `5 seconds` in Tutor Console settings. |
| **Student bypasses via Safe Mode** | Machine boot sequence allows local OS bypass | Lock workstation BIOS with Supervisor Password; enforce UEFI Secure Boot; set internal SSD as non-changeable first boot device. |
