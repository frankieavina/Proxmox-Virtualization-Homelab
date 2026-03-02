# Homelab Network Architecture — Part 2

## Overview

This section describes the homelab network architecture and physical layout used in this project.

The design focuses on building a small-scale enterprise-style network that supports virtualization, storage infrastructure, wireless connectivity, and future scalability.

The architecture is built around the Ubiquiti UCG Ultra router and Netgear GS724T core switch.

## Network Devices

### Core Routing and Firewall
- Ubiquiti UCG Ultra — Handles WAN routing, firewall policies, and QoS

### Switching Infrastructure
- Netgear GS724T — Core managed switch for server and distribution traffic

### Virtualization Platform
- Proxmox VE — Planned virtualization environment for homelab workloads

### Storage Server
- Unraid — Network-attached storage and Docker container hosting platform

## Simplified Network Architecture

```
                        INTERNET
                            │
                    ┌──────────────┐
                    │ Spectrum Modem │
                    └───────┬──────┘
                            │
                    ┌────────────────┐
                    │ UCG Ultra Router │
                    │ (Firewall + QoS) │
                    └───────┬────────┘
                            │
        ┌────────────────────┼────────────────────┐
        │                                         │
Living Room Access Layer                Server Infrastructure Backbone
(Managed Switch + AP + TV)              (Bedroom 2 Server Room)

        │                                         │
   Wi-Fi Users (Guest/Home)              ┌────────────────────┐
   Smart TV Traffic                      │ GS724T Core Switch  │
                                         ├──────────┬─────────┤
                                         │          │         │
                                  Unraid Server  Proxmox Nodes Print Server
```


## Device Zone Design

### Living Room Zone (Edge Access Layer)

Devices in this zone:

- One managed switch
- One Wi-Fi access point
- Smart TV (wired)

Purpose:
- Provide user connectivity
- Support guest Wi-Fi
- Reduce cable clutter

Traffic Flow:

```
Devices → Access Switch → Router → Internet
```

---

### Server Room Zone (Infrastructure Layer)

Devices located in Bedroom 2:

- Core managed switch
- Unraid storage server
- Virtualization nodes (future)
- Print server
- Optional access point

Primary software platforms:

- Proxmox VE virtualization platform
- Unraid NAS storage system

## Wireless Strategy

Two access points will be deployed:

- One in the living room
- One near the server room

Both APs operate in Access Point Mode.

Wireless gaming devices may connect via Wi-Fi if cabling is not desired.

## Future VLAN Strategy (Planned)

| VLAN | Purpose |
|---|---|
| 10 | Management Network |
| 20 | Server Infrastructure |
| 30 | User Devices |
| 40 | IoT / Gaming Devices |
| 50 | Guest Network |

## Physical Setup Steps

### Step 1 — ISP Connection
1. Connect coaxial cable → modem
2. Modem → router WAN port

### Step 2 — Backbone Ethernet Link

Run one long Ethernet cable:

```
UCG Ultra Router → Bedroom 2 Core Switch
```

This cable acts as the infrastructure backbone.

### Step 3 — Living Room Access Switch

Connect:

- Router → Living room managed switch
- Living room switch → TV
- Living room switch → Access Point

This zone supports guest Wi-Fi and entertainment devices.

### Step 4 — Server Room Deployment

In Bedroom 2:

- Backbone cable → GS724T core switch
- Core switch → storage server
- Core switch → virtualization nodes (future)
- Core switch → print server

## Architecture Philosophy

This homelab follows a simplified enterprise design model:

- Edge routing at the router
- Core switching in server zone
- Access switching for user devices
- Distributed wireless coverage

## Expansion Potential

This network can support:

- Virtual machine clustering
- Active Directory lab environments
- Infrastructure automation
- Monitoring dashboards
