# 02 — NAS (OpenMediaVault)

Network storage for the lab. The 2TB SSD holds all downloads and media, and it's shared over **NFS** so the media-stack VM can mount it as if it were local disk. Keeping media off the Proxmox host means the host stays lean and the storage can grow independently.

**Runs on:** Raspberry Pi 5 (4GB) + a 2TB SSD (USB or M.2 hat).

---

## 1. Flash the Pi 5

The most reliable route on a Pi 5 is **OpenMediaVault installed on top of Raspberry Pi OS Lite**, using the official OMV install script.

1. **Raspberry Pi Imager** → **Raspberry Pi OS Lite (64-bit)** onto the Pi's boot media (SD card or NVMe).
2. In the Imager settings: hostname `nas`, enable SSH, set user/password, wired ethernet.
3. Boot, SSH in, update:
   ```bash
   ssh <user>@10.0.0.11
   sudo apt update && sudo apt -y full-upgrade
   sudo reboot
   ```
4. Give it a **static IP** — DHCP reservation on the router (bind MAC → `10.0.0.11`) is cleanest, same as the Pi-hole.

> Keep the OS on the SD/NVMe boot media and use the **2TB SSD purely for data**. Don't install the OS on the drive you're going to share.

---

## 2. Install OpenMediaVault

```bash
sudo wget -O - https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/install | sudo bash
```

This takes a while and reboots itself. When done, the web UI is at `http://10.0.0.11`.

**Default login:** `admin` / `openmediavault` — change the password immediately (**System → Users**, or the top-right menu).

> OMV wants to manage the network interface. If SSH drops after install, it's usually because OMV reconfigured `eth0` — reconnect on the same static IP.

---

## 3. Prepare the 2TB SSD

In the OMV web UI:

1. **Storage → Disks** — confirm the 2TB SSD shows up. **Wipe** it if it has old partitions (this erases it).
2. **Storage → File Systems → Create/+**:
   - Device: the 2TB SSD
   - Type: **EXT4**
   - Create, then **Mount** it.
3. Note the mount path (something like `/srv/dev-disk-by-uuid-XXXX/`).

---

## 4. Create shared folders

**Storage → Shared Folders → Create**. Make folders that match how the *arr* apps expect to see storage. A clean layout that avoids hardlink/permission pain later:

```
/data
├── torrents      (downloads land here)
│   ├── movies
│   ├── tv
│   └── music
└── media         (arr apps hardlink/move finished files here)
    ├── movies
    ├── tv
    └── music
```

Create at least a top-level **`data`** shared folder on the SSD; the subfolders can be made from the CLI or the file browser. Keeping downloads and media **under one parent** lets Sonarr/Radarr use fast hardlinks instead of slow copies.

---

## 5. Export it over NFS

1. **Services → NFS → Settings** → enable, save.
2. **Services → NFS → Shares → Create**:
   - Shared folder: `data`
   - Client: `10.0.0.0/24` (your LAN) — or lock to just the media VM's IP, e.g. `10.0.0.20`
   - Privilege: **Read/Write**
   - Extra options: `subtree_check,insecure,no_root_squash`
     - `no_root_squash` keeps container UIDs working; only do this on a trusted LAN.
3. Apply the pending config (top banner).

The export path will look like `10.0.0.11:/export/data`.

---

## 6. Mount it on the media VM

On the media-stack VM (see [03-media-stack](../03-media-stack/)):

```bash
sudo apt -y install nfs-common
sudo mkdir -p /mnt/data
# test mount
sudo mount -t nfs 10.0.0.11:/export/data /mnt/data
df -h | grep data
```

Make it permanent in `/etc/fstab`:

```fstab
10.0.0.11:/export/data  /mnt/data  nfs  defaults,_netdup,nofail,x-systemd.automount  0  0
```

> `nofail` + `x-systemd.automount` means the VM still boots if the NAS is briefly unavailable, and mounts on first access. If your version chokes on `_netdup`, use `_netdev` instead.

Now `/mnt/data` on the VM is the 2TB SSD. The compose file in the next folder points every container at this path.

---

## 7. Backups / snapshots (optional but wise)

- Add a second drive and use OMV's **rsync** jobs to mirror `/data` on a schedule.
- Or push critical config off-box. Media is re-downloadable; your *arr* databases and configs are the painful part to lose — back those up.

---

## Next

→ **[03-media-stack](../03-media-stack/)** — Jellyfin + the *arr* apps, all pulling from `/mnt/data`.
