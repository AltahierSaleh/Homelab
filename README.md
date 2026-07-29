# Homelab

A complete, reproducible homelab built on a single Proxmox mini-PC, backed by two Raspberry Pis for DNS and storage. This repo documents **exactly** how it was built, service by service, so you can rebuild the same thing (or cherry-pick the parts you want).

Every guide is written to be **copy-paste friendly**. Anywhere you see a `PLACEHOLDER` or a value in `<angle brackets>`, swap in your own. Nothing here assumes secrets — API keys, passwords, VPN creds, and domains are all left as placeholders for you to fill in.
all ip addresses used are placeholder ip's that you should switch with whatever corresponds with you machines and router.

---

## Parts list (what I used)

The exact gear this lab was built with. None of it is required to follow the guides — any equivalents work — but this is the actual bill of materials.

### Compute

| Part | Role |
|---|---|
| Intel i5 mini-PC (4 cores / 16GB RAM, HD Graphics 630) | Proxmox host — runs all VMs and LXCs |
| Raspberry Pi 3 B+ | Pi-hole (network DNS + ad blocking) |
| Raspberry Pi 5 (4GB) + 2TB SSD | OpenMediaVault NAS (media + downloads storage) |

### Networking, rack & power

| Part | Role |
|---|---|
| **NETGEAR GS308** — 8-Port Gigabit Network Switch | Wired backbone for the host, both Pis, and uplink to the router. Unmanaged, silent, plug-and-play. |
| **GeeekPi DeskPi RackMate 4U Cabinet** (10", DeskPi RackMate T0) | The mini 10-inch rack that houses everything. |
| **GeeekPi DeskPi RackMate 1U Shelf** (with RJ45 CAT6 + Mini HDMI passthrough) | Rack-mount shelf for the mini-PC and Pis inside the cabinet. |
| **ANVODE Recessed Desk Power Socket** (2× USB-A, 2× USB-C, 3 outlets, 2m lead) | Hidden in-desk power feed for the rack and peripherals. |

<img width="2992" height="2992" alt="20260726_154245" src="https://github.com/user-attachments/assets/769a3561-6015-4452-8fc3-cdd1a2f609fe" />

<img width="2992" height="2992" alt="20260726_154339" src="https://github.com/user-attachments/assets/8e6d9fd5-4973-4753-a479-14560586b792" />

> Cabling: standard Cat5e/Cat6 patch leads between the switch, the rack shelf's RJ45 port, and each device.

---

## What's in this lab

| Layer | Hardware | Runs |
|---|---|---|
| **Hypervisor** | Intel i5 mini-PC, 4 cores / 16GB RAM | Proxmox VE (`whatever-you-want`) |
| **DNS** | Raspberry Pi 3 B+ | Pi-hole (network-wide DNS + ad blocking) |
| **Storage** | Raspberry Pi 5 (4GB) + 2TB SSD | OpenMediaVault (NAS) |

Everything else runs as **VMs and LXC containers on the Proxmox host**:

| Service | Type | Cores | RAM | Disk | Purpose |
|---|---|---|---|---|---|
| Media stack | VM | 2 | 6GB | 32GB | Jellyfin, Sonarr, Radarr, Prowlarr, Lidarr, Bazarr, Jellyseerr, gluetun (VPN) |
| Monitoring | LXC | 1 | 1.5GB | 16GB | Grafana, Prometheus, node_exporter, cAdvisor |
| Homarr | LXC | 1 | 512MB | 4GB | Dashboard / homepage |
| Reverse proxy | LXC | 1 | 512MB | 4GB | Nginx Proxy Manager (front door) |
| Auth | LXC | 1 | 1GB | 8GB | Authentik or Authelia (SSO in front of exposed services) |

**Total allocated:** ~9.5GB RAM, leaving ~6.5GB for Proxmox and headroom. Cores are oversubscribed on paper — normal and fine for lightweight LXCs.

---

## Network map

```
                          Internet
                             │
                        ┌────┴────┐
                        │ Router  │  10.0.0.1  (gateway, /24 subnet)
                        └────┬────┘
                             │  LAN 10.0.0.0/24
        ┌────────────────────┼────────────────────────┐
        │                    │                        │
  ┌─────┴─────┐       ┌──────┴───────┐          ┌─────┴────────┐
  │ Pi 3 B+   │       │ Proxmox host │          │ Pi 5 + 2TB   │
  │ Pi-hole   │       │ proxmox      │          │OpenMediaVault│
  │ (DNS)     │       │              │          │ (NAS / NFS)  │
  └───────────┘       │  VMs + LXCs: │          └──────┬───────┘
                      │  - media VM  │─────NFS mount───┘
                      │  - monitoring│
                      │  - homarr    │
                      │  - npm proxy │
                      │  - auth      │
                      └──────────────┘

DNS: every device points at the Pi-hole; Pi-hole forwards to 1.1.1.1 as fallback.
```

---

## Build order

Follow the folders in order — each one assumes the previous layers exist.

1. **[00-proxmox-host](./00-proxmox-host/)** — Install the hypervisor. Everything runs on it.
2. **[01-networking-pihole](./01-networking-pihole/)** — DNS + static IP plan. Do this early so every later box gets a clean name and a stable address.
3. **[02-nas-openmediavault](./02-nas-openmediavault/)** — Storage. The media stack mounts this over the network.
4. **[03-media-stack](./03-media-stack/)** — Jellyfin + the *arr* apps behind a VPN. Needs the NAS.
5. **[04-monitoring](./04-monitoring/)** — Grafana + Prometheus. Optional but recommended early so you can watch resource use as you add services.
6. **[05-homarr](./05-homarr/)** — A single dashboard linking everything.
7. **[06-reverse-proxy](./06-reverse-proxy/)** — Nginx Proxy Manager: clean hostnames + HTTPS.
8. **[07-auth](./07-auth/)** — SSO layer in front of anything you expose.

---

## Conventions used throughout

- **Static IPs** live in `10.0.0.x`, gateway `10.0.0.1`. Pick a scheme and stick to it (this lab reserves a block per service).
- **DNS** on every host points at the Pi-hole first, `1.1.1.1` as fallback.
- **LXCs** are Debian 13, **unprivileged**, with `nesting=1` and `keyctl=1` enabled — required to run Docker inside them.
- **Docker + the compose plugin** are installed inside every LXC/VM that runs containers.
- **noVNC** is the reliable console in Proxmox for this hardware; the default xterm.js console has connection issues here.

---

## Disclaimer

This is a personal lab. Values like retention, RAM allocations, and image tags are what worked on this hardware — treat them as a starting point, not gospel. Read each guide before pasting, and never commit real secrets to a public repo (use `.env` files kept out of git).
