# AWS Zero-Trust VPC Architecture: Bastion Jump Host & Split-Tunnel WireGuard VPN

## 1. Project Overview
This lab demonstrates the design and implementation of a segmented **Virtual Private Cloud (VPC)** architecture in Amazon Web Services (`ap-southeast-2` / Sydney). The primary objective is to enforce **Zero-Trust network isolation** for internal backend workloads—ensuring private instances have zero direct exposure to the public internet while maintaining secure administrative access via **SSH Agent Forwarding** and an encrypted **Split-Tunnel WireGuard VPN**.

### Key Architectural Goals
* **Network Segmentation:** Separate public-facing gateway resources from isolated internal servers using custom CIDR blocks (`10.10.0.0/16`).
* **Zero Public Exposure:** Deploy internal application servers without public IPv4 addresses or outbound NAT Gateways, eliminating unnecessary attack surfaces and hourly NAT costs.
* **Secure Administrative Access:** Implement two progressive access methods:
  1. **Bastion Jump Host (`ssh -A`)** using strict Security Group chaining.
  2. **Split-Tunnel WireGuard VPN (`UDP 51820`)** routing only internal VPC (`10.10.0.0/16`) and VPN (`10.200.0.0/24`) traffic through the encrypted tunnel.
* **FinOps & Cost Governance:** Enforce a `$0.01` Zero-Spend AWS Budget alert and practice full lifecycle teardown (`Terminate` EC2 instances and verify zero orphaned EBS volumes or Elastic IPs) to keep total lab expenditure at **$0.01**.

---

## 2. Network Topology & IP Addressing Plan

```text
[Admin Workstation (Windows)]
   │  ├── WireGuard Client (10.200.0.2/32)
   │  └── Split-Tunnel AllowedIPs: 10.10.0.0/16, 10.200.0.0/24
   │
   ▼  (Encrypted UDP 51820 / SSH Agent Forwarding)
┌──────────────────────────────────────────────────────────────────────┐
│ AWS Cloud (Region: ap-southeast-2 / Sydney)                          │
│ VPC: Lab-ZeroTrust-vpc (10.10.0.0/16)                                │
│                                                                      │
│  ┌────────────────────────────────┐                                  │
│  │ Public Subnet (10.10.1.0/24)   │ ◄──► [Internet Gateway (IGW)]    │
│  │ Route Table: rtb-public        │                                  │
│  │                                │                                  │
│  │  ┌──────────────────────────┐  │                                  │
│  │  │ VPN-Bastion-Gateway      │  │                                  │
│  │  │ • Instance: t3.micro     │  │                                  │
│  │  │ • Private IP: 10.10.1.80 │  │                                  │
│  │  │ • WireGuard: 10.200.0.1  │  │                                  │
│  │  │ • SG: SG-Bastion-Public  │  │                                  │
│  │  └────────────┬─────────────┘  │                                  │
│  └───────────────┼────────────────┘                                  │
│                  │ Internal VPC Routing (Local)                      │
│                  ▼                                                   │
│  ┌────────────────────────────────┐                                  │
│  │ Private Subnet (10.10.2.0/24)  │  (No IGW / No NAT Gateway)       │
│  │ Route Table: rtb-private1      │                                  │
│  │                                │                                  │
│  │  ┌──────────────────────────┐  │                                  │
│  │  │ Internal-App-Server      │  │                                  │
│  │  │ • Instance: t3.micro     │  │                                  │
│  │  │ • Private IP: 10.10.2.185│  │                                  │
│  │  │ • Public IP: None (-)    │  │                                  │
│  │  │ • SG: SG-Internal-Private│  │                                  │
│  │  └──────────────────────────┘  │                                  │
│  └────────────────────────────────┘                                  │
└──────────────────────────────────────────────────────────────────────┘
```

### Resource & IP Allocation Matrix

| Component | Name / Identifier | Subnet / Zone | Private IP | Public IP | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VPC** | `Lab-ZeroTrust-vpc` | `ap-southeast-2` | `10.10.0.0/16` | — | Isolated custom virtual network |
| **Public Subnet** | `Lab-ZeroTrust-subnet-public1` | `ap-southeast-2a` | `10.10.1.0/24` | Auto-assign | Hosts VPN & Bastion gateway |
| **Private Subnet** | `Lab-ZeroTrust-subnet-private1` | `ap-southeast-2a` | `10.10.2.0/24` | Disabled | Hosts isolated backend server |
| **Gateway VM** | `VPN-Bastion-Gateway` (`t3.micro`) | Public Subnet | `10.10.1.80` | Dynamic | Jump Host & WireGuard Server |
| **Backend VM** | `Internal-App-Server` (`t3.micro`) | Private Subnet | `10.10.2.185` | **None (`-`)** | Isolated internal Ubuntu server |
| **VPN Tunnel** | `wg0` (`AWS-ZeroTrust-Lab`) | Overlay Network | `10.200.0.1` / `.2` | — | Encrypted split-tunnel overlay |

---

## 3. Security Group Hardening (Stateful Firewall)

Instead of allowing broad CIDR ranges on internal servers, **Security Group Chaining** was implemented so that the private instance only accepts traffic originating from the Bastion/VPN gateway's Security Group ID.

| Security Group | Attached To | Direction | Protocol / Port | Source / Destination | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`SG-Bastion-Public`** | `VPN-Bastion-Gateway` | Inbound | TCP `22` (SSH) | Admin Public IP (`/32`) | Restricted SSH bootstrap access |
| **`SG-Bastion-Public`** | `VPN-Bastion-Gateway` | Inbound | UDP `51820` | `0.0.0.0/0` | WireGuard VPN encrypted tunnel |
| **`SG-Internal-Private`** | `Internal-App-Server` | Inbound | TCP `22` (SSH) | **`SG-Bastion-Public`** | Allows SSH only via Gateway |
| **`SG-Internal-Private`** | `Internal-App-Server` | Inbound | All ICMP - IPv4 | **`SG-Bastion-Public`** | Internal connectivity diagnostics |

---

## 4. Implementation Walkthrough

### Phase 1: Custom VPC & Subnet Isolation
1. Created **`Lab-ZeroTrust-vpc`** (`10.10.0.0/16`) in `ap-southeast-2` with 1 Availability Zone (`ap-southeast-2a`).
2. Configured two distinct route tables:
   * **`Lab-ZeroTrust-rtb-public`**: Routes `0.0.0.0/0` to **`Lab-ZeroTrust-igw`** (Internet Gateway).
   * **`Lab-ZeroTrust-rtb-private1`**: Contains only the local VPC route (`10.10.0.0/16 -> local`), keeping the subnet completely air-gapped from direct internet ingress/egress.
3. Enabled *Auto-assign public IPv4 address* exclusively on the public subnet.

![AWS VPC Resource Map](images/01-aws-vpc-resource-map.png)

---

### Phase 2: EC2 Provisioning & Subnet Isolation Verification
1. Launched **`VPN-Bastion-Gateway`** (Ubuntu Server 24.04 LTS, `t3.micro`) inside `10.10.1.0/24`, receiving private IP `10.10.1.80` and an ephemeral public IPv4 address.
2. Launched **`Internal-App-Server`** (Ubuntu Server 24.04 LTS, `t3.micro`) inside `10.10.2.0/24` with public IP assignment disabled, receiving only private IP `10.10.2.185`.

![EC2 Instances Isolation Proof](images/02-ec2-instances-isolation.png)

---

### Phase 3: SSH Agent Forwarding (Bastion Jump Host)
To avoid storing private `.pem` keys on the Bastion server, **SSH Agent Forwarding (`-A`)** was configured on the Windows administrative workstation:

```powershell
# 1. Enable and start the OpenSSH Authentication Agent service on Windows
Set-Service -Name ssh-agent -StartupType Manual
Start-Service ssh-agent

# 2. Restrict .pem key NTFS permissions to the current user only
icacls .\aws-lab-key.pem /inheritance:r
icacls .\aws-lab-key.pem /grant:r "$($env:USERNAME):(R)"

# 3. Load key into ssh-agent and connect to Bastion with Agent Forwarding
ssh-add .\aws-lab-key.pem
ssh -A ubuntu@<BASTION_PUBLIC_IP>

# 4. Hop from Bastion (10.10.1.80) into the isolated Private Subnet (10.10.2.185)
ssh ubuntu@10.10.2.185
```

![SSH Jump Host Proof](images/04-ssh-jump-host-proof.png)

---

### Phase 4: Split-Tunnel WireGuard VPN Deployment
To eliminate the need for two-step SSH hopping and allow direct private-IP administration, WireGuard (`wg0`) was deployed on `VPN-Bastion-Gateway`:

```bash
# Deploy WireGuard server on VPN-Bastion-Gateway (10.10.1.80)
curl -O https://raw.githubusercontent.com/angristan/wireguard-install/master/wireguard-install.sh
chmod +x wireguard-install.sh
sudo ./wireguard-install.sh
```

#### Client Split-Tunnel Configuration (`AWS-ZeroTrust-Lab`)
By configuring `AllowedIPs` strictly to the VPC and VPN subnets, regular workstation internet traffic remains routed through the local ISP while AWS internal traffic traverses the encrypted tunnel:

```ini
[Interface]
PrivateKey = <CLIENT_PRIVATE_KEY>
Address = 10.200.0.2/32
DNS = 1.1.1.1, 1.0.0.1

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
PresharedKey = <PRESHARED_KEY>
AllowedIPs = 10.10.0.0/16, 10.200.0.0/24
Endpoint = <BASTION_PUBLIC_IP>:51820
```

Once the tunnel handshake succeeded, direct SSH access to the private instance worked seamlessly without public exposure:

```powershell
ssh -i .\aws-lab-key.pem ubuntu@10.10.2.185
```

![WireGuard Split-Tunnel Active & Direct Private SSH](images/03-vpn-tunnel-active.png)

---

## 5. Troubleshooting & Lessons Learned

| Issue Encountered | Root Cause | Resolution |
| :--- | :--- | :--- |
| **`Start-Service ssh-agent` failed on Windows** | The OpenSSH Authentication Agent service defaults to `Disabled` on Windows workstations. | Configured `Set-Service -Name ssh-agent -StartupType Manual` prior to starting the service. |
| **`UNPROTECTED PRIVATE KEY FILE` warning** | Inherited NTFS read permissions (`BUILTIN\Users`) on the `.pem` key directory caused OpenSSH to ignore the identity file. | Stripped inherited ACLs using `icacls /inheritance:r` and granted read-only access exclusively to the active user. |
| **WireGuard tunnel active but SSH timed out** | The automated installer populated the Bastion's private interface IP (`10.10.1.80:51820`) as the client `Endpoint`. | Updated the client `[Peer]` configuration to point to the Bastion's public IPv4 address on UDP `51820`. |
| **Ubuntu MOTD `Failed to connect to changelogs.ubuntu.com`** | Expected behavior confirming the private subnet route table (`rtb-private1`) has zero outbound internet access. | Verified isolation integrity—internal servers remain unreachable from and unable to initiate traffic to the public internet. |

---

## 6. Resource Cleanup & Cost Verification
Upon completion and documentation capture:
1. Deactivated the `AWS-ZeroTrust-Lab` WireGuard tunnel on the client workstation.
2. Terminated both EC2 instances (`VPN-Bastion-Gateway` and `Internal-App-Server`) to release compute and attached EBS volumes.
3. Verified **0 allocated Elastic IPs** and **0 unattached EBS Volumes**, keeping non-compute VPC constructs (`Lab-ZeroTrust-vpc`, Subnets, Route Tables, IGW, and Security Groups) intact at **$0.00/hour**.