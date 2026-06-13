# MyOMV_NAS
A custom personal NAS built by repurposing old hardware running Debian 12 and OpenMediaVault on a USB pen drive, with secure worldwide access via Tailscale.

# 🖥️ Bare-Metal NAS & Private Cloud Infrastructure

## 📖 Introduction & Project Overview
This project documents the end-to-end transformation of an 15-year-old, depreciated desktop PC into a headless, secure, and globally accessible Network Attached Storage (NAS) server. Built entirely on a bare-metal Debian 12 (Bookworm) core and managed via OpenMediaVault (OMV 7), this server serves as a self-hosted private cloud, code repository backup environment, and high-speed local file share.

## ⚙️ Hardware Specifications
* **Motherboard:** Asus P8H61-M LX3 R2.0 (Legacy BIOS)
* **CPU:** Intel Pentium (Stock cooler)
* **Memory:** 8GB Dual-Channel DDR3 RAM 
* **Boot Drive / OS:** 16GB SanDisk Cruzer Blade (USB 2.0)
* **Data Drive (Sandbox):** 1TB Seagate Barracuda ST31000524AS 7200 RPM SATA HDD
* **Network:** Wired Gigabit Ethernet

## 🏗️ System Architecture & The "Why"

**Why a Flash-Drive Boot Setup?**
Due to physical hardware degradation in the internal SATA data cables and motherboard storage controllers (which triggered Asus Anti-Surge power cutoffs and controller drops under heavy write loads), the primary operating system was intentionally routed to boot externally via USB, isolating the system architecture from the failing internal buses.

**Diagnostic Methodology & Hardware Fault Isolation:**
Rather than blindly replacing components, the legacy hardware failures were isolated through rigorous low-level testing:

* **SSD Flash Controller Lockout:** Initial formatting attempts via Windows `diskpart` successfully wiped the partition tables in system memory.However, the Debian installer consistently threw `DID_BAD_TARGET` and `ext4 file system creation failed` I/O errors the moment sustained write operations were initiated. This confirmed the budget SSD had triggered a hardware-level NAND flash read-only lock to protect its remaining memory cells from electrical instability.
* **Electrical/SATA Degradation:** Using the original, degraded 90-degree SATA power splitters caused severe voltage sags when the mechanical hard drive motor spun up. The motherboard's Asus Anti-Surge protection instantly detected the unsafe electrical loop and hard-killed all system power to prevent catastrophic component failure.
* **S.M.A.R.T. Diagnostics (The Smoking Gun):** Once OpenMediaVault was successfully deployed on the USB, pulling the S.M.A.R.T. hardware ledger for the attached 1TB Seagate HDD provided undeniable proof of the physical breakdown:
  * *ID 199 (UDMA CRC Error Count):* Registered an astronomical 5038 errors. This metric specifically isolates data corruption occurring *during transit* between the motherboard and the drive, definitively proving the SATA cables and motherboard ports were failing.
  * *ID 187 & 197 (Uncorrectable/Pending Sectors):* Logged physical read-head failures and damaged sectors on the magnetic platters, forcing the drive into a critical "Bad" health status. 

*(Diagnostic Evidence)* Debian ext4 Mount Failure Error<img width="2670" height="1200" alt="Screenshot_20260613-145623_Google" src="https://github.com/user-attachments/assets/c027a351-6c8d-4f83-9dd6-342529a0aab4" />

OMV S.M.A.R.T. Ledger showing UDMA CRC Errors<img width="1392" height="991" alt="Screenshot 2026-06-13 at 3 05 37 PM" src="https://github.com/user-attachments/assets/49eb97ac-3c43-4d35-bfd0-107c01b7e1bd" />

**Preventing Flash Degradation (`folder2ram`):**
Running a full Debian OS on a standard USB flash drive typically destroys the drive's P/E (Program/Erase) cycles within weeks due to constant background system logging. This architecture utilizes the `openmediavault-flashmemory` plugin. It intercepts continuous write requests to system directories (like `/var/log` and `/var/tmp`) and writes them to a virtual RAM disk. Logs are only flushed to the physical USB drive periodically or during graceful shutdowns, extending the lifespan of the boot drive exponentially.

## 🌍 Worldwide Networking & Zero-Trust Access
To bypass the inherent security risks of port-forwarding on a home ISP router, this infrastructure utilizes a Peer-to-Peer Mesh VPN overlay.
* **Tailscale (WireGuard):** Enables secure, encrypted, global access to the server using a fixed overlay IP (`100.126.76.4`) without exposing any ports to the public internet (NAT Traversal).
* **mDNS (Bonjour) & SMB/CIFS:** Local network resolution is handled seamlessly via `pluto.local`. File sharing is strictly authenticated using dedicated user privileges (e.g., `mpluto` for admin Read/Write/Execute), with Guest/Anonymous access explicitly disabled at the SMB configuration level to prevent unauthorized local writes.

## 🛠️ Comprehensive Troubleshooting & Hurdles
This build required extensive low-level systems administration to bypass legacy hardware locks, network constraints, and filesystem stalemates.

### 1. Legacy BIOS Bootloader Rejection (Blinking Underscore)
* **The Challenge:** After successfully unpacking Debian, the legacy Asus BIOS refused to boot the OS, hanging indefinitely on a blinking underscore. The older motherboard required a hardcoded "Active" boot flag on Sector 0 to execute legacy Master Boot Records.
* **The Resolution:** Mounted the target drive on macOS, accessed the raw disk via terminal, and manually injected the active hex flag (`0x80`) into the Master Boot Record using `fdisk`:
    ```bash
    sudo fdisk -e /dev/disk4
    flag 1
    write
    ```

### 2. Dual-USB Controller Conflict & Missing GRUB
* **The Challenge:** The older motherboard's USB host controller choked when attempting to initialize two identical 16GB SanDisk drives on the same controller bus, actively hiding the target OS drive from the BIOS Boot Menu and causing the installer to skip GRUB compilation.
* **The Resolution:** Executed a "Linux Hotplug Bypass." Booted solely from the installer USB, waited for the Debian Linux kernel to load into memory, and *then* hot-plugged the target OS drive. Dropped into the Debian Rescue Shell, bypassed the BIOS blindspot, and manually compiled the bootloader directly to the hardware:
    ```bash
    grub-install /dev/sdb
    ```

### 3. Ghost Partitions & Kernel Cache Locks
* **The Challenge:** The Debian installer continuously failed to build the `ext4` filesystem (`ext4 file system creation failed`). The system RAM cache was stubbornly holding onto old partition block maps from previous OS attempts, flagging the disk sectors as "busy" and rejecting format commands.
* **The Resolution:** Exited the graphical installer into a background BusyBox terminal (`Ctrl+Alt+F2`) and pulverized the partition headers by writing pure zeros to the raw silicon. Forced the Linux kernel to drop the cache locks by re-polling the hardware via `Detect disks`:
    ```bash
    dd if=/dev/zero of=/dev/sda bs=1M count=10
    wipefs -a /dev/sda
    ```

### 4. Aggressive ISP Router & Dynamic IP Traps
* **The Challenge:** Attempting to assign a static IP (`192.168.1.16`) to the server at the OS level resulted in the ISP-locked Genexis router retaliating. It severed the server's internet access, dropping the Tailscale tunnel and crashing the `SSH` service (`Connection refused`).
* **The Resolution:** Reverted the `enp3s0` interface to DHCP via the OMV Web GUI. Shifted network reliance entirely to Tailscale's permanent overlay IP and Apple's `mDNS` protocol (`pluto.local`), ensuring rock-solid connection stability regardless of underlying local DHCP shifts.

### 5. macOS SMB Caching Tantrums
* **The Challenge:** macOS Finder refused to authenticate administrative credentials over SMB, returning "Invalid Password" because it aggressively cached a previous anonymous "Guest" session token in its background keychain.
* **The Resolution:** Force-killed the macOS network authentication daemon to flush the ghost tokens, and injected the username directly into the URI string to force a secure handshake:
    ```bash
    sudo killall NetAuthAgent
    killall Finder
    # Connected via explicit path: smb://mpluto@100.126.76.4
    ```

### 6. Old Linux Root Permissions on Legacy Drives
* **The Challenge:** The repurposed 1TB hard drive contained an old Ubuntu installation. Authenticated SMB connections lacked the native `root` authority to delete protected core directories like `/bin` and `lost+found`.
* **The Resolution:** SSH'd directly into the server as root, recursively vaporized the old OS directories bypassing the SMB layer, and explicitly transferred directory ownership to the primary admin account:
    ```bash
    rm -rf /srv/dev-disk-by-uuid-xxx/*
    chown -R mpluto:users /srv/dev-disk-by-uuid-xxx/
    ```

## 🚀 Future Expansion Plans
* **Hard Drive Lifecycle Replacement:** The current 1TB Seagate drive has exceeded 16,500 power-on hours and is actively logging `UDMA CRC` and `Uncorrectable Sector` errors. It currently serves strictly as an isolated, sacrificial sandbox and will be replaced with an enterprise-grade NAS HDD (e.g., Seagate IronWolf or WD Red Plus) combined with a new SATA cable.
* **Docker & Immich Deployment:** Once stable, fault-tolerant storage is acquired, Docker will be deployed to self-host `Immich`—a high-performance, machine-learning-powered Google Photos alternative for automatic mobile backups.
* **Data Pipelines:** Leverage the robust storage infrastructure for continuous integration, code repository backups, and hosting OpenCV computer vision datasets.
<img width="2670" height="1200" alt="Screenshot_20260613-145623_Google" src="https://github.com/user-attachments/assets/46e5ebc8-cabc-447d-96f7-e3601ff3b2d7" />
