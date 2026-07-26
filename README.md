# Homelab

A complete, reproducible homelab built on a single Proxmox mini-PC, backed by two Raspberry Pis for DNS and storage. This repo documents **exactly** how it was built, service by service, so you can rebuild the same thing (or cherry-pick the parts you want).

Every guide is written to be **copy-paste friendly**. Anywhere you see a `PLACEHOLDER` or a value in `<angle brackets>`, swap in your own. Nothing here assumes secrets — API keys, passwords, VPN creds, and domains are all left as placeholders for you to fill in.

---

## What's in my homelab

| Layer | Hardware | Runs |
|---|---|---|
| **Hypervisor** | Intel i5 mini-PC, 4 cores / 16GB RAM | Proxmox VE (`taherProxmox`) |
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
        │                    │                         │
  ┌─────┴─────┐       ┌──────┴───────┐          ┌──────┴───────┐
  │ Pi 3 B+   │       │ Proxmox host │          │ Pi 5 + 2TB   │
  │ Pi-hole   │       │ taherProxmox │          │ OpenMediaVault│
  │ (DNS)     │       │              │          │ (NAS / NFS)  │
  └───────────┘       │  VMs + LXCs: │          └──────┬───────┘
                      │  - media VM ─┼─────NFS mount───┘
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
