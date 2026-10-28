# 🛡️ homelab-infrastructure

> **Status: decommissioned (personal learning project).** The lab ran on a Proxmox host connected through a Windows machine acting as the gateway. It is no longer in use, see [Lessons learned](#-lessons-learned).

**TL;DR:** Proxmox-based home lab built to practise blue-team basics: DNS filtering, remote access without public inbound ports, endpoint telemetry in a SIEM (Wazuh), and a tested detection + automated response scenario (SSH brute force).

---

## ✅ What was implemented and tested

- **DNS filtering:** AdGuard Home in an unprivileged LXC, upstream resolution over DNS-over-HTTPS.
- **Remote access without public inbound ports:** Tailscale (WireGuard-based) with a dedicated LXC acting as subnet router.
- **Self-hosted password manager:** Vaultwarden in Docker inside an unprivileged LXC, reachable only through the tailnet via `tailscale serve`.
- **SIEM:** Wazuh VM with agents on the Windows gateway and the Proxmox host. Proxmox auth logs forwarded through a custom log pipeline (rsyslog + `auth.log`), custom alerts validated.
- **Detection and response test:** SSH brute force against the Proxmox host from Kali (WSL, `hydra`, custom wordlist).
  - Detection: Wazuh correlated repeated level 5 authentication failures into a **level 10 alert** (Wazuh's built-in SSH brute-force correlation).
  - MITRE ATT&CK: T1110 (Brute Force).
  - Response: Wazuh active response blocked the attacker IP with `iptables` on the hypervisor.
- **Hardening check:** `nmap` scan of the host, unnecessary service `rpcbind` (port 111) stopped, disabled and masked via `systemctl`; closure confirmed by a re-scan.
- **Backups and patching:** Proxmox native backups of LXCs, weekly backup of the Vaultwarden SQLite database and the AdGuard config; monthly host patching.

---

## 🏗️ Architecture

| Zone | Component | Notes |
| --- | --- | --- |
| Gateway | Windows host with Internet Connection Sharing (NAT) | Wi-Fi uplink, Ethernet cable to the Proxmox host. Workaround, because the server could not be connected to the home router. |
| Filtering | AdGuard Home (unprivileged LXC) | Primary DNS for the lab, DoH upstream. |
| Access | Tailscale subnet router (LXC `tailscale-gateway`) | No public inbound ports, authenticated access through the tailnet. |
| Services | Vaultwarden (Docker in an unprivileged LXC) | Reachable only through the tailnet over HTTPS. |
| Defense | Wazuh (VM) | Agents on the Windows gateway and the Proxmox host. |

**Important limitation:** ICS is plain NAT. It put the lab behind the Windows host without touching the home router, but there were no VLANs and no firewall rules between the zones, so this was **not** real network segmentation.

---

## 🛡️ Threat Model (Light)

- **What I protect:** privacy of the internal network, no publicly exposed services, limited resolution of known malicious domains.
- **Considered threats:** phishing and C2 beaconing, credential guessing against internal services, scanning and lateral movement inside the lab.
- **Controls in place:** DNS filtering (AdGuard Home), access only through Tailscale, unprivileged LXC containers (reduces the risk of container breakout), SIEM monitoring of the Windows gateway and the Proxmox host.

---

## 🔍 Detection

**Tested:**
- **SSH brute force (T1110):** repeated failed logins from Kali triggered a level 10 alert with the attacker's source IP, followed by automatic IP blocking (see evidence below).

**Standard Wazuh modules on monitored hosts (default configuration, not customised in this project):**
- Rootcheck (checks of system binaries and known rootkit signs).
- File Integrity Monitoring (changes in directories such as `/etc/` and `/usr/bin/`).

---

## 🧪 Security Validation & Active Response (Red/Blue Teaming)

To verify the SIEM's effectiveness and active response, I ran a controlled simulation:

- **Attack vector:** SSH brute force against the Proxmox hypervisor.
- **Simulation:** Kali Linux in WSL (Windows Subsystem for Linux) using `hydra` and a custom wordlist.

<img width="536" height="209" alt="Hydra and nmap output from the attack simulation" src="https://github.com/user-attachments/assets/9cffccb6-7889-4d70-898f-42982600c5ae" />

<img width="830" height="163" alt="Hydra brute force attempt against the Proxmox host" src="https://github.com/user-attachments/assets/2eda690d-d269-47a8-9d15-f12e9c8670f6" />

- **Detection (blue team):** the Wazuh manager aggregated individual level 5 authentication failures and generated a **level 10 alert**.

<img width="2763" height="570" alt="Wazuh level 10 alert after the brute force" src="https://github.com/user-attachments/assets/06da0f79-86b7-4504-a858-189155613315" />

- **Active response (automatic IP blocking):** the system dropped the attacker's IP with `iptables` at the hypervisor level, cutting off the source in real time.

---

## 🧰 Security Hygiene

- **Secrets management:** handled manually via environment variables. Migration to Mozilla SOPS was planned but not done.
- **Backups:** Proxmox native backups for LXCs. The Vaultwarden SQLite database and the AdGuard config were backed up weekly.
- **Patching:** host OS (Proxmox/Debian) patched monthly. Docker containers (Vaultwarden) updated through pull/rebuild cycles.
- **Hardening (attack surface reduction):** open ports audited with `nmap`. Unnecessary services on the hypervisor (e.g. `rpcbind` on port 111) stopped, disabled and masked via `systemctl`.

---

## 📊 Status & Roadmap

**Done**
- [x] AdGuard Home in an unprivileged LXC + DoH upstream.
- [x] Tailscale subnet router for access without public ports.
- [x] Wazuh SIEM with the Windows gateway connected.
- [x] Wazuh agent on Proxmox with a custom log pipeline (rsyslog + `auth.log`), custom alerts validated.
- [x] Active response: automatic IP blocking for brute force, tested with Hydra.
- [x] Attack machine: Kali Linux in WSL (keeps the load off the hypervisor).

**Planned, not implemented**
- [ ] Forwarding logs from AdGuard Home, Vaultwarden and Tailscale to Wazuh, with defined retention.
- [ ] Detection of DNS NXDOMAIN spikes (possible DGA activity).
- [ ] Detection of internal scanning and lateral movement between containers.
- [ ] Secrets management with Mozilla SOPS.
- [ ] Real network segmentation (separate router or managed switch, VLANs, firewall rules between zones).

---

## 📸 Evidence & Artifacts

### 1. Active defense & telemetry

**Wazuh agents overview:** telemetry collected from the hypervisor and the Windows gateway.

<img width="945" height="544" alt="Wazuh agents overview" src="https://github.com/user-attachments/assets/415aa69e-79db-4d66-a977-e37b080d37a3" />

**Active response action:** the attacker's connection is terminated (`Connection timed out`) after the SIEM-triggered block.

<img width="1353" height="257" alt="Attacker connection timed out after active response" src="https://github.com/user-attachments/assets/cf51a649-872d-40f5-bf65-ce20f96cc573" />

**Hardening validation:** nmap scan confirming that port 111 is closed.

<img width="483" height="146" alt="Nmap scan confirming port 111 is closed" src="https://github.com/user-attachments/assets/8b2b99c4-f2d9-4bda-b45b-353408a1d13c" />

### 2. Infrastructure foundations

<img width="938" height="1016" alt="AdGuard Home dashboard" src="https://github.com/user-attachments/assets/894b21f7-1870-4cf7-b383-c42617f07e9c" />

*DNS filtering: ad and telemetry blocking.*

<img width="752" height="395" alt="Tailscale subnet routing" src="https://github.com/user-attachments/assets/454d2b19-5a70-4285-96b3-61a8f91c47d0" />

*Subnet routing and access through the tailnet.*

<img width="1910" height="1021" alt="Wazuh event log aggregation" src="https://github.com/user-attachments/assets/7beabf66-4b42-4e83-97d8-79f9d27beca1" />

*Endpoint telemetry and event log aggregation.*

<img width="1919" height="1018" alt="Proxmox resource usage" src="https://github.com/user-attachments/assets/af961207-99a5-4937-ba5e-53658dc1b6bd" />

*Hypervisor resource utilization and containers.*

---

## 📝 Lessons learned

- The server could not be connected directly to the home router, so a Windows machine had to act as the gateway. That made the lab depend on one desktop PC and ruled out proper segmentation.
- Next time: dedicated router or managed switch, VLANs per zone, firewall rules between zones.
- Define log sources and retention first and verify each log pipeline end to end before listing it as a feature.
- Keep the actual Wazuh rules and configuration in the repository from day one, not only the description.

*Last updated: 2026-10-28*
