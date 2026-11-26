# Case Study: Provisioning Chrooted FTP Access for Scoped Subdirectory Deployment in Virtualmin/ProFTPD

## Overview
* **Environment:** Linux Web Hosting Server managed via Webmin/Virtualmin, ProFTPD Daemon
* **Scope:** Delegated file deployment access for a single web subdirectory without exposing root virtual server files, databases, or SSH access.
* **Outcome:** Provisioned a dedicated, chroot-isolated FTP account (`DefaultRoot ~`) restricted strictly to a target subdirectory inside `public_html`.

---

## 1. Problem Statement
A content/project contributor required remote file upload access to manage a specific sub-project directory (`/home/<domain_user>/public_html/<project-folder>`) without having visibility or write permissions to the main website root, other subdomains, or MariaDB databases. 

During initial provisioning and testing, three common issues occurred:
1. **Unscoped System Account:** Creating a standard Linux user via Webmin (*System -> Users and Groups*) bypassed Virtualmin’s UID/GID mapping, risking file ownership mismatches with the Apache/Nginx web server.
2. **Shell & Access Classification Mismatch:** Creating a Website FTP Access User initially left the user's login shell unset (`<None>`), causing Virtualmin to classify the account as `Mail only` and preventing FTP authentication.
3. **Directory Traversal Exposure:** Even after enabling FTP login, ProFTPD allowed the user to navigate up the directory tree (`..`) into parent directories outside their assigned project folder.

---

## 2. Root Cause Analysis
* **Virtual Server Ownership:** Web server processes execute files under the primary virtual server owner's UID/GID. Sub-accounts must share this UID/GID mapping via Virtualmin's *Website FTP Access User* feature rather than standalone Webmin system accounts.
* **PAM / ProFTPD Shell Validation:** ProFTPD validates user login shells against `/etc/shells`. An empty (`<None>`) shell field in `/etc/passwd` invalidates FTP authentication and forces Virtualmin to treat the account as mail-only.
* **Missing Chroot Directive:** By default, ProFTPD sets the user's initial working directory to their home path but does not enforce a chroot jail unless `DefaultRoot ~` is explicitly enabled.

---

## 3. Resolution & Configuration Steps

### Step 1: Provision Virtualmin Website FTP Access User
1. Navigated to **Virtualmin -> `<domain>` -> Edit Users -> Add a website FTP access user**.
2. Configured user credentials and explicitly set **Home directory** to **Website subdirectory**, entering `<project-folder>` (relative to `public_html` to avoid nested `public_html/public_html/` paths).
3. Left **Granted access to databases** empty to ensure zero database exposure.

### Step 2: Assign Non-Interactive Login Shell & Verify FTP Protocol
1. In **Webmin -> System -> Users and Groups**, updated the sub-user's **Shell** from `<None>` to `/bin/false` (verified present in `/etc/shells`) to enforce non-SSH access while allowing ProFTPD authentication.
2. Confirmed in Virtualmin's **Mail and FTP Users** list that **Login access** updated to `FTP only`.
3. Validated client connectivity over standard **FTP (Port 21)** rather than SFTP (Port 22), confirming proper authentication and UID/GID alignment.

### Step 3: Enforce Directory Isolation (`Chroot`) in ProFTPD
1. Navigated to **Webmin -> Servers -> ProFTPD Server -> Files and Directories**.
2. Changed **Limit users to directories** from `Default` to **Home directory** (equivalent to enabling `DefaultRoot ~` in `proftpd.conf`).
3. Applied changes to reload the ProFTPD daemon without dropping active web services.

---

## 4. Verification
* **Authentication & Protocol:** Confirmed successful login via FileZilla using plain FTP on Port 21 (`230 User logged in`), while SSH/SFTP (Port 22) connections remain blocked.
* **Chroot Jail Enforcement:** Verified that the remote root path resolves strictly to `/` (mapped to `/home/<domain_user>/public_html/<project-folder>`), with parent directory traversal (`..`) completely disabled.
* **Web Server Compatibility:** Uploaded test assets and confirmed files inherit the primary domain owner's UID/GID, allowing seamless HTTP/HTTPS serving without permission errors.

---

## Key Takeaways
* Always provision scoped web deployment accounts through **Virtualmin's Website FTP Access Users** rather than raw Webmin system users to preserve UID/GID file permissions.
* Verify `/etc/shells` and user shell attributes (`/bin/false` or `/sbin/nologin`) whenever a Virtualmin FTP user defaults to `Mail only`.
* Always pair scoped FTP accounts with `DefaultRoot ~` in ProFTPD to prevent lateral directory browsing across shared virtual host directories.