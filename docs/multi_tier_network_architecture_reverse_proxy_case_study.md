# Production Engineering Case Study: Multi-Tier Hybrid Network Architecture, Reverse Proxy, & SSL Decoupling

## 📌 Executive Summary
This project outlines the architectural design, implementation, and systematic troubleshooting of exposing an on-premise backend REST API securely to the public internet under a custom domain (`api.bp***.sch.id`). 

The challenge involved routing traffic through a public cloud VPS, navigating a multi-tier MikroTik router topology with dual-ISP load balancing, overcoming asymmetric routing via source NAT (Hairpin/Masquerading), and resolving cryptographic and DNS edge issues (Let's Encrypt ACME challenges, Virtualmin wildcard intercepts, and cipher suite negotiations).

---

## 🏗 High-Level Infrastructure Topology

```
[ Public Clients / Internet ]
           │
           ▼
[ Cloud VPS: "srv1" ] (Public IP: 103.***.***.51)
  ├── Nginx Edge Reverse Proxy (Port 80/443 SSL termination)
  └── Let's Encrypt Automated Certificate Management (ACME Webroot)
           │
           ▼ (Upstream Proxy to IP: 103.**.***.154:2****)
[ Edge Router: MikroTik LB ] (Dual-WAN: ISP1 + ISP2)
  ├── Inbound DST-NAT Translation
  └── Intra-Router Source NAT (SNAT/Masquerade to Host)
           │
           ▼ (Inter-Router Link: 192.***.***.253:2****)
[ Core Switch / Distribution Router: MikroTik Host ]
  └── Inbound DST-NAT Translation to LAN
           │
           ▼ (LAN Link)
[ On-Premise Target Server ] (Ubuntu Server - IP: 172.**.0.***:2****)
  └── Target Application API Service (`/health` endpoint)
```

---

## ⚙️ Network & Component Details (Sanitized)

| Component | Identifier / Interface | IP / Subnet | Port / Service | Role |
| :--- | :--- | :--- | :--- | :--- |
| **Edge DNS** | Cloudflare DNS | *N/A* | DNS (Proxy Disabled / DNS-Only) | Authoritative DNS resolution for subdomains |
| **Cloud VPS (`srv1`)** | CentOS / Webmin / Virtualmin | `103.***.***.51` | `80` (HTTP), `443` (HTTPS) | Public ingress, SSL/TLS termination, reverse proxy |
| **Edge MikroTik (LB)** | Dual WAN (`CORPNet`, `ASNet`) | WAN: `103.**.***.154` / LAN: `192.***.***.0/24` | `2****` (TCP) | Multi-WAN Load Balancer, Inbound NAT & Hairpin SNAT |
| **Core MikroTik (Host)** | Distribution Router | WAN-facing: `192.***.***.253` / LAN: `172.**.0.0/16` | `2****` (TCP) | LAN Gateway & Destination NAT |
| **Backend Target** | Ubuntu Linux Server | `172.**.0.***` | `2****` (TCP) | Production REST API Service (`api` process) |

---

## 🔍 Incident Diagnostics & Step-by-Step Resolutions

### 1. The "Ghost IP" & DNS Caching Anomaly
* **Symptom:** After pointing the A record for `api.bp***.sch.id` to `103.***.***.51`, requests from workstation browsers continuously resolved to Cloudflare CDN edge nodes.
* **Root Cause:** 
  1. Cloudflare Proxy status was initially set to **Proxied (Orange Cloud)** instead of **DNS Only (Grey Cloud)**.
  2. Local workstation OS (`ipconfig /flushdns`) and browser socket cache (`chrome://net-internals/#sockets`) retained cached Anycast DNS records across active sessions.
* **Resolution:** Toggled Cloudflare proxy mode to DNS-Only and cleared host resolver caches. Verified propagation via recursive external DNS lookups:
  ```bash
  nslookup api.bp***.sch.id 8.8.8.8
  ```

---

### 2. Upstream Port Mismatch & Socket Inspection
* **Symptom:** Traffic forwarded from the edge arrived at the on-premise network but timed out without reaching the application layer.
* **Root Cause:** Configuration documentation erroneously recorded the internal backend listening port as `8***`, causing MikroTik Host to translate traffic to an inactive socket.
* **Diagnostic Command:**
  ```bash
  sudo ss -tulpn | grep LISTEN
  ```
  *Result:*
  ```text
  tcp   LISTEN   0   4096   0.0.0.0:2****   0.0.0.0:*   users:(("api",pid=236226,fd=8))
  ```
* **Resolution:** Re-aligned all NAT translation chains across both MikroTik tiers to a consistent **1-to-1 port scheme (`2****` $\rightarrow$ `2****` $\rightarrow$ `2****`)**, ensuring immediate delivery to the listening daemon.

---

### 3. Asymmetric Routing over Dual-WAN (Multi-Homed LB)
* **Symptom:** Public inbound requests stalled, resulting in intermittent `SYN_RECV` drops and severe latency.
* **Root Cause:** On dual-ISP load-balancing routers, return packets from the target application were routed out via the default route table (often through the alternative ISP gateway), violating stateful TCP return paths.
* **Resolution:** Enforced an intra-network Source NAT (`srcnat`) rule on the MikroTik Load Balancer. By masquerading inbound packets targeting the Host distribution router, the target server was forced to send reply packets back directly to the LB gateway:
  ```routeros
  /ip firewall nat add chain=srcnat action=masquerade dst-address=192.***.***.253 protocol=tcp dst-port=2**** comment="MASQ: Fix Asymmetric Return Path"
  ```

---

### 4. Nginx Subdomain Hijack by Wildcard VHost
* **Symptom:** Accessing `http://api.bp***.sch.id` silently redirected or rendered content from the institution's primary apex website (`bp***.sch.id`).
* **Root Cause:** The default Virtualmin server configuration contained a catch-all wildcard server directive:
  ```nginx
  server_name bp***.sch.id 192.***.***.7 *.bp***.sch.id;
  ```
  Without a dedicated VirtualHost block for `api.bp***.sch.id` listening on the same IP-port pair, Nginx evaluated the wildcard pattern as the best match.
* **Resolution:** Implemented explicit HTTP and HTTPS server blocks for `api.bp***.sch.id` with higher specificity, terminating reverse proxy upstream requests to the internal edge gateway.

---

### 5. Let's Encrypt ACME Challenge Collision on Reverse Proxies
* **Symptom:** Executing `certbot certonly --webroot` returned:
  ```text
  Detail: 103.***.***.51: Invalid response from https://api.bp***.sch.id/.well-known/acme-challenge/...: 404
  ```
* **Root Cause:** The HTTP-to-HTTPS redirect forwarded ACME validation requests into the `location /` proxy block. The request was handed off to the backend API on-premise, which had no ACME token, returning HTTP `404`.
* **Resolution:** Isolated ACME verification requests at the Nginx edge by granting local filesystem precedence for challenge tokens:
  ```nginx
  location /.well-known/acme-challenge/ {
      root /home/bp***331924/public_html;
  }
  ```

---

### 6. Cipher Suite & TLS Handshake Negotiation Failure
* **Symptom:** Browser displayed `ERR_SSL_VERSION_OR_CIPHER_MISMATCH` upon accessing `https://api.bp***.sch.id`.
* **Root Cause:** Inheritance of legacy SSL ciphers from deprecated global includes (`include /etc/nginx/ssl.conf;`) conflicted with modern browser requirements under HTTP/2.
* **Resolution:** Overrode legacy ciphers in the specific virtual host block, explicitly enforcing modern TLS primitives:
  ```nginx
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers HIGH:!aNULL:!MD5;
  ssl_prefer_server_ciphers on;
  ```

---

## 🛠 Final Verified Production Configurations

### A. Cloud Edge Reverse Proxy (`/etc/nginx/nginx.conf` on `srv1`)
```nginx
# HTTP Ingress - Force Canonical HTTPS Redirection
server {
    listen 192.***.***.7:80;
    server_name api.bp***.sch.id;

    location /.well-known/acme-challenge/ {
        root /home/bp***331924/public_html;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

# HTTPS Ingress - SSL Termination & Upstream Proxy
server {
    listen 192.***.***.7:443 ssl;
    server_name api.bp***.sch.id;

    ssl_certificate /etc/letsencrypt/live/api.bp***.sch.id/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.bp***.sch.id/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location /.well-known/acme-challenge/ {
        root /home/bp***331924/public_html;
    }

    location / {
        proxy_pass http://103.**.***.154:2****;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
    }
}
```

### B. MikroTik Edge Load Balancer Configuration (RouterOS)
```routeros
# Inbound Destination NAT from VPS
/ip firewall nat add chain=dstnat action=dst-nat \
    dst-address=103.**.***.154 protocol=tcp dst-port=2**** \
    to-addresses=192.***.***.253 to-ports=2**** \
    comment="Forward WAN 2**** to Host Router"

# Source NAT to enforce symmetric routing back to LB
/ip firewall nat add chain=srcnat action=masquerade \
    dst-address=192.***.***.253 protocol=tcp dst-port=2**** \
    comment="MASQ to Host: Enforce Symmetric Return"
```

### C. MikroTik Core / Host Configuration (RouterOS)
```routeros
# Destination NAT from Host interface to On-Premise Target Server
/ip firewall nat add chain=dstnat action=dst-nat \
    protocol=tcp dst-port=2**** \
    to-addresses=172.**.0.*** to-ports=2**** \
    comment="Forward 2**** to Local Ubuntu Backend"
```

---

## 📈 Key Engineering Takeaways
1. **Network Symmetry in Multi-Tier Routing:** In nested NAT or multi-WAN environments, ingress pathing does not guarantee egress symmetry. Source NAT at the ingress router prevents packets from being discarded by ISP reverse-path forwarding (RPF) filters.
2. **Reverse Proxy Precedence:** Catch-all configurations in shared hosting panels (e.g., Virtualmin/cPanel) require explicit subdomain blocks with matched IP/port listeners to prevent unintended domain routing.
3. **Decoupled TLS Lifecycle Management:** When fronting internal microservices with a public reverse proxy, validating ACME challenges locally on the proxy node eliminates the need to configure certificate stores on internal backend application servers.
