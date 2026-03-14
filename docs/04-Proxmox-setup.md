# Phase 4 — Proxmox VE Setup

## Overview

This phase documents the Proxmox VE configuration for both nodes after the base OS installation (covered in Part 1). The focus here is post-install hardening, storage integration with Unraid, optional clustering, and the initial VM deployment strategy.

Both Proxmox nodes are **on-demand** — they are powered on for lab work and powered off when not in use. This is intentional to manage home power consumption.

---

## Node Reference

| Node | Hostname | Device | CPU | RAM | IP Address |
|---|---|---|---|---|---|
| pve01 | `pve01.homelab.local` | Lenovo ThinkCentre M720q | Intel i5-9400T | 8 GB | 192.168.x.10 |
| pve02 | `pve02.homelab.local` | Lenovo ThinkCentre M710q | Intel i5-7500T | 8 GB | 192.168.x.11 |

> Both nodes are currently running 8 GB RAM. RAM upgrade to 16 GB per node
> is planned. Until then, avoid running Windows Server and multiple Linux VMs
> simultaneously on the same node.

---

## Part 1 Recap — What Was Already Done

> Part 1 covered the base Proxmox VE installation. The following was completed:
> - Proxmox VE ISO downloaded and flashed to USB
> - OS installed on both nodes with static IPs set during install
> - Hostnames configured: `pve01` and `pve02`
> - Initial web UI access verified at `https://<node-ip>:8006`

---

## Post-Install Configuration

Complete the following on **both nodes** after installation.

### 1. Switch to No-Subscription Repository

The default enterprise repo requires a paid subscription. Replace it with the free no-subscription repo.

```bash
# Disable enterprise repo
echo "# disabled" > /etc/apt/sources.list.d/pve-enterprise.list

# Add no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

# Update and upgrade
apt update && apt full-upgrade -y
```

### 2. Remove Subscription Nag (Optional)

```bash
# Removes the "No valid subscription" popup in the web UI
sed -Ezi.bak "s/(Ext.Msg.show\(\{[^}]+title: gettext\('No valid sub)/void\(\{\/\/\1/g" \
  /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy
```

### 3. Set DNS Servers

```bash
# Edit resolv.conf
nano /etc/resolv.conf

# Add the following (use your router IP and a fallback)
nameserver 192.168.x.1
nameserver 1.1.1.1
```

### 4. Configure NTP Time Sync

Accurate time is important for clustering and logging.

```bash
# Check current time sync status
timedatectl status

# Edit NTP config if needed
nano /etc/chrony/chrony.conf

# Add or verify pool line:
# pool 2.debian.pool.ntp.org iburst
systemctl restart chrony
```

### 5. Enable IOMMU (for GPU / PCI Passthrough — optional)

If you plan to pass through hardware to VMs:

```bash
# Edit GRUB config
nano /etc/default/grub

# For Intel CPUs, change GRUB_CMDLINE_LINUX_DEFAULT to:
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"

# Apply changes
update-grub

# Add kernel modules
echo -e "vfio\nvfio_iommu_type1\nvfio_pci\nvfio_virqfd" >> /etc/modules
update-initramfs -u -k all

# Reboot
reboot
```

---

## Storage Configuration

### Default Storage Layout (Post-Install)

| Storage ID | Type | Location | Default Use |
|---|---|---|---|
| `local` | Directory | `/var/lib/vz` | ISO images, CT templates, backups |
| `local-lvm` | LVM-Thin | Internal SSD | VM disks, container rootfs |

### Add Unraid NAS as NFS Storage

Unraid serves as centralized storage for ISO images and VM backups. Mount it on both nodes.

#### Step 1 — Create NFS Share on Unraid

In the Unraid web UI:
1. Go to **Shares** → **Add Share**
2. Name the share `proxmox`
3. Set export to `Yes` with security `Public` (restrict to LAN only)
4. Note the Unraid NAS IP address

#### Step 2 — Mount NFS on Proxmox

In the Proxmox web UI on each node:
1. Go to **Datacenter** → **Storage** → **Add** → **NFS**
2. Fill in:

| Field | Value |
|---|---|
| ID | `unraid-nfs` |
| Server | `192.168.x.20` (Unraid IP) |
| Export | `/mnt/user/proxmox` |
| Content | ISO image, VZDump backup file, Container template |
| Nodes | Select both pve01 and pve02 |

#### Verify NFS Mount

```bash
# On each Proxmox node
pvesm status

# Should show unraid-nfs as active
# Test by listing contents
ls /mnt/pve/unraid-nfs/
```

---

## Networking — VLAN-Aware Bridge

To allow VMs to be placed on specific VLANs (when implemented in Phase 5), configure the Linux bridge as VLAN-aware on both nodes.

```bash
# Edit network interfaces
nano /etc/network/interfaces
```

Replace the existing `vmbr0` block with:

```
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.x.10/24        # Use .11 on pve02
    gateway 192.168.x.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

```bash
# Apply network changes
ifreload -a

# Verify bridge is up
ip addr show vmbr0
```

> With `bridge-vlan-aware yes`, you can assign a VLAN tag to any VM's
> network device in the VM hardware settings. Tag 20 = Servers VLAN,
> Tag 30 = Users VLAN, etc.

---

## Optional — Cluster Setup

Clustering both nodes provides a unified web UI, enables VM live migration between nodes, and is a valuable skill for the portfolio.

> **Note:** A 2-node cluster has no quorum majority. To avoid split-brain,
> add the Raspberry Pi as a QDevice (tie-breaker). This is covered below.

### Create the Cluster (run on pve01 only)

```bash
pvecm create homelab-cluster
```

### Join the Cluster (run on pve02 only)

```bash
pvecm add 192.168.x.10
```

### Verify Cluster Status

```bash
pvecm status
pvecm nodes
```

### Add Raspberry Pi as QDevice (tie-breaker for 2-node cluster)

```bash
# On Raspberry Pi — install corosync-qnetd
apt update && apt install corosync-qnetd -y

# On pve01 — add the QDevice
pvecm qdevice setup 192.168.x.30

# Verify quorum is healthy
pvecm status
```

---

## VM Deployment Strategy

### RAM Budget Per Node (8 GB)

Proxmox itself uses approximately 1–1.5 GB of RAM for the host OS. Plan VM allocations accordingly.

| Node | Host OS | Available for VMs | Recommended Max VMs |
|---|---|---|---|
| pve01 (8 GB) | ~1.5 GB | ~6.5 GB | 2–3 small VMs |
| pve02 (8 GB) | ~1.5 GB | ~6.5 GB | 1–2 VMs (Windows needs 4 GB) |

### Recommended Initial VM Layout

#### pve01 — Proxmox Node 1 (M720q, i5-9400T)

| VM Name | OS | vCPU | RAM | Disk | Purpose |
|---|---|---|---|---|---|
| `ubuntu-services` | Ubuntu Server 22.04 | 2 | 2 GB | 32 GB | Pi-hole, Nginx Proxy Manager, Uptime Kuma via Docker |
| `monitoring` | Ubuntu Server 22.04 | 2 | 2 GB | 32 GB | Prometheus + Grafana |
| `pbs` | Proxmox Backup Server | 2 | 2 GB | 32 GB | VM backup — targets Unraid NFS share |

#### pve02 — Proxmox Node 2 (M710q, i5-7500T)

| VM Name | OS | vCPU | RAM | Disk | Purpose |
|---|---|---|---|---|---|
| `win-server` | Windows Server 2022 | 2 | 4 GB | 60 GB | Active Directory lab |
| `pfsense-lab` | pfSense | 1 | 1 GB | 16 GB | Firewall / VLAN practice |
| `kali` | Kali Linux | 2 | 2 GB | 40 GB | Security lab (on demand) |

> Do not run `win-server` and `kali` simultaneously on pve02 — not enough
> RAM. Power on only the VM needed for the current lab session.

### Creating a VM — General Steps

1. In Proxmox web UI, click **Create VM**
2. Set VM ID and name
3. Select ISO from `unraid-nfs` storage
4. Set disk size on `local-lvm`
5. Assign vCPU count (do not exceed physical core count)
6. Assign RAM
7. Set network bridge to `vmbr0`, assign VLAN tag if applicable
8. Review and create
9. Start VM and complete OS installation

### Creating a VM Template (recommended for Linux VMs)

Templates save time when spinning up new lab VMs.

```bash
# After installing and configuring a base Ubuntu VM:

# Inside the VM — install qemu-guest-agent and clean up
apt install qemu-guest-agent -y
apt autoremove -y && apt clean
cloud-init clean    # if cloud-init is installed
poweroff

# Back on Proxmox host — convert VM to template
qm template <vmid>

# Clone from template when needed
qm clone <template-vmid> <new-vmid> --name new-vm --full
```

---

## Useful Proxmox CLI Commands

```bash
# List all VMs and containers
qm list
pct list

# Start / stop a VM
qm start <vmid>
qm stop <vmid>

# View VM config
qm config <vmid>

# Check cluster status
pvecm status

# Check storage status
pvesm status

# View resource usage
pvesh get /nodes/<nodename>/status

# Check Proxmox version
pveversion -v
```

---

## Snapshots and Backups

### Manual Snapshot (quick lab restore point)

```bash
# Take a snapshot before making major changes to a VM
qm snapshot <vmid> <snapname> --description "before AD install"

# List snapshots
qm listsnapshot <vmid>

# Rollback
qm rollback <vmid> <snapname>
```

### Scheduled Backup to Unraid via PBS

1. In Proxmox web UI → **Datacenter** → **Backup**
2. **Add** a backup job:
   - Storage: `unraid-nfs` or PBS instance
   - Schedule: Weekly (lab environment — daily is unnecessary)
   - Mode: Snapshot
   - Select VMs to include
3. Run a manual backup first to verify it works

---

## Troubleshooting

| Issue | Resolution |
|---|---|
| Web UI not accessible | Check `systemctl status pveproxy` — restart if needed |
| VM won't start — KVM error | Verify CPU virtualization is enabled in BIOS (VT-x for Intel) |
| NFS storage shows inactive | Verify Unraid is powered on and NFS share is exported |
| Cluster node shows offline | Check network connectivity and `corosync` service status |
| No internet from VM | Verify bridge config and gateway setting in VM network settings |

---

## What Was Learned

> Complete this section as you work through the lab.

- [ ] How Proxmox manages storage with LVM-Thin vs directory storage
- [ ] How to configure a VLAN-aware Linux bridge
- [ ] How to mount NFS storage from Unraid
- [ ] How to create and clone VM templates
- [ ] How Proxmox clustering works and why quorum matters
- [ ] How to take snapshots and schedule backups

---

## Next Phase

**[Phase 5 — VLAN Configuration](phase-5-vlan-configuration.md)** covers implementing 802.1Q VLAN segmentation on the GS724T, UCG Ultra firewall rules, and Proxmox VM VLAN tagging.
