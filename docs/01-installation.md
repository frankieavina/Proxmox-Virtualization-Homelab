# 01 – Proxmox VE Installation & Initial Configuration

## Overview
This document details the installation and baseline configuration of Proxmox VE on Lenovo ThinkCentre systems using macOS for installer creation. The goal is to establish a reliable virtualization platform to host enterprise-style lab environments including Windows Active Directory, Linux infrastructure services, and clustered virtualization.

---

## Hardware Environment
- Platform: Lenovo ThinkCentre Tiny
- CPU: Intel-based platform
- Storage: Internal SSD
- Network: Gigabit Ethernet
- Management Client: macOS

---

## ISO Preparation (macOS)
The Proxmox VE installer ISO was written to USB using macOS Terminal to ensure reliable UEFI boot compatibility.

### Steps:
```bash
diskutil list
diskutil unmountDisk /dev/diskX
sudo dd if=~/Downloads/proxmox-ve_8.x.iso of=/dev/rdiskX bs=4m status=progress
diskutil eject /dev/diskX
```

This method ensures a raw disk image is written, avoiding common boot failures seen with GUI USB writers.

---

## BIOS Configuration (Lenovo ThinkCentre)

Key BIOS settings:

- Boot Mode: UEFI
- Secure Boot: Disabled
- Intel Virtualization Technology: Enabled
- VT-d / IOMMU: Enabled
- SATA Mode: AHCI
- USB Boot: Enabled

Boot menu accessed using **F12**.

---

## Proxmox VE Installation

The graphical installer was used for deployment.

### Key Configuration:
- Hostname: `pve-01.homelab.local`
- IP Address: Static
- Gateway: Ubiquiti UCG Ultra
- DNS: Ubiquiti UCG Ultra
- Root authentication enabled

Graphical installer was selected to ensure full hardware detection and streamlined configuration.

---

## Web Interface Access

After installation, Proxmox was accessed via:

```
https://<node-ip>:8006
```

Authentication:
- Username: `root`
- Realm: Linux PAM

---

## Repository Configuration & System Updates

By default, Proxmox enables enterprise repositories that require a paid subscription. These repositories were disabled and replaced with the community no-subscription repository.

### Enterprise Repository Removal
```bash
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/ceph.list
```

### No-Subscription Repository Enablement
```bash
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
```

### System Update
```bash
apt update && apt full-upgrade -y
```

---

## Results

- Proxmox VE successfully installed and fully updated
- Web management interface operational
- Virtualization extensions verified
- System prepared for VM creation and clustering

---

## Lessons Learned

- Lenovo ThinkCentre systems require precise UEFI-compatible USB creation
- macOS `dd` imaging provides the highest success rate
- Repository configuration is essential to avoid enterprise subscription errors
- CLI repository management is expected in production environments

---

## Next Steps

- Storage layout optimization
- VM template creation
- Active Directory deployment
- Multi-node Proxmox clustering
