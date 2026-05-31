# Storage Architecture: Bare-Metal OpenMediaVault Deployment & Multi-Protocol Network Resolution
> **Host Node:** homelabpi (Raspberry Pi 4 / Debian Linux)
> **Primary Services:** OpenMediaVault (OMV) Core, Samba Daemon (smbd), NFS Daemon (nfsd)
> **Target Client OS Environments:** Windows 10/11 Workstations, POSIX/Unix Clients (macOS)

---

## 1. Architectural Design & System Topology
This project details the deployment of a centralized, low-overhead Network Attached Storage (NAS) tier designed to deliver high-throughput file-sharing services across a heterogeneous local network.

### Design Decisions:
* **Host Environment:** Selected a bare-metal Debian minimal deployment via OpenMediaVault rather than resource-heavy virtualization (like Proxmox) to optimize the hardware's limited RAM and CPU constraints.
* **Storage Pool:** Formatted underlying disks using the **ext4** file system to guarantee stable Linux native permissions compliance and maximum driver performance.
* **Network Protocol Selection:** Standardized on **SMB/CIFS** to ensure native, high-speed integration with Windows workstations while maintaining compatibility with Unix-based systems.

---

## 2. Incident Logs and Resolutions

### 🚨 Incident 1: Cross-Platform SMB Session Collision & Authentication Loops

#### Problem Description
During client provisioning, a Windows workstation successfully mounted the primary network directory (`\\192.168.0.239\nas`). However, when attempting to concurrently map a restricted secondary directory (`\\192.168.0.239\photos`) using distinct access credentials, the Windows shell entered an infinite authentication loop, ultimately failing with a network rejection error.

#### Root-Cause Analysis
This failure stems from a core security design architecture within the Windows Network Redirector (`mrxsmb`). The Windows kernel enforces a strict **Single-Session-Per-Target-IP** policy. 

A single client machine cannot open multiple concurrent authenticated connections to the exact same destination IP address using different usernames or credential scopes. Because the Windows session table had already bound the local user to `192.168.0.239` for the public share, the secondary mount request created an immediate kernel-level routing deadlock.

#### Technical Resolution & Mitigation Blueprint
To resolve the session deadlock without altering backend user permissions, the local network namespace was split to present the single target node as two distinct logical entities to the Windows client.

1. **Purge Volatile Session Tables** 
	Flushed the local Windows network cache and Kerberos/NTLM authentication tokens via PowerShell to forcibly forget the stuck connections:
   ```powershell
	   net use * /delete /y
	   klist purge 
   ```


2. **Local Hostname Segmentation (The Initial Workaround)**
	With a clean session slate, the network namespace was initially split using local NetBIOS name resolution. By using distinct string identifiers, the Windows kernel was forced to log separate session contexts despite targeting the same physical host:

	- **Path A (IP-Bound):** `net use X: \\192.168.0.239\nas` (Mapped to Guest context)
    
	- **Path B (Name-Bound):** `net use Y: \\homelabpi\photos` (Mapped to Admin context via Samba's `nmbd` daemon)


### 🚨 Incident 2: Scaling Network Storage Beyond the LAN via Mesh VPN Overlay

#### The Architectural Limitation
While the local NetBIOS name resolution successfully tricked the Windows kernel logbook by leveraging string diversity, a critical architectural dependency was identified: **L2 Broadcast Reliance**. This configuration functions exclusively when the client is physically present on the local subnet (`192.168.0.0/24`) and can intercept local router broadcasts. Off-site client provisioning (e.g., via public WAN or remote access) breaks this resolution entirely.


#### Upgrade to Overlay Network Namespace Segmentation (The Final Solution)
To decouple the storage tier from physical location constraints, the infrastructure was scaled to utilize a virtual Mesh VPN overlay network: **Tailscale**.

![[tailscale-nas-architecture.png]]

By deploying the Tailscale daemon on both the host node and the client, the host was provisioned with a secure, permanent CGNAT IP address and a corresponding MagicDNS namespace identity string (`homelabpi.tailnet-xxxx.ts.net`).
	
The Windows Network Redirector (`mrxsmb`) no longer just sees a different string spelling—it sees an entirely separate, isolated **virtual network interface card (vNIC)** routing traffic to a completely distinct IP range (`100.x.x.x`). This fully resolves the routing deadlock permanently, regardless of physical location:

```powershell
# Drive 1: Local hardware path (LAN bound)
net use X: \\192.168.0.239\nas /user:nasguest /persistent:yes

# Drive 2: Encrypted overlay network path (Globally accessible)
net use Y: \\homelabpi.tailnet-xxxx.ts.net\photos /user:nasadmin /persistent:yes
#### Verification & System State
```

Following the deployment of the Tailscale overlay space, the Windows kernel successfully maintained two parallel, isolated security contexts. Network stability benchmarks confirmed concurrent, multi-user mounting capability across both paths with zero token degradation or authentication loops, whether connected locally or remotely via external networks.