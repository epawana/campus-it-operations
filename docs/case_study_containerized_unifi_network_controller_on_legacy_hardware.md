# Case Study: Migrating & Containerizing a UniFi Network Controller on Legacy Bare-Metal Hardware

## Overview

This repository documents the end-to-end migration of a production **UniFi Network Application (v10.x)** managing 12 wireless Access Points (APs) from a Windows desktop environment to an isolated, containerized Linux stack (**Ubuntu Server 24.04 LTS + Docker Compose**). 

Beyond routine service migration, this deployment required resolving low-level CPU instruction set incompatibilities on legacy enterprise hardware (**Intel Xeon Westmere-EP**), configuring Layer-2 host networking across a Hyper-V hypervisor, and implementing granular MongoDB Role-Based Access Control (RBAC) for automated backup restoration.

---

## 1. Problem Statement & Objectives

### Legacy Bottlenecks
* **Unreliable Service Availability:** The previous controller ran as a background application on a multipurpose Windows host, making wireless management vulnerable to OS auto-restarts, user session logouts, and resource contention.
* **Tight Coupling & Difficult Recovery:** Application binaries, Java runtimes, and embedded databases were tightly coupled to the host OS, complicating disaster recovery and future hardware migrations.

### Architectural Goals
1. **Service Isolation:** Decouple network management into a dedicated, headless Linux Virtual Machine with automated boot recovery.
2. **Infrastructure-as-Code (IaC) Portability:** Define the entire application and database stack declaratively via Docker Compose with persistent bind-mounted volumes.
3. **Zero-Downtime Migration:** Transition 12 active Access Points without disrupting end-user Wi-Fi connectivity.

---

## 2. Target Architecture

```text
+-------------------------------------------------------------------+
| Bare-Metal Host (Intel Xeon E5620 - Westmere-EP / Windows Hyper-V)|
|                                                                   |
|  +-------------------------------------------------------------+  |
|  | Hyper-V External Virtual Switch (Bridged to Physical LAN)   |  |
|  +-----------------------------+-------------------------------+  |
|                                |                                  |
|  +-----------------------------v-------------------------------+  |
|  | Guest VM: Ubuntu Server 24.04 LTS (Gen 2 / 2 vCPU / 4GB RAM)|  |
|  |                                                             |  |
|  |  +-------------------------------------------------------+  |  |
|  |  | Docker Engine (network_mode: host)                    |  |  |
|  |  |                                                       |  |  |
|  |  |  +------------------------+   +--------------------+  |  |  |
|  |  |  | unifi-network-app      |   | unifi-db           |  |  |  |
|  |  |  | (LinuxServer.io v10.x) +---> (MongoDB 4.4)      |  |  |  |
|  |  |  | Ports: 8443, 8080,     |   | Port: 27017        |  |  |  |
|  |  |  |        10001/udp       |   |                    |  |  |  |
|  |  |  +-----------+------------+   +---------+----------+  |  |  |
|  |  +--------------|--------------------------|-------------+  |  |
|  |                 v                          v                |  |
|  |        [ ./unifi-data ]           [ ./mongo-data ]          |  |
|  +-------------------------------------------------------------+  |
+-------------------------------------------------------------------+
```

### Key Design Decisions
* **`network_mode: host`:** UniFi APs rely on Layer-2 broadcast discovery (`UDP 10001`) and direct HTTP inform callbacks (`TCP 8080`). Using host networking avoids Docker bridge NAT translation overhead and preserves native L2 visibility across the LAN.
* **Hyper-V External Virtual Switch:** Connects the guest VM directly to the physical switching fabric with static IP addressing.

---

## 3. Engineering Challenges & Root Cause Analysis (RCA)

### Challenge 1: Container Crash Loop with `Exit Code 132` (`SIGILL`)
* **Symptom:** Upon deploying the initial stack with `mongo:7.0`, the `unifi-db` container immediately entered a restart loop returning exit code `132` (*Illegal Instruction*).
* **Root Cause Analysis:** 
  * Exit code `132` (`128 + 4`) indicates a `SIGILL` signal—the CPU attempted to execute an unsupported binary instruction.
  * Inspecting host CPU flags confirmed the bare-metal processor was an **Intel Xeon E5620 (Westmere-EP architecture)**, whose SIMD instruction set caps at **SSE 4.2**.
  * Starting with **MongoDB 5.0+**, official pre-compiled binaries require **AVX (Advanced Vector Extensions)**, which were first introduced in Intel Sandy Bridge processors.
* **Resolution:** Pinned the database container image to **`mongo:4.4`**, the final major MongoDB release compiled without mandatory AVX instructions, while maintaining full compatibility with the UniFi v10.x runtime.

### Challenge 2: `Unauthorized` Database Errors During Startup & `.unf` Backup Restoration
* **Symptom:** After resolving the database crash, the UniFi application failed context initialization (`Error 13: Unauthorized` on `unifi_audit`), and subsequent `.unf` backup restoration attempts failed silently with `not authorized on unifi_restore to execute command { dropDatabase: 1 }`.
* **Root Cause Analysis:** Modern UniFi Network Application releases dynamically interact with **four distinct MongoDB databases** rather than just the primary configuration and statistics stores:
  1. `unifi` (Core configuration)
  2. `unifi_stat` (Telemetry & client statistics)
  3. `unifi_audit` (System & admin audit logs)
  4. `unifi_restore` (Temporary staging database dropped/recreated during `.unf` archive imports)
* **Resolution:** Updated the MongoDB initialization script (`init-mongo.js`) to grant explicit `dbOwner` privileges across all four databases upon container bootstrap.

### Challenge 3: Volume Ownership Mismatch on Host Bind Mounts
* **Symptom:** Java runtime failed to write keystore and unpacked backup assets inside `./unifi-data`.
* **Root Cause Analysis:** Pre-created host directories were owned by `root:root`, whereas the LinuxServer.io container drops privileges to unprivileged user `PUID=1000` / `PGID=1000`.
* **Resolution:** Recursively aligned host bind-mount ownership (`chown -R 1000:1000`) prior to stack initialization.

---

## 4. Sanitized Configuration Reference

### `docker-compose.yml`
```yaml
services:
  unifi-db:
    image: docker.io/mongo:4.4
    container_name: unifi-db
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./mongo-data:/data/db
      - ./init-mongo.js:/docker-entrypoint-initdb.d/init-mongo.js:ro
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=<CHANGE_ME_ROOT_PASSWORD>

  unifi-network-application:
    image: lscr.io/linuxserver/unifi-network-application:latest
    container_name: unifi-network-application
    restart: unless-stopped
    network_mode: host
    depends_on:
      - unifi-db
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Jakarta
      - MONGO_USER=unifi
      - MONGO_PASS=<CHANGE_ME_UNIFI_PASSWORD>
      - MONGO_HOST=127.0.0.1
      - MONGO_PORT=27017
      - MONGO_DBNAME=unifi
      - MONGO_AUTHSOURCE=unifi
      - MEM_LIMIT=3072
      - MEM_STARTUP=1024
    volumes:
      - ./unifi-data:/config
```

### `init-mongo.js`
```javascript
db.getSiblingDB("unifi").createUser({
  user: "unifi",
  pwd: "<CHANGE_ME_UNIFI_PASSWORD>",
  roles: [
    { role: "dbOwner", db: "unifi" },
    { role: "dbOwner", db: "unifi_stat" },
    { role: "dbOwner", db: "unifi_audit" },
    { role: "dbOwner", db: "unifi_restore" }
  ]
});
```

---

## 5. Migration Execution & Verification

1. **Pre-Migration Preparation:** Exported a full `.unf` configuration backup from the legacy Windows controller and gracefully stopped the legacy Windows service to prevent split-brain AP inform conflicts.
2. **State Restoration:** Uploaded the `.unf` archive to the containerized controller (`https://<CONTROLLER_IP>:8443`).
3. **Post-Migration Verification:**
   * Verified TCP `8443` (Web UI) and TCP `8080` (Device Inform) socket listeners via `ss -tulpn`.
   * Confirmed all **12 Access Points** automatically re-adopted and transitioned to `Connected` state within ~90 seconds without dropping active client SSID sessions.
   * Authored an internal Standard Operating Procedure (SOP) documenting hardware constraints (preventing accidental upgrades to AVX-dependent MongoDB images) and cold-backup procedures.

---

## 6. Key Takeaways

* **Hardware-Aware Container Pinning:** Container abstraction does not isolate applications from host CPU instruction set limits. Always verify SIMD flags (`/proc/cpuinfo`) when deploying database engines on legacy hypervisors.
* **Least-Privilege Database Provisioning:** Understanding application-level database lifecycles (such as staging databases used during restore routines) prevents silent runtime failures in production.
* **Operational Handover:** Translating complex troubleshooting steps into clear, reproducible runbooks ensures long-term maintainability for future system administrators.
