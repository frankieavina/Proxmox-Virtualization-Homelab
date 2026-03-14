# Phase 2 — Network Architecture & Layout

## Overview

This phase documents the full network topology, device placement, physical layout, and design decisions for the homelab. The goal is to simulate a small enterprise-style network across a real home environment using consumer and prosumer hardware.

---

## Physical Layout

The home is divided into three network zones based on physical location and function.

| Zone | Location | Purpose |
|---|---|---|
| Edge / Living Room | Living room | ISP modem, router, WiFi coverage |
| Server Room | Bedroom 2 | Core switching, all servers, wired infrastructure |
| User Zone | User bedroom | Personal devices, gaming |

### Why This Layout?

The ISP modem must stay in the living room (ISP requirement). All servers are consolidated in Bedroom 2 to centralize cabling, noise, and heat. A single Ethernet backbone run connects the two rooms, keeping the physical footprint minimal while maintaining clean network segmentation.

---

## Hardware Inventory

### Network Infrastructure

| Device | Model | Role | Location |
|---|---|---|---|
| Router / Firewall | Ubiquiti UCG Ultra | Edge routing, firewall, DHCP, QoS | Living room |
| Core Switch | Netgear GS724T (managed) | Central server room switching, 802.1Q trunk | Bedroom 2 |
| WiFi AP — Living Room | TP-Link Deco W6000 | Wireless coverage, 2× wired ports (Smart TV) | Living room |
| WiFi AP — Server Room | TP-Link Deco W6000 | Wireless coverage, wired uplink to GS724T | Bedroom 2 |

### Servers

| Device | Model | CPU | RAM | Storage | Role |
|---|---|---|---|---|---|
| NAS | Custom (Supermicro X11SCA-F) | Intel i5-8500T | 32 GB | 2× 6TB SAS via Adaptec ASR-7805 HBA | Unraid NAS, Docker containers, VMs |
| Proxmox Node 1 | Lenovo ThinkCentre M720q | Intel i5-9400T | 8 GB | Internal SSD | Primary virtualization node |
| Proxmox Node 2 | Lenovo ThinkCentre M710q | Intel i5-7500T | 8 GB | Internal SSD | Secondary virtualization node |
| Print Server | Raspberry Pi 5 | — | — | — | CUPS print server |

### End User & IoT Devices

| Device | Connection | Zone |
|---|---|---|
| Smart TV (living room) | Wired via Deco W6000 eth port | Living room |
| Smart TV (room 3) | WiFi | IoT / VLAN 40 |
| PS5 | WiFi | User bedroom |
| Laptop / Personal PC | WiFi | User bedroom |
| Parents' phones (×2) | WiFi | User bedroom |
| IP Cameras ×4 (doorbell, garage, backyard, side) | WiFi | IoT |
| Qolsys IQ Panel (security alarm) | WiFi | IoT |
| Smart thermostat | WiFi | IoT |
| Amazon Firesticks ×3 | WiFi | IoT |
| 3D printer | WiFi | Bedroom 2 / IoT |

---

## Network Topology

### Traffic Flow (East → West)

```
Internet (Spectrum ~110/11 Mbps)
  └── ISP Modem (living room)
        └── Ubiquiti UCG Ultra (router/firewall)
              └── [Single Ethernet backbone run] ──────────────────────────┐
                                                                            │
                                                              Netgear GS724T (core switch)
                                                              ├── Unraid NAS
                                                              ├── Proxmox Node 1 (M720q)
                                                              ├── Proxmox Node 2 (M710q)
                                                              ├── Raspberry Pi 5 (print server)
                                                              └── Deco W6000 (server room AP, wired uplink)
                                                                    └── [Wireless mesh backhaul]
                                                                          └── Deco W6000 (living room AP)
                                                                                ├── Smart TV (wired eth port)
                                                                                └── WiFi clients
                                                                                      ├── PS5
                                                                                      ├── Laptop / PC
                                                                                      ├── Phones ×2
                                                                                      ├── Cameras ×4
                                                                                      ├── Firesticks ×3
                                                                                      └── IoT devices
```

### Key Design Decisions

**Single backbone Ethernet run**
Only one cable crosses between rooms (living room → bedroom 2). This minimizes physical cable runs while maintaining a clean, manageable infrastructure. All inter-room traffic flows through this single trunk.

**Deco mesh in AP mode with wired backhaul on server room side**
The Deco W6000 in Bedroom 2 is wired directly into the GS724T, making it the primary mesh node. The living room Deco connects back to it wirelessly. This means all WiFi traffic — from the living room and throughout the home — routes through the server room switch before reaching the router. This is intentional and enables future VLAN enforcement at the switch level.

**No switch in the living room**
The Deco W6000's two built-in Ethernet ports handle the only wired device in the living room (Smart TV). An additional switch is unnecessary and was intentionally omitted to keep the living room clean.

**UCG Ultra as the single routing and firewall layer**
All inter-VLAN routing, firewall rules, QoS, and DHCP are handled by the UCG Ultra. This mirrors an enterprise edge device pattern and centralizes policy management.

---

## Planned VLAN Design (Phase 5)

VLANs have not been implemented yet. The following is the planned segmentation.

| VLAN | Name | Subnet | Devices |
|---|---|---|---|
| 10 | Management | 192.168.10.0/24 | UCG Ultra, GS724T, Deco APs |
| 20 | Servers | 192.168.20.0/24 | Unraid, Proxmox ×2, Raspberry Pi |
| 30 | Users | 192.168.30.0/24 | Laptop, PC, parents' phones |
| 40 | IoT | 192.168.40.0/24 | Cameras, alarm, thermostat, Firesticks, TVs, PS5, 3D printer |
| 50 | Guest | 192.168.50.0/24 | Visitor WiFi — internet only |

> VLAN implementation is covered in [Phase 5 — VLAN Configuration](phase-5-vlan-configuration.md).

---

## Always-On vs On-Demand Devices

This homelab is designed as an **on-demand lab environment**. Not all devices run 24/7 in order to manage power consumption.

| Device | Power Profile | Reason |
|---|---|---|
| Netgear GS724T | Always on | Core network infrastructure |
| Raspberry Pi 5 | Always on | Print server — low power draw |
| Ubiquiti UCG Ultra | Always on | Router — required for internet |
| Deco W6000 ×2 | Always on | WiFi — required for daily use |
| Unraid NAS | On demand | Powered on for lab work only |
| Proxmox Node 1 | On demand | Powered on for lab work only |
| Proxmox Node 2 | On demand | Powered on for lab work only |

---

## ISP Connection Details

| Parameter | Value |
|---|---|
| Provider | Spectrum |
| Download speed | ~110 Mbps |
| Upload speed | ~11 Mbps |
| Latency | ~20 ms |
| Modem | ISP provided — fixed location |

---

## Network Diagram
Full visual topology



---

## Next Phase

**[Phase 3 — Physical Deployment](phase-3-physical-deployment.md)** covers the step-by-step physical setup, cable runs, port mapping, and device verification.
