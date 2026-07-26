# 01 — Networking + Pi-hole DNS

Do this early. Once DNS and a static-IP plan are in place, every box you build afterward gets a stable address and a clean name, and the whole network gets ad/tracker blocking for free.

**Runs on:** Raspberry Pi 3 B+ (a low-power always-on box — perfect for DNS).

---

## 1. Plan your addressing

The lab uses a single flat `/24`: `10.0.0.0/24`, gateway `10.0.0.1`. Reserve a predictable block so you never have to guess where something lives:

| Range | Use |
|---|---|
| `10.0.0.1` | Router / gateway |
| `10.0.0.2` | Pi-hole (this box) |
| `10.0.0.10` | Proxmox host |
| `10.0.0.11` | NAS (OpenMediaVault) |
| `10.0.0.20–29` | LXC/VM services (monitoring, homarr, npm, auth, media…) |
| `10.0.0.100+` | DHCP pool for clients |

Adjust to your own subnet, but **keep servers out of the DHCP pool** so they never collide with a phone or laptop.

---

## 2. Flash Raspberry Pi OS to the Pi 3

1. Install **Raspberry Pi Imager** (<https://www.raspberrypi.com/software/>).
2. Choose **Raspberry Pi OS Lite (64-bit)** — no desktop needed.
3. Click the gear / **Edit Settings** before writing:
   - Set hostname: `pihole`
   - Enable **SSH** (password or key)
   - Set username/password
   - Configure Wi-Fi only if you must — **wired ethernet is strongly preferred** for a DNS server.
4. Write to the SD card, boot the Pi, SSH in:
   ```bash
   ssh <user>@10.0.0.2   # or find it via your router first
   sudo apt update && sudo apt -y full-upgrade
   ```

---

## 3. Give the Pi a static IP

Easiest and most reliable: **DHCP reservation on your router** — bind the Pi's MAC to `10.0.0.2`. Do that in the router UI.

If you'd rather set it on the Pi (Bookworm uses NetworkManager):

```bash
# Find the connection name
nmcli connection show
# Set a static IP (replace 'Wired connection 1' with yours)
sudo nmcli connection modify "Wired connection 1" \
  ipv4.addresses 10.0.0.2/24 \
  ipv4.gateway 10.0.0.1 \
  ipv4.dns "1.1.1.1" \
  ipv4.method manual
sudo nmcli connection up "Wired connection 1"
```

---

## 4. Install Pi-hole

```bash
curl -sSL https://install.pi-hole.net | bash
```

The installer is a guided TUI. Choices that matter:

- **Upstream DNS provider:** Cloudflare (`1.1.1.1`) — this is your fallback resolver.
- **Blocklists:** accept the default (StevenBlack) list to start.
- **Static IP:** confirm `10.0.0.2` when prompted.
- At the end it prints the **admin web password** — save it. (Reset later with `pihole setpassword`.)

Admin UI: `http://10.0.0.2/admin`.

---

## 5. Point the network at Pi-hole

Two ways — pick one:

**A. Router-wide (recommended).** In the router's DHCP settings, set the **primary DNS server** to `10.0.0.2`. Now every device that gets a lease uses Pi-hole automatically. Leave secondary DNS blank or also `10.0.0.2` — if you set a public secondary (like `8.8.8.8`), devices will sometimes bypass Pi-hole.

**B. Per-host.** Set DNS to `10.0.0.2` on each server individually (this is what the Proxmox host and each LXC do — DNS = `10.0.0.1` if the router forwards, or `10.0.0.2` directly).

> Fallback: the lab keeps `1.1.1.1` as Pi-hole's *upstream* so name resolution still works if a blocklist misbehaves — but the clients themselves point only at Pi-hole.

---

## 6. Optional niceties

- **Local DNS records:** Pi-hole → **Local DNS → DNS Records** to give services friendly names, e.g. `jellyfin.home → 10.0.0.20`. Great before you add the reverse proxy.
- **Custom blocklists:** add more from <https://firebog.net/> if you want stricter filtering.
- **Keep it updated:** `pihole -up` occasionally.

---

## Verify

From any client:
```bash
nslookup doubleclick.net    # should resolve to 0.0.0.0 (blocked)
nslookup google.com         # should resolve normally
```
Check the **Query Log** in the admin UI to confirm devices are actually asking Pi-hole.

---

## Next

→ **[02-nas-openmediavault](../02-nas-openmediavault/)** — stand up network storage for the media stack.
