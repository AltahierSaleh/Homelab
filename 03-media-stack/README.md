# 03 — Media Stack

The classic self-hosted media pipeline: **Jellyfin** streams, the ***arr* apps** (Sonarr, Radarr, Lidarr, Prowlarr, Bazarr) automate finding/organizing content, **Jellyseerr** takes requests, and everything torrent-related routes through **gluetun** (a VPN kill-switch container) so your download traffic never leaks.

**Runs on:** a dedicated **VM** (2 cores / 6GB RAM / 32GB disk). It's a VM rather than an LXC because the VPN tunnel and `/dev/net/tun` handling are cleaner with a real kernel. Media lives on the **NAS**, mounted at `/mnt/data` — not on the VM's disk.

> ⚠️ These files are a **clean reference**. Copy them in, then drop your own values into `.env`. Everything provider- or account-specific is a placeholder.

---

## Files here

| File | What it is |
|---|---|
| `docker-compose.yml` | The full stack. Commented so you can see why each choice was made. |
| `.env.example` | Copy to `.env` and fill in your PUID/PGID, timezone, paths, and VPN creds. |
| `.gitignore` | Keeps `.env` out of git. |

---

## 1. Build the VM

In Proxmox: **Create VM**, Debian 13, 2 cores / 6GB / 32GB, static IP (e.g. `10.0.0.20`), DNS → Pi-hole. Install Docker + the compose plugin (same steps as the [LXC baseline](../00-proxmox-host/#3--the-lxc-baseline-used-by-every-container-service), minus the nesting flags — a VM doesn't need them).

Mount the NAS at `/mnt/data` — see [02-nas-openmediavault §6](../02-nas-openmediavault/#6--mount-it-on-the-media-vm).

---

## 2. Configure and launch

```bash
# On the VM
sudo mkdir -p /opt/media && cd /opt/media
# copy docker-compose.yml and .env.example here, then:
cp .env.example .env
nano .env                     # fill in PUID/PGID, TZ, VPN creds

docker compose up -d
docker compose ps             # everything should be 'running'
```

### Confirm the VPN actually works (do this first!)

The download client rides inside gluetun. Verify the tunnel is up **before** you torrent anything:

```bash
# Should print the VPN's IP, NOT your home IP
docker exec gluetun wget -qO- https://ipinfo.io/ip
```

If gluetun is unhealthy, qbittorrent has **no internet at all** (that's the kill-switch working as intended). Check `docker logs gluetun`.

---

## 3. Wire the apps together

Order matters. Do it in this sequence:

1. **qbittorrent** — `http://<VM-IP>:8080`. Default login `admin` / temp password shown in `docker logs qbittorrent`. Set download path to `/data/torrents`.
2. **Prowlarr** — `http://<VM-IP>:9696`. Add your indexers here once; it syncs them to the *arr* apps.
   - In Prowlarr → **Settings → Apps**, add Sonarr/Radarr/Lidarr. Because they're on different networks, use the app's container IP or `<VM-IP>:<port>` and its API key.
3. **Sonarr** (`:8989`), **Radarr** (`:7878`), **Lidarr** (`:8686`):
   - **Download client:** point at qbittorrent. Since qbit lives inside gluetun, the host is the **gluetun/VM address** on port `8080` (not `localhost`).
   - **Root folder:** `/data/media/tv`, `/data/media/movies`, `/data/media/music` respectively.
4. **Bazarr** (`:6767`) — connect it to Sonarr + Radarr for subtitles.
5. **Jellyfin** (`:8096`) — add libraries pointing at `/data/media/*`. First-run wizard walks you through it.
6. **Jellyseerr** (`:5055`) — connect it to Jellyfin (for users) and to Sonarr/Radarr (to fulfill requests).

---

## 4. Why the `/data` layout matters

Every container mounts the NAS as `/data` (or a subfolder of it). Because downloads (`/data/torrents`) and the library (`/data/media`) share **one parent mount**, the *arr* apps can **hardlink** finished files instead of copying them — instant, and no double disk usage. If you instead mount `/downloads` and `/movies` as separate volumes, you lose hardlinks and every import becomes a slow full copy. This is the single most common self-hosting mistake — the layout above avoids it. (See [TRaSH Guides](https://trash-guides.info/) for the canonical explanation.)

---

## 5. Hardware transcoding (optional)

The mini-PC's Intel HD Graphics 630 can offload Jellyfin transcoding via QuickSync. Uncomment the `devices: - /dev/dri:/dev/dri` block in the Jellyfin service, pass the GPU through to the VM in Proxmox, then enable **VAAPI/QuickSync** under Jellyfin → Playback → Transcoding.

---

## 6. Maintenance

```bash
docker compose pull && docker compose up -d   # update all images
docker compose logs -f <service>              # tail a service
```

Back up `/opt/appdata` (the `CONFIG_ROOT`) regularly — that's where all your *arr* databases and settings live. Media is re-downloadable; that config is not.

---

## Next

→ **[04-monitoring](../04-monitoring/)** — watch CPU/RAM/disk across the lab.
