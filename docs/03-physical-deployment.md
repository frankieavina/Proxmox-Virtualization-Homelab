# Phase 3 — Physical Deployment

## Overview

This phase documents the physical setup of the homelab — cable runs, device placement, switch port assignments, and initial connectivity verification. The goal is a clean, documented physical infrastructure that mirrors how a small enterprise network would be deployed.

---

## Pre-Deployment Checklist

Before connecting any devices, verify the following:

- [ ] Cat6 cable run from living room to bedroom 2 is complete and tested
- [ ] All devices are physically positioned and powered off
- [ ] Cable tester used to verify continuity on the backbone run
- [ ] All cables are labeled at both ends
- [ ] Switch is rack-mounted or shelf-mounted and accessible
- [ ] Power strips / surge protectors are in place for server room

---

## Step 1 — Living Room Setup

The living room contains only the ISP modem, the UCG Ultra router, and the Deco W6000 AP. There is no dedicated switch in this zone.

### Connections

| From | To | Port / Notes |
|---|---|---|
| ISP coax / fiber | ISP Modem | ISP-required location |
| ISP Modem (LAN) | UCG Ultra (WAN port) | Single Ethernet cable |
| Deco W6000 LR | — | No wired uplink — connects via wireless mesh backhaul to Deco SR |
| Deco W6000 LR (eth port 1) | Smart TV | Wired connection via AP's built-in switch port |

### Configuration Notes

- The Deco W6000 in the living room is a **satellite node** in the mesh — it has no wired uplink to the router
- All traffic from the living room routes wirelessly to the server room Deco, then through the GS724T to the UCG Ultra
- The UCG Ultra handles all DHCP, routing, and firewall for the entire network

---

## Step 2 — Backbone Cable Run

A single Cat6 cable connects the living room to the server room. This is the only inter-room Ethernet run in the entire homelab.

### Cable Details

| Parameter | Value |
|---|---|
| Cable label | `BACKBONE-LR-TO-SR` |
| From | UCG Ultra LAN port |
| To | GS724T Port 1 (uplink) |
| Cable type | Cat6 |
| Tested | Yes — verify continuity before connecting |

### Steps

1. Route cable from UCG Ultra through wall/attic/baseboard to Bedroom 2
2. Label both ends `BACKBONE-LR-TO-SR` before terminating
3. Test with cable tester — verify all 8 conductors pass
4. Connect living room end to UCG Ultra LAN port
5. Connect server room end to GS724T Port 1

> This single cable carries all traffic between the router and the entire
> server room infrastructure. Ensure it is Cat6 rated and properly seated.

---

## Step 3 — Server Room Setup (Bedroom 2)

All server and infrastructure devices live in this room. The GS724T acts as the central hub for everything.

### Device Placement Order

Power on devices in this order after cabling is complete:

1. Netgear GS724T (core switch)
2. Unraid NAS
3. Deco W6000 (server room AP)
4. Raspberry Pi 5
5. Proxmox Node 1 (M720q)
6. Proxmox Node 2 (M710q)

> Proxmox nodes and Unraid are **on-demand only** — power them on when
> beginning lab work, power off when done. The switch, Pi, and Deco AP
> run 24/7.

---

## GS724T Switch Port Map

| Port | Device | Cable Label | VLAN (future) |
|---|---|---|---|
| 1 | UCG Ultra (backbone uplink) | `BACKBONE-LR-TO-SR` | Trunk |
| 2 | Unraid NAS (eth0) | `NAS-01` | 20 |
| 3 | Proxmox Node 1 — M720q | `PVE-01` | 20 |
| 4 | Proxmox Node 2 — M710q | `PVE-02` | 20 |
| 5 | Raspberry Pi 5 | `PI-01` | 20 |
| 6 | Deco W6000 (server room AP) | `DECO-SR` | Trunk (10/20/30/40/50) |
| 7–24 | Reserved for future expansion | — | — |

> Label each cable at both ends before connecting. Use a consistent naming
> convention (device abbreviation + index number).

---

## Step 4 — Deco W6000 Mesh Configuration

The two Deco W6000 units form a mesh network. One is wired (server room), one is wireless satellite (living room).

### Setup Steps

1. Download the TP-Link Deco app on a mobile device
2. Factory reset both units if previously configured
3. Connect the **server room Deco** to the GS724T (Port 6) via Ethernet
4. Follow the Deco app setup — designate the server room unit as the **main node**
5. Add the living room unit as a **satellite node** — it will connect wirelessly
6. Set both units to **Access Point mode** (disable Deco DHCP)
   - In the app: More → Advanced → Operation Mode → Access Point Mode
7. Configure a single SSID for seamless roaming across both APs
8. Record the management IP assigned by the UCG Ultra for both units

### SSID Planning

| SSID | Band | Purpose | Future VLAN |
|---|---|---|---|
| `homelab` | 2.4 / 5 GHz | Primary user network | 30 |
| `homelab-iot` | 2.4 GHz | IoT devices | 40 |
| `homelab-guest` | 2.4 / 5 GHz | Visitor access | 50 |

> Multiple SSIDs with VLAN tagging depend on Deco firmware support.
> Verify under Settings → Advanced → VLAN in the Deco app.
> See [Phase 5 — VLAN Configuration](phase-5-vlan-configuration.md) for details.

---

## Step 5 — Initial Connectivity Verification

After all devices are connected and powered on, verify the following before moving to Phase 4.

### Checklist

- [ ] All server room devices receive a DHCP IP from UCG Ultra
- [ ] UCG Ultra web UI is accessible from laptop
- [ ] GS724T web UI is accessible from laptop
- [ ] Unraid web UI is accessible at its assigned IP
- [ ] Proxmox Node 1 web UI accessible at port 8006 (e.g., `https://192.168.x.10:8006`)
- [ ] Proxmox Node 2 web UI accessible at port 8006
- [ ] Raspberry Pi responds to ping
- [ ] Deco app shows both units online and in AP mode
- [ ] Smart TV in living room has internet access
- [ ] Laptop connected to WiFi has internet access
- [ ] Ping from laptop to Unraid NAS succeeds
- [ ] Ping from laptop to both Proxmox nodes succeeds

### Commands to Run from Laptop

```bash
# Verify internet routing
ping 8.8.8.8

# Verify DNS resolution
nslookup google.com

# Verify LAN connectivity to servers
ping 192.168.x.20    # Unraid NAS
ping 192.168.x.10    # Proxmox Node 1
ping 192.168.x.11    # Proxmox Node 2
ping 192.168.x.30    # Raspberry Pi

# Check all hops to internet
traceroute 8.8.8.8
```

---

## Device IP Log

Record actual assigned IPs here after deployment. Replace placeholders with real values.

| Device | Hostname | MAC Address | IP Address | Notes |
|---|---|---|---|---|
| UCG Ultra | `ucg-ultra` | — | 192.168.x.1 | Default gateway |
| GS724T | `gs724t` | — | 192.168.x.2 | Switch management IP |
| Unraid NAS | `unraid` | — | 192.168.x.20 | Static DHCP lease |
| Proxmox Node 1 | `pve01` | — | 192.168.x.10 | Static DHCP lease |
| Proxmox Node 2 | `pve02` | — | 192.168.x.11 | Static DHCP lease |
| Raspberry Pi 5 | `rpi5` | — | 192.168.x.30 | Static DHCP lease |
| Deco W6000 (SR) | `deco-sr` | — | 192.168.x.4 | AP mode — management IP |
| Deco W6000 (LR) | `deco-lr` | — | 192.168.x.5 | AP mode — management IP |

> Assign static DHCP leases in the UCG Ultra for all infrastructure devices.
> This ensures IPs never change after a reboot.

---

## Physical Deployment Photos

> Add photos here as deployment progresses.

- [ ] `img/server-room-overview.jpg` — full rack/shelf view
- [ ] `img/switch-port-map.jpg` — labeled ports on GS724T
- [ ] `img/backbone-cable-run.jpg` — cable route between rooms
- [ ] `img/living-room-setup.jpg` — modem, router, Deco placement

---

## Next Phase

**[Phase 4 — Proxmox Setup](phase-4-proxmox-setup.md)** covers Proxmox installation, post-install configuration, storage setup, clustering, and initial VM deployment.
