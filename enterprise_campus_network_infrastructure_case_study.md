# Enterprise Dual-Tier Campus Network Infrastructure & Edge Hardening

[![MikroTik RouterOS](https://img.shields.io/badge/MikroTik-RouterOS_v6-orange.svg)](https://mikrotik.com)
[![Network Architecture](https://img.shields.io/badge/Architecture-Two--Tier_Hierarchical-blue.svg)]()
[![Security](https://img.shields.io/badge/Hardening-Firewall_%26_Anti--Backdoor-green.svg)]()
[![Status](https://img.shields.io/badge/Deployment-Production_Ready-brightgreen.svg)]()

A comprehensive enterprise network infrastructure deployment featuring a multi-router topology (Edge Gateway & Core Distribution), automated multi-WAN recursive failover, granular bandwidth shaping, campus-wide captive portal, and post-incident security hardening.

---

## Architecture Overview

```
                      +-----------------------------+     +-----------------------------+
                      | Primary ISP (Fiber/Static)  |     | Secondary ISP (DHCP Uplink) |
                      | Static CIDR: 203.0.113.0/29 |     | Gateway: 192.168.1.1        |
                      +-----------------------------+     +-----------------------------+
                                     │                                   │
                              (ether2)▼                           (ether1)▼
                      +─────────────────────────────────────────────────────────────────+
                      |            EDGE GATEWAY ROUTER (MikroTik RB450Gx4)              |
                      |            Hostname: Edge-Gateway | ROS: 6.49.x                 |
                      |  - Dual-WAN Recursive Failover Engine (Google DNS Ping Target)  |
                      |  - Edge Hardening: Stateful Firewall, RAW Filter, Management ACL|
                      |  - Remote Engineering Ingress: OpenVPN (AES-256) & L2TP/IPsec   |
                      +─────────────────────────────────────────────────────────────────+
                                                      │
                                                      │ Transit Trunk (192.168.120.0/23)
                                                      │ (bridge1: ether3-ether5)
                                                      ▼
                      +─────────────────────────────────────────────────────────────────+
                      |         CORE DISTRIBUTION & SERVICES (MikroTik RB450Gx4)        |
                      |         Hostname: Core-Services | ROS: 6.45.x                   |
                      |  - Captive Portal / Hotspot Engine (Token/MAC Authentication)   |
                      |  - Transparent Enforced Content Filtering (OpenDNS DST-NAT)    |
                      |  - Hierarchical Chained DHCP Pools & Kid-Control Time Policies  |
                      |  - Perimeter Layer 7 Application & Bandwidth Shaping Control    |
                      +─────────────────────────────────────────────────────────────────+
                                      │                                 │
                   (eth2-Admin)       ▼                                 ▼ (eth3-Campus)
              +──────────────────────────────+            +──────────────────────────────+
              |    ADMIN / DATA CENTER ZONE  |            |   CAMPUS, RESIDENTIAL & LABS |
              |    - Subnet: 172.16.0.0/24   |            |   - Zone 1: 172.16.2.0/23    |
              |    - Hyper-V VMs & Services  |            |   - Zone 2: 172.16.4.0/23    |
              |    - Biometric Face/FP Term. |            |   - Expansion: 172.16.6.0/23 |
              +──────────────────────────────+            +──────────────────────────────+
```

---

## Key Highlights & Implemented Solutions

1. **Multi-WAN Recursive Routing & Zero-Downtime Failover**
   - Implemented scope-targeted recursive routing on the Edge Router using remote DNS IP probes (`8.8.8.8`) rather than physical interface link status, mitigating upstream black-hole states.
   - Primary ISP traffic defaults to dedicated optical bandwidth with instant dynamic transition to secondary ISP links upon packet loss thresholds.

2. **Defense-in-Depth Edge Hardening & Incident Remediation**
   - **Remediation:** Neutralized an active cron-based backdoor vulnerability (`scheduler injection`) targeting credentials export, purged compromised artifacts, and restricted privilege sets.
   - **Open DNS Resolver Mitigation:** Implemented bidirectional RAW and Input filter policies dropping unauthorized inbound DNS probes (`UDP/TCP 53`) across all WAN interfaces.
   - **Service Boundary Isolation:** Locked critical administration daemons (WinBox, SSH) to trusted internal management subnets (`192.168.120.0/23`) and authorized VPN endpoints (`192.168.99.0/24`). Disabled unencrypted vectors (Telnet, FTP, HTTP, API).
   - **MAC Layer Protection:** Bound `mac-server` and `mac-winbox` exclusively to an internal management interface list (`LOCAL-MGMT`) to eliminate WAN brute-force risk while preventing physical console lockout.

3. **Transparent DNS Hijacking & Threat Filtering**
   - Applied destination NAT rewrite rules capturing all local DNS traffic (`UDP 53`) across campus segments, forcing redirection through **OpenDNS FamilyShield** resolving clusters (`208.67.222.123` / `208.67.220.123`).
   - Hardened outbound perimeter with Layer-7 pattern recognition and TLS SNI matchers targeting entertainment, social media, and CDN platforms during business and learning hours.

4. **Multi-Subnet Chained IPAM & DHCP Orchestration**
   - Architected contiguous sequential address pools with automated multi-tier fallback (`poolLT1 -> poolLT3 -> poolLT4`), scaling support for over 1,500 active campus endpoints.
   - Managed complete static IP assignments for surveillance NVRs, biometric terminals, Hyper-V virtualized enterprise servers, and UniFi Enterprise wireless access points.

---

## IP Address Management (IPAM) Architecture

### Subnet Allocation Matrix

| Segment Name | Interface / Scope | Network CIDR | Gateway IP | DHCP Pool Range | Assignment Type |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **WAN - Primary ISP** | `ether2-ASNet` | `203.0.113.0/29` | `203.0.113.1` | N/A | Static Dedicated Public |
| **WAN - Secondary ISP** | `ether1-CORPNet`| `192.168.1.0/24` | `192.168.1.1` | Dynamic | Dynamic Upstream DHCP |
| **Inter-Router Transit**| `bridge1` | `192.168.120.0/23` | `192.168.120.1` | `.120.2 - .121.254` | Chained Pool (Primary) |
| **Transit Expansion 1** | `bridge1` | `192.168.122.0/23` | `192.168.122.1` | `.122.2 - .123.254` | Chained Pool (Next 1) |
| **Transit Expansion 2** | `bridge1` | `192.168.124.0/23` | `192.168.124.1` | `.124.2 - .125.254` | Chained Pool (Next 2) |
| **Remote Access VPN** | PPP Dynamic | `192.168.99.0/24` | `192.168.99.1` | `.99.10 - .99.30` | L2TP/IPsec & OpenVPN |
| **Headquarters & Admin**| `eth2-YYSN` | `172.16.0.0/24` | `172.16.0.1` | `.0.20 - .0.254` | Static Lease & DHCP |
| **Academic & Dorms L1** | `eth3-Ikhwan` | `172.16.2.0/23` | `172.16.2.1` | `.2.10 - .3.254` | Hotspot + Captive Portal |
| **Academic & Dorms L2** | `eth3-Ikhwan` | `172.16.4.0/23` | `172.16.4.1` | `.4.10 - .5.254` | Chained Hotspot Pool |
| **Campus Expansion** | `eth3-Ikhwan` | `172.16.6.0/23` | `172.16.6.1` | `.6.10 - .7.254` | Standby Chained Pool |

---

## Critical Infrastructure Static Mapping (Exemplar)

| Static IP | Hardware MAC (Masked) | Device Category | Functional Description / Mapping |
| :--- | :--- | :--- | :--- |
| `172.16.0.1` | `XX:XX:XX:AA:00:01` | Core Router | Default Gateway Core RB450Gx4 |
| `172.16.0.52` | `00:15:5D:XX:XX:02` | Virtual Server | Enterprise Portal VM (`portal.corp.internal`) |
| `172.16.0.76` | `04:95:E6:XX:XX:B0` | Enterprise AP | UniFi High-Density AP - Admin Block Floor 1 |
| `172.16.0.125`| `A0:B3:CC:XX:XX:AA` | Bare-Metal Server | Core Database & ERP Application Node |
| `172.16.0.170`| `00:15:5D:XX:XX:03` | Virtual Server | Campus LMS Learning Management Cluster |
| `172.16.0.235`| `00:15:5D:XX:XX:05` | Virtual Server | Digital Library Repository (`library.corp.internal`) |
| `172.16.0.45` | `00:17:61:XX:XX:9C` | Biometric Terminal| Primary Access Gate Biometric Fingerprint (Bypassed) |
| `172.16.0.78` | `D4:43:0E:XX:XX:F9` | Surveillance | Main Cafeteria High-Res IP Dome Camera |
| `172.16.0.139`| `00:17:61:XX:XX:B7` | Biometric Terminal| Optical AI Facial Recognition Terminal 01 |
| `172.16.2.13` | `78:45:58:XX:XX:D3` | Enterprise AP | UniFi AP - Student Center & Academic Lobby |
| `172.16.2.35` | `50:57:9C:XX:XX:00` | Network Peripheral| Department Network Multi-Function Printer (MFP) |
| `172.16.2.221`| `E4:24:6C:XX:XX:EB` | Surveillance | Academic Corridor IP Camera Node (Bypassed) |
| `172.16.3.252`| `BC:5E:33:XX:XX:F0` | Surveillance | Hikvision 32-Channel Central NVR Unit |

---

## Engineering Implementation Details

### 1. Dual-WAN Recursive Failover Configuration (Edge)
```routeros
# Primary route targets virtual IP 8.8.8.8 through physical ISP-1 Gateway
/ip route
add check-gateway=ping distance=1 dst-address=0.0.0.0/0 gateway=203.0.113.1
add dst-address=8.8.8.8/32 gateway=192.168.1.1 scope=10
add check-gateway=ping distance=2 dst-address=0.0.0.0/0 gateway=8.8.8.8 scope=11
```

### 2. Edge Firewall Hardening & Open Resolver Drop Policy
```routeros
/ip firewall filter
# Drop non-internal DNS amplification attempts on both uplinks
add action=drop chain=input comment="Drop External DNS WAN-Primary (UDP)" dst-port=53 in-interface=ether2-ASNet protocol=udp
add action=drop chain=input comment="Drop External DNS WAN-Primary (TCP)" dst-port=53 in-interface=ether2-ASNet protocol=tcp
add action=drop chain=input comment="Drop External DNS WAN-Backup (UDP)" dst-port=53 in-interface=ether1-CORPNet protocol=udp
add action=drop chain=input comment="Drop External DNS WAN-Backup (TCP)" dst-port=53 in-interface=ether1-CORPNet protocol=tcp

# State-aware boundary filtering
add action=accept chain=input connection-state=established,related,untracked comment="Allow State-Aware Input"
add action=drop chain=input connection-state=invalid comment="Drop Invalid Input States"
add action=accept chain=input protocol=icmp comment="Allow Diagnostic ICMP"
add action=accept chain=input dst-port=1194 protocol=tcp comment="Allow Ingress OpenVPN"
add action=accept chain=input dst-port=500,1701,4500 protocol=udp comment="Allow Ingress L2TP/IPsec"
add action=accept chain=input in-interface=bridge1 comment="Allow Internal Subnet Management"
add action=drop chain=input comment="Drop All Other Ingress"

# Forward Chain Protection
add action=accept chain=forward connection-state=established,related,untracked comment="Allow Active Flows"
add action=drop chain=forward connection-state=invalid comment="Drop Invalid Forwarding"
```

### 3. Enforced Transparent DNS Hijacking (Core)
```routeros
/ip firewall nat
add action=dst-nat chain=dstnat comment="Redirect UDP 53 to FamilyShield Primary" \
    dst-port=53 protocol=udp src-address-list=lokal to-addresses=208.67.222.123 to-ports=53
add action=dst-nat chain=dstnat comment="Redirect UDP 53 to FamilyShield Secondary" \
    dst-port=53 protocol=udp src-address-list=lokal to-addresses=208.67.220.123 to-ports=53
```

---

## Operational Verification & Disaster Recovery Drills

- [x] **Redundant Uplink Black-Hole Drill:** Simulated optical signal failure on Primary Interface `ether2-ASNet`. Verified that routing tables automatically transitioned global egress traffic to the Secondary ISP within 12 seconds with uninterrupted core routing.
- [x] **Penetration Test (External Port Ingress):** Executed full external `nmap -sS -sU -PN` scan across public interface `203.0.113.154`. Confirmed ports `53 (DNS)`, `80 (HTTP)`, `8291 (WinBox)`, and `22 (SSH)` returned `filtered/closed` from untrusted networks.
- [x] **Captive Portal Bypass Validation:** Verified biometric systems, surveillance cameras, and internal servers maintained persistent connection through IP bindings without captive portal redirects.
- [x] **Remote Telemetry Access:** Validated client tunnel authentication across OpenVPN using SHA1/AES-256 ciphers with successful routing to downstream assets.

---

## Author & Project Credits

* **Network Architect / Lead Engineer:** Senior IT Infrastructure & Systems Specialist  
* **Deployment Context:** Enterprise Educational Campus & Residential Foundation  
* **Tech Stack:** MikroTik RouterOS, L2/L3 Switching, OpenVPN, Ubiquiti UniFi Ecosystem, Layer-7 Filtering