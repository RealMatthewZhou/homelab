# 🖥️ Home NAS & Multi-Protocol Storage Infrastructure

A centralized, low-overhead Network Attached Storage (NAS) tier deployed on bare-metal hardware. This repository documents the architectural design decisions, cross-platform network resolutions, and real-world system engineering incidents encountered during the provisioning and scaling of the storage node.

---

## 🛠️ Hardware & Environment Stack

* **Host Node:** Raspberry Pi 4 (Model B)
* **Operating System:** Debian Linux (Minimal, Bare-Metal)
* **Storage Management:** OpenMediaVault (OMV) Core
* **File Sharing Daemons:** Samba (`smbd`), NetBIOS (`nmbd`), NFS (`nfsd`)
* **Overlay Network:** Tailscale VPN Daemon
* **Target Clients:** Windows 10/11 Workstations, macOS / POSIX Clients

---

## 📂 Repository Layout

The project structure separates core documentation logs from visual assets, and is as follows:

```text
README.md                           [File] Project overview and navigation guide
mkdocs.yml                          [File] Documentation configuration

attachments/                        [Folder] Embedded system diagrams and network maps
    tailscale-nas-architecture.png  [File] Visual infrastructure topology blueprint

storage-nas/                        [Folder] Core technical documentation logs
    OMV-Troubleshooting-Log.md      [File] Architectural spec, incident timelines, and fixes
```
