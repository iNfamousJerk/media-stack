# Auto-Ripping Machine Plan

> **Status:** Draft / Proposed  
> **Author:** Hermes  
> **Date:** 2026-05-22

---

## Overview

Plug in a USB Blu-ray drive, insert a disc, and have it automatically rip, organize, and land in Jellyfin — zero manual steps. Ripped files are also covered by the nightly PBS backup.

---

## What You'll Need

| Item | Est. Cost | Notes |
|------|-----------|-------|
| USB Blu-ray drive | $60-100 | LG BP60NB10, Verbatim 43888, or similar |
| Blank disc space | Free | Already part of your /data/media |
| Dedicated CT | Free | 2 GB RAM, 2 cores, 16 GB root |

**For 4K UHD Blu-rays:** Make sure the drive is UHD-friendly (LG WH14NS40 or BU40N in a USB enclosure, then flashed with LibreDrive firmware). Most ~$60 drives won't do UHD out of the box.

---

## Software: ARM (Automatic Ripping Machine)

ARM is the gold standard — designed specifically for this. It:
- Monitors for disc insertion (udev + polling)
- Rips with MakeMKV (lossless, decrypted)
- Optionally transcodes with HandBrakeCLI (H.265 saves ~50% space)
- Renames and organizes files
- Has a web UI for monitoring
- Can output directly to your Jellyfin media folders

---

## Architecture

```
┌─ Ripper CT (CT 110 suggested) ─────────────────────────┐
│                                                          │
│  USB Blu-ray ──── USB passthrough ──── CT               │
│       │                                  │              │
│       ▼                                  ▼              │
│   udev detects    ──→  ARM Docker container             │
│   disc inserted         │                               │
│                         ├── MakeMKV → .mkv              │
│                         ├── (optional) HandBrake → H.265 │
│                         └── Output to /data/media/      │
│                                              │          │
└──────────────────────────────────────────────┼──────────┘
                                               ▼
                                   Jellyfin picks it up
                                               │
                                               ▼
                                   Nightly rsync → PBS backup
```

---

## Implementation Steps

### Step 1: Create the Ripper CT

On the Proxmox host, create a new CT:
- **CT ID:** 110 (or next available)
- **OS Template:** Debian 12 / Ubuntu 24.04 LXC template
- **Resources:** 2 cores, 2 GB RAM, 16 GB root disk
- **Privileged:** Yes (required for optical drive access)
- **Network:** Static IP on your homelab subnet (e.g., 10.2.7.110)

### Step 2: Pass Through the USB Blu-ray Drive

**On the Proxmox host**, find the USB device:

```bash
lsusb
# Example output:
# Bus 003 Device 002: ID 2109:0715 VIA Labs, Inc. External Blu-ray Drive
```

Then **either** edit `/etc/pve/lxc/110.conf` directly:

```
lxc.cgroup2.devices.allow: c 11:* rwm
lxc.mount.entry: /dev/sr0 dev/sr0 none bind,optional,create=file
lxc.mount.entry: /dev/sg0 dev/sg0 none bind,optional,create=file
```

**Or** use the Proxmox web UI:
1. Select CT 110 → Resources → Add → Device Passthrough
2. Select `/dev/sr0` and `/dev/sg0`

Reboot the CT and verify the drive is visible:

```bash
lsblk | grep sr0    # should show the optical drive
sudo apt install -y lsdvd
lsdvd               # should show disc info (insert any disc first)
```

### Step 3: Mount /data/media Inside the Ripper CT

ARM needs to write ripped files to your media directory. Choose one method:

**Option A: NFS mount (cleanest)**

On the ripper CT:
```bash
# Install NFS client
apt install nfs-common

# Add to /etc/fstab for auto-mount
mkdir -p /data/media
echo "<media-ct-ip>:/data/media /data/media nfs defaults,soft,intr,rsize=8192,wsize=8192 0 0" >> /etc/fstab
mount -a
```

**Option B: SSHFS (no config on media CT side)**

On the ripper CT:
```bash
apt install sshfs
sshfs <user>@<media-ct-ip>:/data/media /data/media
```

### Step 4: Deploy ARM via Docker

Create `/opt/arm/docker-compose.yml`:

```yaml
version: "3.9"

services:
  arm:
    image: ghcr.io/automatic-ripping-machine/automatic-ripping-machine:latest
    container_name: arm
    privileged: true
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - ./config:/etc/arm
      - ./logs:/home/arm/logs
      - /data/media:/home/arm/media
      - /dev:/dev:ro
    devices:
      - /dev/sr0:/dev/sr0
      - /dev/sg0:/dev/sg0
    ports:
      - "8085:8080"       # ARM web UI
    restart: unless-stopped
```

Start it:

```bash
cd /opt/arm
docker compose up -d
```

### Step 5: Configure ARM

Access the web UI at `http://<ripper-ct-ip>:8085` and configure:

**General Settings:**
- Post-rip action: Move to media directory
- Output directory: `/home/arm/media/movies`

**Ripper Settings:**
- Ripper: **MakeMKV** (best quality, lossless)
- Minimum title length: 600 seconds (skip short features/extras)

**Transcoder Settings (Optional):**
- Enable HandBrake if you want H.265 compression (saves ~50% space)
- Preset: `H.265 NVENC` (if you have an Nvidia GPU) or `H.265 MKV` (CPU)
- Skip for 4K discs (transcoding 4K HDR is very slow on CPU)

**Naming:**
- Pattern: `{title} ({year})/{title} ({year})-{source}`

**Notification (Optional):**
- Can send to Discord when a rip completes

### Step 6: Test the Pipeline

1. Insert a test disc (any DVD or Blu-ray)
2. Watch the ARM web UI — should show "Disc detected" → "Ripping" → "Transcoding" → "Moving"
3. Check `/data/media/movies/` for the output
4. Verify Jellyfin picks it up (scan library or wait for auto-refresh)
5. Run the backup script to confirm the new movie gets rsynced to PBS

---

## End-to-End Workflow

```
1. Walk up to the server, insert a Blu-ray     (~10 seconds)
2. ARM detects the disc                         (~30 seconds)
3. MakeMKV rips the main feature                (~15-30 min)
4. (Optional) HandBrake transcodes to H.265      (~30-60 min)
5. Movie lands in /data/media/movies/Movie (2024)/ (instant)
6. Jellyfin picks it up                          (~1-5 min)
7. Nightly PBS backup covers it automatically    (next 2 AM)
```

---

## Multi-Disc Workflow (TV Series / Box Sets)

ARM handles multiple discs one at a time:
1. Rip disc 1 → output to /data/media/tv/Show Name/Season 01/
2. Eject disc 1, insert disc 2 → ARM auto-detects
3. Continue until the set is done

For TV series, configure ARM to output to the TV directory instead of movies:

```
Output: /home/arm/media/tv/{title}/Season {season}/{title} - S{season}E{episode}
```

---

## Tips & Pitfalls

| Issue | Fix |
|-------|-----|
| "No disc detected" after inserting | Run `lsblk` to check if the CT sees `/dev/sr0`. Restart the ARM container. |
| MakeMKV beta key expired | ARM auto-updates the MakeMKV beta key from their forum. If not, manually update in ARM settings. |
| UHD Blu-ray won't rip | Drive needs LibreDrive firmware. Check MakeMKV forums for compatible drives. |
| Rips are very slow | MakeMKV rips at disc read speed (2x-6x for Blu-ray). This is normal. |
| ARM web UI won't load | Check `docker compose logs -f` for errors. Port `8085` may be in use. |
| Jellyfin not seeing new movies | Trigger a manual library scan or wait for auto-refresh (default every 15 min). |
| Disc ejects mid-rip | Dirty or damaged disc. Clean it with a microfiber cloth. |

---

## What You'll Have When Done

```
CT 110 (Ripper) ── USB Blu-ray Drive
  │
  └─ ARM Docker Container
       │
       └─ /data/media/movies/Movie Name (2024)/
            ├── Movie Name (2024).mkv
            └── Movie Name (2024).nfo     (metadata from Radarr)
                                  │
                                  ▼
                         Jellyfin library
                                  │
                                  ▼
                         Nightly PBS backup
```

---

## Future Upgrades

| Upgrade | When | Benefit |
|---------|------|---------|
| Add a GPU to the ripper CT | After getting the stack running | Hardware H.265 encoding (faster transcoding) |
| Set up multi-drive auto-ripping | If you rip a lot | Rip 2-4 discs simultaneously |
| Integrate with Radarr | After ARM is stable | Auto-import, rename, metadata download |
| Add a disc slot-loading drive | For a cleaner rack look | No tray to stick out |
