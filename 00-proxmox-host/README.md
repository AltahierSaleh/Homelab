# 00 — Proxmox Host

The foundation of the whole lab. One mini-PC runs **Proxmox VE**, and every service (except DNS and NAS, which live on the Raspberry Pis) runs as a VM or LXC container on top of it.

## Hardware

- **Mini-PC:** 4-core Intel i5, 16GB RAM, Intel HD Graphics 630 (handy for Jellyfin hardware transcoding later)
- **Hostname:** `taherProxmox`
- **Network:** wired to the router, gateway `10.0.0.1`, `/24` subnet

Any small x86 box with virtualization support (Intel VT-x / AMD-V) works. 16GB RAM is the practical floor for this many services; 32GB gives real breathing room.

---

## 1. Install Proxmox VE

1. Download the **Proxmox VE ISO** from <https://www.proxmox.com/en/downloads>.
2. Flash it to a USB stick with [balenaEtcher](https://etcher.balena.io/) or `dd`:
   ```bash
   # Linux/macOS — double check the disk path first!
   sudo dd if=proxmox-ve_*.iso of=/dev/sdX bs=4M status=progress conv=fsync
   ```
3. Boot the mini-PC from USB, choose **Install Proxmox VE (Graphical)**.
4. During setup:
   - **Target disk:** the mini-PC's internal drive.
   - **Country / timezone / keyboard:** your own.
   - **Password + email:** set a strong root password.
   - **Management network:**
     - Hostname (FQDN): `taherProxmox.local` (or `<hostname>.<domain>`)
     - IP: a **static** address like `10.0.0.10/24`
     - Gateway: `10.0.0.1`
     - DNS: `10.0.0.1` (the Pi-hole, once it exists — `1.1.1.1` is fine during initial setup)
5. Reboot, pull the USB.

Access the web UI at `https://<PROXMOX_IP>:8006` (accept the self-signed cert warning). Log in as `root` with the password you set.

---

## 2. Post-install housekeeping

Proxmox nags about the enterprise repo unless you have a subscription. Switch to the free (no-subscription) repo. In the web UI shell (or SSH as root):

```bash
# Disable the enterprise repos
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list 2>/dev/null
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/ceph.list 2>/dev/null

# Add the no-subscription repo (adjust 'trixie' to your Debian base if different)
echo "deb http://download.proxmox.com/debian/pve trixie pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

apt update && apt -y dist-upgrade
```

> A popular community helper for this and much more is the Proxmox VE Helper-Scripts project (<https://community-scripts.github.io/ProxmoxVE/>). Read scripts before running them.

---

## 3. The LXC baseline (used by every container service)

Most services here run in **LXC containers**, not full VMs — they're lighter and boot instantly. The catch: to run Docker *inside* an LXC, the container must be **unprivileged with nesting + keyctl enabled**.

### Get a container template

**Datacenter → Storage → local → CT Templates → Templates**, then download **debian-13-standard**. (Or via CLI: `pveam update && pveam available | grep debian-13` then `pveam download local <template>`.)

### Create the container (web UI)

**Create CT** (top right), then:

| Tab | Setting | Value |
|---|---|---|
| General | Hostname | e.g. `monitoring`, `homarr`, `npm`, `auth` |
| General | **Unprivileged container** | ✅ checked |
| General | Password / SSH key | set one |
| Template | Template | `debian-13-standard` |
| Disk | Size | per service (see root README table) |
| CPU | Cores | per service |
| Memory | RAM / Swap | per service |
| Network | IPv4 | **Static**, e.g. `10.0.0.21/24` |
| Network | Gateway | `10.0.0.1` |
| DNS | DNS server | `10.0.0.1` (Pi-hole) |

**Don't start it yet.**

### Enable nesting + keyctl (required for Docker-in-LXC)

Two options:

**Web UI:** select the container → **Options → Features → Edit** → tick **Nesting** and **keyctl**.

**CLI (faster):**
```bash
# Replace 101 with your container's ID
pct set 101 --features nesting=1,keyctl=1
```

Now start the container.

### Install Docker inside the LXC

Open the container console (**use noVNC** — the xterm.js console is flaky on this hardware) or SSH in, then:

```bash
apt update && apt -y install ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/debian $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
  > /etc/apt/sources.list.d/docker.list
apt update
apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

docker run --rm hello-world   # sanity check
```

That's the reusable recipe. Every LXC-based service in this repo (monitoring, Homarr, reverse proxy, auth) starts from exactly this baseline, then just drops a `docker-compose.yml` on top.

---

## 4. VMs vs LXCs — when to use which

- **LXC** for anything simple and Linux-native (dashboards, proxies, monitoring). Lower overhead, instant boot.
- **VM** when you want full isolation or a real kernel — used here for the **media stack**, because the VPN container (gluetun) and device/DNS handling are cleaner in a proper VM. See [03-media-stack](../03-media-stack/).

---

## Next

→ **[01-networking-pihole](../01-networking-pihole/)** — set up DNS and lock in your static-IP plan before building more boxes.
