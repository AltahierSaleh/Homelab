# 06 — Reverse Proxy (Nginx Proxy Manager)

Instead of remembering `10.0.0.20:8096`, you hit `https://jellyfin.<yourdomain>`. **Nginx Proxy Manager (NPM)** is a friendly web UI over nginx that handles hostnames, HTTPS certificates (Let's Encrypt), and routing — it's the single front door in front of every service.

**Runs on:** an LXC (1 core / 512MB RAM / 4GB disk), Docker + nesting — the standard [LXC baseline](../00-proxmox-host/#3--the-lxc-baseline-used-by-every-container-service).

---

## 1. Launch

```bash
sudo mkdir -p /opt/npm && cd /opt/npm
# copy docker-compose.yml here
docker compose up -d
```

Admin UI: `http://<LXC-IP>:81`.
**Default login:** `admin@example.com` / `changeme` — you're forced to change it immediately.

---

## 2. Point DNS at the proxy

You want every service hostname to resolve to the NPM box. Two layers:

- **Internal (LAN):** in Pi-hole → **Local DNS → DNS Records**, point each hostname at NPM's IP, e.g. `jellyfin.<yourdomain> → 10.0.0.2x`. Or use a single wildcard via **dnsmasq** if your Pi-hole supports it.
- **External (optional):** if you own a real domain, create DNS records at your registrar/Cloudflare. **How you expose the lab to the internet is your call** — common options:
  - **Cloudflare Tunnel** — no open ports; the tunnel connects outward. Safest.
  - **Port-forward 80/443** on the router to the NPM box + a dynamic-DNS hostname.
  - **VPN-only** (Tailscale/WireGuard) — nothing public; you reach NPM over the VPN.

Pick based on your risk tolerance. If anything is internet-facing, put the [auth layer](../07-auth/) in front of it.

---

## 3. Add a proxy host

For each service, in NPM → **Hosts → Proxy Hosts → Add Proxy Host**:

| Field | Example |
|---|---|
| Domain Names | `jellyfin.<yourdomain>` |
| Scheme | `http` |
| Forward Hostname / IP | `10.0.0.20` (the service's box) |
| Forward Port | `8096` |
| Block Common Exploits | ✅ |
| Websockets Support | ✅ (needed for Jellyfin, Grafana, etc.) |

Then the **SSL** tab:

- **SSL Certificate → Request a new certificate** (Let's Encrypt)
- ✅ Force SSL, ✅ HTTP/2
- Agree to the ToS, save.

> Let's Encrypt over HTTP-01 needs port 80 reachable from the internet. If you're **not** forwarding ports (Cloudflare Tunnel or VPN-only), use a **DNS-01 challenge** instead — NPM supports DNS providers like Cloudflare via an API token, which issues certs without any open ports.

Repeat for each service (Sonarr, Radarr, Grafana, Homarr, Jellyseerr…).

---

## 4. Result

```
Browser ──► https://jellyfin.<yourdomain>
              │  (valid HTTPS cert)
              ▼
        Nginx Proxy Manager  (10.0.0.2x)
              │  http://10.0.0.20:8096
              ▼
           Jellyfin
```

---

## Next

→ **[07-auth](../07-auth/)** — add a login wall (SSO) in front of anything you expose.
