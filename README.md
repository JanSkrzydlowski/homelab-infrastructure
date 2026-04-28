# 🛡️ homelab-infrastructure

**TL;DR:**
Proxmox-based SOC training range and secure self-hosted infrastructure. Currently features active DNS sinkholing, secure password management, and endpoint telemetry collection. Key security capabilities include zero public inbound ports (via Tailscale mesh VPN), strict unprivileged LXC segmentation, and active DNS-over-HTTPS.

## 📌 Key Security Features
- **Zero-Trust Remote Access:** No exposed public ports; all ingress is authenticated via WireGuard-based Tailscale mesh.
- **Strict Isolation:** Services run in unprivileged LXCs mapped to restricted host UIDs, preventing container breakout escalation.
- **DNS Sinkholing:** Network-wide blocking of telemetry, malicious domains, and ads via AdGuard Home.
- **Endpoint Telemetry:** Active host-level monitoring and Event Log forwarding via Wazuh.

---

## 🏗️ Architecture Design (Phased Approach)
The infrastructure is split into logical security zones:

### 1. Gateway Zone (Windows ICS / NAT)
- Windows host connects to the home router via Wi‑Fi (internet uplink).
- Windows shares internet to the lab via a direct Ethernet cable to the Proxmox host.
- **Justification:** Using Windows ICS provides immediate lab isolation and segmentation. It allows for dedicated infrastructure testing without risking or modifying the primary home ISP router configuration.

### 2. Filtration Zone (AdGuard Home)
- Unprivileged LXC running AdGuard Home as primary DNS for the lab network.
- Upstream resolution via DNS-over-HTTPS (DoH).

### 3. Access Zone (Tailscale VPN)
- Zero-trust mesh VPN active for remote administration.
- Dedicated LXC (`tailscale-gateway`) acts as a Subnet Router, allowing authenticated access to the lab network.

### 4. Self-Hosted Services Zone (Vaultwarden)
- Password management hosted locally via Vaultwarden.
- Running in an isolated Docker environment inside an unprivileged LXC.
- Accessible exclusively through the Tailnet via secure HTTPS (leveraging `tailscale serve` as an internal reverse proxy).

### 5. Defense Zone (Blue Team – Wazuh)
- Wazuh VM acts as the centralized SIEM/XDR platform.
- Agents deployed on endpoints (Windows Gateway, Proxmox) for continuous telemetry and detection.

---

## 🛡️ Threat Model (Light)

- **What I Protect:** Internal network privacy, zero public infrastructure exposure, and limiting resolution of known malware/C2 domains.
- **Primary Threats:** Phishing and malware C2 beaconing, credential stuffing against internal services, and internal lateral movement/service scanning.
- **Security Controls:** Network segmentation (ICS/NAT), DNS sinkholing (AdGuard), zero-trust access (Tailscale), and unprivileged containerization (LXC).

---

## 📡 Logging & Telemetry

| Source | Log Type | Destination | Retention Policy |
| --- | --- | --- | --- |
| AdGuard Home | DNS Queries / Blocks | Wazuh (Planned) | 30 Days |
| Proxmox Host | Syslog / Auth Logs | Wazuh | 90 Days |
| Vaultwarden | App Logs / Auth Attempts | Wazuh (Planned) | 90 Days |
| Tailscale | Access / Audit Logs | Wazuh (Planned) | 90 Days |
| Windows Gateway | Windows Event Logs (Sys/Sec) | Wazuh | 90 Days |

---

## 🔍 Detection Use Cases

- **SSH Brute Force Detection:** Automated correlation of multiple failed login attempts. *(Validated: System triggers Level 10 alert and identifies attacker source IP)*.
- **Rootkit & Anomaly Detection:** Continuous scanning of system binaries (`ls`, `ps`, `diff`) for unauthorized modifications via Rootcheck.
- **File Integrity Monitoring (FIM):** Real-time tracking of changes in critical directories like `/etc/` and `/usr/bin/`.
- **DNS NXDOMAIN Spikes:** Detection of potential DGA (Domain Generation Algorithm) activity from malware.
- **Unauthorized Lateral Movement:** Monitoring for internal port scanning or service enumeration between LXCs.

---

## 🛡️ Security Validation (Blue Team Operations)

To verify the SIEM's effectiveness, I conducted a controlled **Red Teaming** simulation:

1. **Attack Vector:** SSH Brute Force targeting the Proxmox hypervisor.
2. **Simulation:** 10+ failed authentication attempts from a Windows-based gateway.
3. **Observation:** The Wazuh Manager successfully aggregated individual Level 5 events and generated a **Level 10 (Critical)** alert.

<img width="1909" height="1003" alt="Zrzut ekranu 2026-04-28 210020" src="https://github.com/user-attachments/assets/785a9666-aec1-415d-aabb-854b46a5e51f" />

*Figure: Real-time detection of a brute-force attack. The system correctly correlates logs and flags the source IP.*

---
## 🛠️ Security Hygiene

- **Secrets Management:** Currently handled manually via environmental variables. Migration to Mozilla SOPS for encrypted Gitops storage is planned.
- **Backups:** Proxmox native backups for LXCs. Vaultwarden SQLite database and AdGuard config are backed up weekly to allow bare-metal restoration within 30 minutes.
- **Patching:** Host OS (Proxmox/Debian) patched monthly. Docker containers (Vaultwarden) updated via automated pull/rebuild cycles.

---
     
  - ## 📊 Status & Roadmap (Last updated: 2026-04-28)

- [x] Deploy AdGuard Home in an unprivileged LXC + DoH upstream.
- [x] Configure Tailscale Subnet Router for zero-trust access.
- [x] Deploy Wazuh SIEM and connect Windows Gateway endpoint.
- [x] **Wazuh Expansion:** Deploy agents to Proxmox and configure custom log pipelines (rsyslog + auth.log).
  - *Status:* **Success.** Custom alerts (Level 10) validated.
- [ ] **Active Response:** Configure automated IP blocking (shunning) for Brute Force attempts.
- [ ] **Offensive Security:** Deploy a dedicated Kali Linux VM for advanced lateral movement simulations.

---

## 📸 Evidence & Artifacts

<img width="938" height="1016" alt="Zrzut ekranu 2026-04-27 210851" src="https://github.com/user-attachments/assets/894b21f7-1870-4cf7-b383-c42617f07e9c" />

*Network-wide ad and telemetry blocking.*

<img width="752" height="395" alt="Zrzut ekranu 2026-04-27 211110" src="https://github.com/user-attachments/assets/454d2b19-5a70-4285-96b3-61a8f91c47d0" />

*Zero-trust subnet routing and access controls.*

<img width="1910" height="1021" alt="image" src="https://github.com/user-attachments/assets/7beabf66-4b42-4e83-97d8-79f9d27beca1" />

*Endpoint telemetry and event log aggregation.*

<img width="1919" height="1018" alt="Zrzut ekranu 2026-04-27 211204" src="https://github.com/user-attachments/assets/af961207-99a5-4937-ba5e-53658dc1b6bd" />

*Hypervisor resource utilization and container orchestration.*
