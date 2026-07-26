# 05 — Homarr (Dashboard)

One page that links to everything in the lab, with live status tiles and widgets (media stack, downloads, system stats). It's the "home base" you actually bookmark.

**Runs on:** an LXC (1 core / 512MB RAM / 4GB disk), Docker + nesting — the standard [LXC baseline](../00-proxmox-host/#3--the-lxc-baseline-used-by-every-container-service).

---

## 1. Launch

```bash
sudo mkdir -p /opt/homarr && cd /opt/homarr
# copy docker-compose.yml here
export SECRET_ENCRYPTION_KEY=$(openssl rand -hex 32)   # or put it in an .env
echo "SECRET_ENCRYPTION_KEY=$SECRET_ENCRYPTION_KEY" > .env
docker compose up -d
```

Open `http://<LXC-IP>:7575` and create the admin account on first load.

---

## 2. Add your services

Homarr → **Edit mode** → add tiles. For each service, set the URL to its address on the LAN, e.g.:

| Tile | URL |
|---|---|
| Jellyfin | `http://10.0.0.20:8096` |
| Sonarr | `http://10.0.0.20:8989` |
| Radarr | `http://10.0.0.20:7878` |
| Prowlarr | `http://10.0.0.20:9696` |
| Jellyseerr | `http://10.0.0.20:5055` |
| Grafana | `http://10.0.0.2x:3000` |
| Pi-hole | `http://10.0.0.2/admin` |
| Proxmox | `https://10.0.0.10:8006` |

Many tiles support **integrations** — paste the service's API key and Homarr shows live data (queue counts, download speed, Pi-hole blocks, etc.).

> Once the [reverse proxy](../06-reverse-proxy/) is up, swap these raw `IP:port` URLs for clean hostnames like `https://jellyfin.<yourdomain>`.

---

## Next

→ **[06-reverse-proxy](../06-reverse-proxy/)** — friendly hostnames + HTTPS instead of `IP:port`.
