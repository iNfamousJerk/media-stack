# VPN + Backup + Auto-Ripping Architecture Plan

> **Status:** Draft / Proposed  
> **Author:** Hermes  
> **Date:** 2026-05-22  
> **Updated:** 2026-05-22 — Added auto-ripping machine, revised PBS for same-host + existing HDDs

---

## Part 1: NordVPN + Media Stack Integration

### Goal

Route all torrent traffic through NordVPN while keeping the *arrs and Jellyfin on the normal network. qBittorrent must lose internet access if the VPN drops (kill switch).

### Architecture

```
┌──────────────────────────────────────────────────────────┐
│                      Docker Host                          │
│                                                           │
│  ┌──────────────────────┐     ┌──────────────────────┐   │
│  │    gluetun (VPN)     │     │    media-stack net    │   │
│  │  ┌────────────────┐  │     │  ┌────────────────┐   │   │
│  │  │ NordVPN WG     │  │     │  │ radarr        │   │   │
│  │  │ (network_mode) │──│─────│  │ sonarr         │   │   │
│  │  └────────────────┘  │     │  │ prowlarr       │   │   │
│  │                      │     │  │ lidarr         │   │   │
│  │  ┌────────────────┐  │     │  │ bazarr         │   │   │
│  │  │ qbittorrent    │  │     │  │ jellyfin       │   │   │
│  │  │ (inside VPN)   │  │     │  │ flaresolverr   │   │   │
│  │  └────────────────┘  │     │  │ cross-seed     │   │   │
│  └──────────────────────┘     │  └────────────────┘   │   │
│                               └──────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| VPN client | **Gluetun** | Most maintained Docker VPN gateway. Native NordVPN WireGuard support. Built-in kill switch. Health check endpoint. |
| Protocol | **WireGuard** | Faster than OpenVPN, lower overhead. NordVPN supports it natively. |
| qBittorrent network | `network_mode: service:gluetun` | All qBittorrent traffic exits through the VPN tunnel. Kill switch is automatic. |
| *Arr connectivity | **Regular `media-stack` bridge** | They connect to qBittorrent via the host's published port (not over VPN). They don't need VPN for indexer access — Prowlarr handles that through FlareSolverr if needed. |
| Kill switch | **Gluetun built-in** | If VPN drops, qBittorrent has no network path. No iptables hacks needed. |
| Port forwarding | **NordVPN dedicated IP / P2P servers** | Some NordVPN servers support port forwarding. Required for good torrent speeds. Use `-p 6881:6881/tcp -p 6881:6881/udp` mapped through gluetun. |

### Implementation Plan

1. Add Gluetun service to `docker-compose.yml` with NordVPN WireGuard config
2. Change qBittorrent's network to `network_mode: "service:gluetun"`
3. Publish qBittorrent ports THROUGH Gluetun (not directly)
4. Keep all other services on `media-stack` network
5. The *arrs talk to qBittorrent on `localhost:8080` (since qBittorrent and the host are on the same machine, the published port on `0.0.0.0` is reachable)
6. Add `.env.example` with `NORDVPN_PRIVATE_KEY`, `NORDVPN_COUNTRY`, `NORDVPN_SERVER` vars

### `.env` Variables Needed

```bash
NORDVPN_PRIVATE_KEY=           # WireGuard private key from NordVPN
NORDVPN_COUNTRY=us             # Server country
NORDVPN_SERVER=us1234          # Optional: specific server
NORDVPN_DEDICATED_IP=          # Optional: if you have a dedicated IP
PUID=1000
PGID=1000
TZ=America/New_York
```

### What NordVPN Credentials You'll Need

1. NordVPN account with P2P support (standard plan includes this)
2. Generate **WireGuard keys** from the NordVPN dashboard or API
3. Find the best P2P server (typically in your country for lowest latency)
4. Optionally: add a **dedicated IP** for stable port forwarding

---

## Part 2: Proxmox Backup Server (Same-Host with Existing HDDs)

### Goal

Set up Proxmox Backup Server **on the same Proxmox host** as an LXC, using your existing HDDs (2× 1TB + 1× 2TB). This protects against data corruption and accidental deletion, but NOT against full hardware failure (no second machine yet — you can add that later for true DR).

### What PBS Protects vs. What Doesn't

| Scenario | Survives? | Notes |
|----------|-----------|-------|
| Accidental file deletion | ✅ | Restore from rsync backup |
| Disk corruption on media drive | ✅ | Restore from PBS + rsync |
| Docker config broken | ✅ | CT-level PBS backup restores entire container |
| Whole Proxmox host dies | ❌ | Need second node for true DR |
| Fire / theft / power surge | ❌ | Need off-site backup for this |

### Disk Layout Plan with Your HDDs

You have 3 drives to work with. Here's the recommended layout:

```
┌─ Proxmox Host ─────────────────────────────────────────────┐
│                                                             │
│  1TB SSD (boot drive) — PVE OS + Docker + /data/media       │
│                                                             │
│  2× 1TB HDDs ──→ ZFS Mirror (1TB usable)                   │
│                    PBS LXC datastore for VM/CT backups      │
│                    (redundant — if one drive dies, keep     │
│                     your Proxmox container backups)         │
│                                                             │
│  1× 2TB HDD  ──→ Standalone ext4/XFS                       │
│                    Mounted into PBS LXC                     │
│                    rsync target for /data/media backup      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Total usable backup capacity:** ~3 TB (1 TB for CT backups, 2 TB for media)
**Why a mirror for the 1TB drives?** The most important backup is your Docker configs, *arr databases, and OS state. Those are tiny files that change often — ZFS mirror + PBS dedup keeps them safe with redundancy. The media files are large but mostly static, so they go on the larger 2TB drive as a daily rsync target.

### Alternative: MergerFS Single Pool

If you don't care about redundancy and just want max capacity:

```
2× 1TB + 1× 2TB ──→ mergerfs pool (~4TB JBOD) ──→ split as:
                       │  ├── /datastore/proxmox (500 GB for CT backups)
                       │  └── /datastore/media   (3.5 TB for media rsync)
```

Simpler but if any drive dies you lose that drive's data.

### Architecture Diagram

```
┌─ Proxmox Host ─────────────────────────────────────────────────────┐
│                                                                     │
│  ┌─ Media CT (Docker) ─────────────────────┐                       │
│  │                                         │                       │
│  │  qbittorrent │ radarr │ sonarr │ ...    │                       │
│  │  └─ /data/media ──┐                     │                       │
│  └───────────────────┼─────────────────────┘                       │
│                      │                                             │
│                      │  daily rsync                                │
│                      ▼                                             │
│  ┌─ PBS LXC ───────────────────────────────┐                       │
│  │                                         │                       │
│  │  PBS Web UI : 8007                      │                       │
│  │  ┌─────────────────────────────────┐    │                       │
│  │  │ /datastore/proxmox/             │    │ ← ZFS mirror (2×1TB) │
│  │  │   (CT backups via PBS native)   │    │                       │
│  │  └─────────────────────────────────┘    │                       │
│  │  ┌─────────────────────────────────┐    │                       │
│  │  │ /mnt/media-backups/             │    │ ← 2TB HDD (ext4)     │
│  │  │   (daily/ dirs with hard-links) │    │                       │
│  │  └─────────────────────────────────┘    │                       │
│  └─────────────────────────────────────────┘                       │
│                                                                     │
│  ┌─ Auto-Ripper CT ────────────────────────┐                       │
│  │  ARM Web UI : 8085                      │                       │
│  │  MakeMKV → HandBrake → /data/media      │                       │
│  │  USB Blu-ray drive passed through       │                       │
│  └─────────────────────────────────────────┘                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Implementation Plan

#### Step 1: Prepare the HDDs

```bash
# Identify your drives
lsblk -o NAME,SIZE,MODEL,MOUNTPOINT

# Option A: ZFS mirror for 2× 1TB drives
apt install zfsutils-linux
zpool create -f pbs-pool mirror /dev/sdb /dev/sdc

# Option B: Simple ext4 (simpler, no redundancy)
mkfs.ext4 /dev/sdb  # 1TB
mkfs.ext4 /dev/sdc  # 1TB
mkfs.ext4 /dev/sdd  # 2TB
```

#### Step 2: Create PBS LXC

Create a privileged LXC (CT 250, for example) on Proxmox with:
- 2-4 GB RAM, 2-4 cores
- 16 GB root disk (PBS doesn't need much for the OS)
- **Mount points** for your HDDs:
  - ZFS mirror at `/datastore/proxmox` (inside the LXC)
  - 2TB HDD at `/mnt/media-backups` (inside the LXC)
- Network: static IP on your homelab subnet

Pass through the HDDs to the LXC via Proxmox's mount point feature (`mp0`, `mp1`, etc.)

#### Step 3: Install PBS Inside the LXC

```bash
# On the PBS LXC (Debian/Ubuntu)
apt update && apt upgrade -y
echo "deb http://download.proxmox.com/debian/pbs $(lsb_release -cs) pbs-no-subscription" > /etc/apt/sources.list.d/pbs.list
wget -q https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg -O /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg
apt update
apt install proxmox-backup-server -y

# Create datastore for CT backups
proxmox-backup-manager datastore create proxmox /datastore/proxmox

# Create PBS user for authentication
proxmox-backup-manager user create backup-user@pbs
proxmox-backup-manager user update backup-user@pbs --password <password>
```

#### Step 4: Configure Proxmox to Back Up the Media CT

In the Proxmox web UI → Datacenter → Storage → Add → Proxmox Backup Server:
- ID: `pbs`
- Server: IP of PBS LXC
- Datastore: `proxmox`
- Username: `backup-user@pbs`
- Password: as set above

Then select the media CT → Backup → Add → Schedule: Weekly (Sunday 3 AM) → Storage: pbs

#### Step 5: Set Up Media rsync to PBS

On the **media CT** (where Docker runs), create a backup script:

```bash
#!/bin/bash
# /usr/local/bin/backup-media.sh
# Daily rsync of /data/media to PBS LXC's 2TB HDD

PBS_HOST="10.2.7.250"       # PBS LXC IP
SSH_PORT=22
MEDIA_MOUNT="/mnt/pbs-media-backup"

# Mount via SSHFS (no NFS server needed on PBS)
mkdir -p "$MEDIA_MOUNT"
sshfs root@$PBS_HOST:/mnt/media-backups "$MEDIA_MOUNT" -o reconnect,ServerAliveInterval=15

# rsync with hard-link rotation
rsync -avh --delete \
  --link-dest="$MEDIA_MOUNT/latest" \
  /data/media/ "$MEDIA_MOUNT/$(date +%Y-%m-%d)/"

# Update latest symlink
rm -f "$MEDIA_MOUNT/latest"
ln -s "$(date +%Y-%m-%d)" "$MEDIA_MOUNT/latest"

# Unmount
umount "$MEDIA_MOUNT"
```

Cron: `0 2 * * * /usr/local/bin/backup-media.sh`

#### Step 6: Verify Restore Works

Test by restoring the media CT from PBS backup to validate the pipeline.

---

## Part 3: Auto-Ripping Machine

### Goal

Plug in a USB Blu-ray drive, insert a disc, and have it automatically rip, name, and organize into Jellyfin — no manual steps. Ripped files also get backed up by the daily rsync to PBS.

### Architecture

```
┌─ Auto-Ripper CT ────────────────────────────────────────────┐
│                                                              │
│  USB Blu-ray drive ──→ Detect disc inserted                 │
│       │                      (udev / ARM daemon)             │
│       ▼                                                     │
│  MakeMKV ──→ Raw MKV rip (or decrypted ISO)                 │
│       │                                                     │
│       ▼                                                     │
│  HandBrakeCLI ──→ Transcode to H.265 (optional)              │
│       │                                                     │
│       ▼                                                     │
│  FileBot / post-processing ──→ Rename & organize             │
│       │                                                     │
│       ▼                                                     │
│  /data/media/movies/  ──→ Jellyfin picks up automatically   │
│       │                                                     │
│       ▼ (same night)                                        │
│  PBS rsync backup ──→ Ripped movie is also backed up        │
└─────────────────────────────────────────────────────────────┘
```

### Software Choice: ARM (Automatic Ripping Machine)

**ARM** is the gold standard for this — it's a full Docker-based solution with:
- Web UI for monitoring and configuration
- MakeMKV integration for Blu-ray/DVD decryption
- Optional HandBrake transcoding
- Built-in file naming and organization
- Can output directly to your media folders

There's also **Auto-MKV** (simpler, no web UI) and the manual route with udev rules, but ARM is the most set-and-forget.

### What You Need to Buy

| Item | Estimated Cost | Notes |
|------|---------------|-------|
| **USB Blu-ray drive** | $60-100 | LG BP60NB10 or similar. Must be UHD-friendly if you want 4K Blu-rays. |
| **Blank disc space** | Free | Already part of your /data/media |

You can also use an internal SATA Blu-ray drive with a USB adapter if you have one lying around.

### Implementation Plan

#### Step 1: Create a Dedicated CT for the Ripper

Create a new CT (e.g., CT 110) with:
- 2 GB RAM, 2 cores
- 16 GB root disk
- **USB passthrough** for the Blu-ray drive
- Access to `/data/media` (NFS mount from the media CT, or bind mount if on the same host)
- Debian/Ubuntu LXC template

#### Step 2: Pass Through the USB Blu-ray Drive

**On the Proxmox host**, identify the USB device:

```bash
lsusb
# Look for your Blu-ray drive, note bus/device
# e.g., Bus 003 Device 002: ID 2109:0715 VIA Labs, Inc.
```

Add to the CT config (`/etc/pve/lxc/110.conf`):

```
lxc.cgroup2.devices.allow: c 11:* rwm
lxc.mount.entry: /dev/sr0 dev/sr0 none bind,optional,create=file
lxc.mount.entry: /dev/sg0 dev/sg0 none bind,optional,create=file
```

Or use Proxmox's USB passthrough in the CT options (Resources → Add → USB passthrough).

#### Step 3: Deploy ARM via Docker

On the ripper CT, create a `docker-compose.yml`:

```yaml
version: "3.9"

services:
  arm:
    image: ghcr.io/automatic-ripping-machine/automatic-ripping-machine:latest
    container_name: arm
    privileged: true        # Required for optical drive access
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - ARM_UID=1000
      - ARM_GID=1000
    volumes:
      - ./config:/etc/arm
      - /home/arm/media:/home/arm/media  # Output to /data/media
      - /dev:/dev:ro                      # Optical drive access
    devices:
      - /dev/sr0:/dev/sr0
      - /dev/sg0:/dev/sg0
    ports:
      - "8085:8080"   # ARM web UI
    restart: unless-stopped
```

Mount the `/data/media` directory from your media CT via NFS or SSHFS so the ripper can write directly where Jellyfin reads.

#### Step 4: Configure ARM

1. Access web UI at `http://<ripper-ct-ip>:8085`
2. Set up:
   - **Ripper**: MakeMKV (best quality, lossless)
   - **Transcoder**: HandBrakeCLI → H.265 (saves ~50% space over raw MKV)
   - **Output**: `/home/arm/media/movies`
   - **Naming**: `{title} ({year})/{title} ({year})-{source}` for Radarr compatibility
3. Insert a test disc to verify the full pipeline

#### Step 5: Integrate with Radarr (Optional but Recommended)

Once the movie lands in `/data/media/movies/Movie Name (2024)/`, Radarr can be configured to:
- Recognize manual imports
- Rename according to your naming scheme
- Trigger re-scan in Jellyfin

This gives you the best of both worlds: ARM handles the raw rip, Radarr handles the organization.

#### Step 6: Verify Backup Coverage

Ripped movies are in `/data/media/movies/`, which is backed up nightly via the rsync script from Part 2. No extra configuration needed.

### Workflow (End-to-End)

```
1. Walk up to the server, insert a Blu-ray into the USB drive
2. ARM detects the disc ~30 seconds later
3. ARM rips with MakeMKV (~15-30 min per movie)
4. (Optional) HandBrake transcodes to H.265 (~30-60 min)
5. Movie lands in /data/media/movies/Movie Name (2024)/
6. Jellyfin picks it up within minutes
7. Next nightly backup, it's rsynced to PBS
```

---

## Part 4: Migration Path

### Phase 1: VPN Integration (Estimated: 30 min)

1. Get NordVPN WireGuard credentials from NordVPN dashboard
2. Add Gluetun to `docker-compose.yml`
3. Reconfigure qBittorrent to use Gluetun's network
4. Test: verify qBittorrent IP is your VPN IP, *arrs are on your home IP
5. Test kill switch: stop Gluetun, verify qBittorrent loses connectivity

### Phase 2: PBS Setup (Estimated: 2-3 hours)

1. Wipe and prep your HDDs (ZFS mirror the 2× 1TB, ext4 the 2TB)
2. Create PBS LXC with HDD passthroughs
3. Install and configure PBS
4. Set up Proxmox backup schedule for the media CT
5. Set up rsync-based media backup script
6. Test: restore the media CT from PBS backup

### Phase 3: Auto-Ripper Setup (Estimated: 1-2 hours)

1. Buy/connect a USB Blu-ray drive
2. Create ripper CT with USB passthrough
3. Deploy ARM via Docker
4. Test with a known disc
5. Verify Jellyfin picks up the ripped movie

### Phase 4: Restore Drill (Estimated: 30 min)

1. Simulate a failure by taking the media CT offline
2. Restore from PBS backup
3. rsync latest media files from PBS HDD
4. Bring the stack back up

---

## End-to-End Data Flow Summary

```
Disc → ARM → /data/media/movies/ ──┐
                                   ├── → Jellyfin (instant)
Torrent → qBittorrent (VPN)       │
  → Radarr/Sonarr → /data/media/ ─┘
                  ↓
            Daily rsync → PBS LXC (2TB HDD)
                  ↓
            Weekly CT backup → PBS LXC (ZFS mirror, 2×1TB)
                  ↓
            (Future: second node for true DR)
```

---

## Open Questions

1. **Do you have a USB Blu-ray drive** already or need to buy one? If buying, do you want 4K UHD support?
2. **Where are your HDDs physically** — are they SATA drives you can install inside the Proxmox box, or USB externals?
3. **Do you want ZFS mirror for the 2× 1TB drives** (redundant but less capacity), or go full JBOD with mergerFS?
4. **Do you want HandBrake transcoding** in the ripping pipeline, or raw MakeMKV rips? (Raw = full quality, transcode = save space)
5. **Disk space on the 1TB boot drive** — do you have room for Docker and /data/media on the same SSD, or should /data/media live on one of the HDDs?
