# 🛡️ homelab-infrastructure

**Proxmox-based HomeLab focused on network security, lightweight virtualization (LXC), and DNS sinkholing, built as a practical Blue Team / SOC training range with a clear, phased architecture.**

## 📊 Status (Last updated: 2026-04-24)
- [x] AdGuard Home (unprivileged LXC) + DoH upstream
- [x] HDD offload via LXC bind mounts (reduce SSD wear)
- [ ] **Next:** Wazuh SIEM + agents
- [ ] Tailscale remote access
- [ ] Kali VM for controlled red teaming

---

## 📌 Project Overview
This repository documents the evolution of a hybrid HomeLab environment designed for:
* **SOC simulation** (centralized logging & detection)
* **Network-wide protection** (DNS sinkhole & privacy-friendly resolution)
* **Infrastructure efficiency** (storage lifecycle optimization / SSD wear reduction)

<img width="1672" height="581" alt="Zrzut ekranu 2026-04-24 094335" src="https://github.com/user-attachments/assets/4cd261f1-518f-408a-b70f-a973ee243d53" />

---

## 🏗️ Architecture Design (Phased Approach)
The infrastructure is split into logical security zones:

### 1. Gateway Zone (Windows ICS / NAT)
* Windows host connects to the home router via Wi‑Fi (internet uplink).
* Windows shares internet to the lab via a direct Ethernet cable to the Proxmox host.
* This creates a separate, dedicated lab subnet behind Windows (NAT/ICS).

### 2. Filtration Zone (AdGuard Home)
* Unprivileged LXC running AdGuard Home as primary DNS for the lab network.
* Upstream resolution via DNS-over-HTTPS (DoH).

### 3. Defense Zone (Blue Team – Wazuh)
* Planned Wazuh VM as a centralized SIEM/XDR platform.
* Agents will be deployed on Windows host, AdGuard LXC, and Proxmox host for telemetry and detections.

### 4. Access Zone (Tailscale)
* Zero-trust mesh VPN for remote administration and family access.
* **Goal:** No public inbound ports exposed.

---

## 🛠️ Technical Deep Dive

### 🔹 Network-Wide DNS Sinkhole & Privacy
AdGuard Home is deployed as the primary DNS server to intercept telemetry, ads, and malicious domains.
* Custom blocklists (e.g., OISD) act as a network “black hole”.
* All upstream DNS requests are routed through DoH (Quad9/Cloudflare), reducing ISP visibility and mitigating basic MITM-style tampering.

<img width="902" height="241" alt="image" src="https://github.com/user-attachments/assets/c7dceb58-9dce-4aad-a16e-69991668721e" />

### 🔹 Hardware Optimization: SSD Wear-Leveling & LXC Bind Mounts
To protect the primary SSD from constant database writes (DNS logs today, SIEM data later), heavy I/O is redirected to an **8TB HDD** using LXC bind mounts.
* **HDD mountpoint on host:** `/mnt/pve/Magazyn-8TB/`
* The AdGuard LXC is unprivileged, so host-side ownership is mapped via Linux user namespaces: `container root → host UID/GID 100000`.
* This keeps a stricter boundary between container and hypervisor while still allowing high-write workloads off the SSD.

<img width="637" height="117" alt="image" src="https://github.com/user-attachments/assets/60ff7085-ba1c-479b-b7cc-f451953cd0ca" />

### 🔹 Defense: Wazuh SIEM/SOC (In Progress)
Next phase is to introduce a Wazuh VM acting as the central “brain”:
* Collect logs and security events via Wazuh agents.
* Monitor: host integrity, suspicious authentication activity, and anomalies across Windows / Proxmox / AdGuard.

### 🔹 Zero-Trust Remote Access & Red Teaming
* **Tailscale (WireGuard-based):** Secure mesh network for authorized access without exposing public ports.
* **Kali Linux VM:** Controlled penetration testing node to simulate attack vs. detection scenarios against the lab environment.

---

## 🚀 Deployment Roadmap
- [x] Deploy AdGuard Home in an unprivileged LXC
- [x] Configure bind mounts for HDD storage optimization
- [ ] **Next:** Install Wazuh SIEM and deploy agents across the infrastructure
- [ ] Configure Tailscale for secure remote management
- [ ] Spin up Kali Linux for offensive security simulations
