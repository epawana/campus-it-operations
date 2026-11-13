# Case Study: Resolving Non-Standard Port Stripping & HTTP 404 Redirects on Nginx Reverse Proxy

* **Date:** November 13, 2026
* **Author:** Ega Pawana (IT Support Coordinator & Systems Administrator)
* **Environment:** Linux (Ubuntu Server), Nginx Reverse Proxy, Redmine (Ruby on Rails / Puma Backend)
* **Category:** Systems Administration & Web Operations

---

## 1. Executive Summary

During the internal deployment of a Redmine project management instance running behind an Nginx reverse proxy on a non-standard listening port (`8081`), users encountered `404 Not Found` errors immediately after submitting state-changing web forms (such as user registration). Root cause analysis revealed that Nginx's default `$host` variable stripped the port number from forwarded HTTP headers, causing the upstream Ruby on Rails application server to generate absolute `Location` redirect headers targeting default HTTP port `80`. By reconfiguring proxy header preservation (`$http_host`, `X-Forwarded-Port`) and implementing `proxy_redirect` rewrite rules at the reverse proxy layer, full application routing integrity was restored without requiring administrative modifications inside the upstream application.

---

## 2. Problem Statement & Architecture Context

### Infrastructure Topology
* **Client Request Endpoint:** `http://<INTERNAL_SERVER_IP>:8081`
* **Reverse Proxy Layer:** Nginx listening on TCP port `8081` (IPv4 & IPv6)
* **Upstream Application Server:** Redmine (Puma/Rails) bound locally to `127.0.0.1:3100`

### Incident Symptoms
1. Initial GET requests to `http://<INTERNAL_SERVER_IP>:8081/account/register` rendered the registration interface properly.
2. Upon form submission (`POST`), the upstream application issued an HTTP `302 Found` redirect response.
3. The browser redirected to `http://<INTERNAL_SERVER_IP>/login` (omitting `:8081`), resulting in a `404 Not Found` error because no web service was bound to port `80`.
4. Administrative access to the Redmine application settings panel (`Administration > Settings > Host name and path`) was unavailable, requiring an infrastructure-level resolution at the web server layer.

---

## 3. Root Cause Analysis

Inspection of the existing virtual host configuration (`/etc/nginx/sites-available/redmine`) identified the following header forwarding behavior:

```nginx
proxy_set_header Host $host;
```

In Nginx's variable evaluation order:
* **`$host`** evaluates the hostname from the request line or `Host` header while explicitly **stripping any port component**.
* Because the upstream Rails application did not receive the non-standard port (`8081`) in the `Host` or `X-Forwarded-Port` headers, it assumed standard port `80` semantics when constructing absolute redirect URLs.
* Furthermore, the lack of `proxy_redirect` directives meant Nginx passed upstream `Location` response headers back to the client unaltered.

---

## 4. Technical Implementation & Resolution

To resolve the issue cleanly at the infrastructure layer, the Nginx server block was updated to preserve full host-port context and rewrite upstream redirect locations dynamically.

### 4.1 Configuration Backup
Before modifying production proxy rules, a configuration snapshot was taken:
```bash
sudo cp /etc/nginx/sites-available/redmine /etc/nginx/sites-available/redmine.bak
```

### 4.2 Updated Nginx Virtual Host (`/etc/nginx/sites-available/redmine`)
```nginx
server {
    listen 8081;
    listen [::]:8081;

    server_name _;

    location / {
        proxy_pass http://127.0.0.1:3100;

        # Preserve original Host header including non-standard port (:8081)
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Rewrite upstream Location headers missing the external port
        proxy_redirect http://$host/ http://$http_host/;
        proxy_redirect http://127.0.0.1:3100/ http://$http_host/;

        proxy_http_version 1.1;
    }
}
```

### 4.3 Validation & Service Reload
Configuration syntax was verified prior to reloading the Nginx worker processes to ensure zero downtime:
```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 5. Verification & Operational Standards

### Verification Results
* Tested end-to-end user registration in an isolated browser session (`Incognito`) via `http://<INTERNAL_SERVER_IP>:8081/account/register`.
* Confirmed that `302` redirect responses retained the `:8081` port suffix (`http://<INTERNAL_SERVER_IP>:8081/login`), eliminating the `404 Not Found` fault.

### Standard Operating Baseline for Non-Standard Port Proxies
For future internal web services deployed behind Nginx on non-standard ports (ports other than `80` or `443`), all reverse proxy `location` blocks must standardize on `$http_host` and explicit `X-Forwarded-Port` propagation rather than `$host`.