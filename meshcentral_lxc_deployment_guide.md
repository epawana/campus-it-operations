# MeshCentral Centralized Remote Management Deployment

## 1. Overview

This guide outlines the deployment and configuration of **MeshCentral**, an open-source, web-based remote management and device control solution. The service is hosted inside an unprivileged Linux Container (LXC) on Proxmox VE, decoupling operational device management from host-level infrastructure while providing centralized endpoint control, remote terminal/desktop access, and device group policies.

## 2. Infrastructure & Runtime Specs

| Parameter | Specification | Notes |
| :--- | :--- | :--- |
| **Parent Host** | Proxmox VE | Reference: [`proxmox-ve-setup.md`](../infrastructure/proxmox-ve-setup.md) |
| **Instance Type** | LXC Container (Unprivileged) | Debian / Ubuntu Minimal Template |
| **Allocated Resources** | 2 vCPU / 2048 MB RAM / 20 GB Disk | Low idle overhead |
| **Runtime Engine** | Node.js (LTS v18/v20) & npm | Managed via systemd |
| **Database** | Embedded NeDB / MongoDB | Scalable depending on endpoint volume |
| **Service Ports** | `443/TCP` (HTTPS / Agent traffic), `80/TCP` (Redirect) | SSL termination handled internally or via Reverse Proxy |

---

## 3. Container Provisioning & Base Environment

### 3.1 Proxmox LXC Initialization (CLI Alternative)

Create and start the LXC container on the Proxmox host:

```bash
# Provision container with ID 200
pct create 200 local:vztmpl/debian-12-standard_12.2-1_amd64.tar.zst \
  --hostname meshcentral-srv \
  --cores 2 \
  --memory 2048 \
  --swap 512 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.150/24,gw=192.168.1.1 \
  --storage local-lvm \
  --unprivileged 1 \
  --onboot 1

# Start container
pct start 200
```

### 3.2 Runtime Dependencies Installation

Access the container console (`pct enter 200` or via SSH) and install Node.js and build utilities:

```bash
# Update repository index
apt update && apt upgrade -y

# Install Node.js LTS repository & dependencies
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs build-essential git

# Verify installed versions
node -v
npm -v
```

---

## 4. MeshCentral Installation & Configuration

### 4.1 Directory Setup & Installation

Install MeshCentral in a dedicated directory:

```bash
# Create dedicated user & deployment directory
useradd -m -s /bin/bash meshadmin
mkdir -p /opt/meshcentral
cd /opt/meshcentral

# Install MeshCentral locally
npm install meshcentral
```

### 4.2 Configuration Structure (`config.json`)

Generate the initial template by running MeshCentral once or create `/opt/meshcentral/meshcentral-data/config.json`:

```json
{
  "$schema": "https://meshcentral.com/schemas/config.json",
  "settings": {
    "cert": "meshcentral.local.domain",
    "wanonly": true,
    "port": 443,
    "aliasPort": 443,
    "redirPort": 80,
    "allowFraming": false,
    "selfUpdate": false,
    "minUserLevel": 0
  },
  "domains": {
    "": {
      "title": "Operations Device Management",
      "title2": "Internal Control Center",
      "auth": "internal",
      "footer": "Controlled Environment - Authorized Access Only",
      "newAccounts": false,
      "userAllowedIP": "192.168.1.0/24",
      "sessionTimeout": 60
    }
  }
}
```

*Key Settings:*
- `"newAccounts": false`: Restricts self-registration after primary administrative account creation.
- `"wanonly": true`: Enforces external host addressing if exposed through a perimeter reverse proxy.
- `"sessionTimeout": 60`: Enforces inactivity logouts after 60 minutes.

---

## 5. Process Automation with Systemd

Create a persistent service unit to ensure high availability and auto-start upon container initialization:

File: `/etc/systemd/system/meshcentral.service`

```ini
[Unit]
Description=MeshCentral Remote Management Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/meshcentral
ExecStart=/usr/bin/node /opt/meshcentral/node_modules/meshcentral
Restart=always
RestartSec=10
Environment=NODE_ENV=production

# Security Hardening
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

Enable and start the service daemon:

```bash
systemctl daemon-reload
systemctl enable --now meshcentral.service
systemctl status meshcentral.service
```

---

## 6. Device Groups & Agent Deployment SOP

### 6.1 Device Group Segregation
Organize target assets into distinct Device Groups from the web console (`https://<LXC_IP>`):
1. **Critical Infrastructure**: Hypervisors, Gateway servers, DNS appliances.
2. **Internal Workstations**: On-premise desktops (e.g., Windows 11 Workstation on VM 100).
3. **Field Laptops / Remote Nodes**: Roaming or telework clients communicating over encrypted tunnels.

### 6.2 Agent Deployment Workflows

* **Windows Systems (VM 100 & Clients):**
  1. Open Web UI > Target Device Group > **Add Agent**.
  2. Download `MeshCentralAgent.exe` and execute as Local Administrator.
  3. Verify service registration (`Mesh Agent` service running in Windows).
* **Linux Endpoints:**
  ```bash
  # Execute quick-install one-liner generated by MeshCentral
  curl -sSL "https://meshcentral.local.domain/meshagents?script=1" | sudo bash
  ```

---

## 7. Operational Runbook & Maintenance

### Service Health Checks & Logs

```bash
# Live runtime logs
journalctl -u meshcentral -f

# Check active listening ports
ss -tulpn | grep node
```

### Backup and Disaster Recovery

All critical configs, SSL encryption certificates, and state databases reside in `meshcentral-data/`:

```bash
# Routine directory backup
tar -czvf /backup/meshcentral-data-$(date +%F).tar.gz /opt/meshcentral/meshcentral-data/
```