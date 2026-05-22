# VPN + Backup Architecture Plan

> **Status:** Draft / Proposed
> **Author:** Hermes
> **Date:** 2026-05-22

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

## Part 2: Proxmox Backup Server + Media Backup

### Goal

Set up a second physical (or VM/LXC) node running Proxmox Backup Server (PBS) with a mount point for media file backups. Keep copies of `/data/media` in case the primary node fails.

### Architecture

```
┌─────────────────────────────────┐     ┌──────────────────────────────────┐
│       Node A (Primary)          │     │       Node B (Backup)            │
│   Proxmox VE + Media CT/VM      │     │   Proxmox Backup Server          │
│                                 │     │                                  │
│  ┌─────────────────────────┐    │     │  ┌──────────────────────────┐    │
│  │ /data/media/            │    │     │  │ /datastore/proxmox/      │    │
│  │   movies/               │────┼─────┼──│   (VM/CT backups via PBS) │    │
│  │   tv/                   │    │ rsync│  └──────────────────────────┘    │
│  │   music/                │    │     │                                  │
│  └─────────────────────────┘    │     │  ┌──────────────────────────┐    │
│                                 │     │  │ /datastore/media/        │    │
│  ┌─────────────────────────┐    │────┼────│   movies/                │    │
│  │ /data/torrents/         │    │ rsync│   tv/                     │    │
│  │   (no backup needed)    │    │     │   music/                   │    │
│  └─────────────────────────┘    │     │  └──────────────────────────┘    │
│                                 │     │                                  │
│  ┌─────────────────────────┐    │     │  (Optional)                     │
│  │ Media stack (Docker)     │    │     │  ┌──────────────────────────┐    │
│  └─────────────────────────┘    │     │  │ Cold storage /           │    │
│                                 │     │  │ Off-site archive (glacier)│    │
└─────────────────────────────────┘     │  └──────────────────────────┘    │
                                        └──────────────────────────────────┘
```

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| PBS host | **Second Proxmox node** (physical recommended) | True disaster recovery — if the main node dies, PBS on a separate machine survives. If you don't have a second machine, run PBS as a privileged LXC on the same host, but this only protects against data corruption, not hardware failure. |
| PBS storage | **Separate disk or ZFS pool** | Don't store PBS datastore on the same disk as the media. Dedicated spinning rust or RAID is ideal. |
| Media backup method | **rsync + cron, not PBS datastore** | PBS is amazing for VM/CT backups (dedup, incremental, encrypted). But for raw media files (large, mostly static), a simple rsync is more efficient. Mount a PBS directory or NFS share as the target. |
| VM/CT backups | **Native PBS client** | Use Proxmox's built-in backup to PBS for the entire media CT. This captures the OS, Docker configs, everything. |
| Schedule | **Daily rsync for media, Weekly PBS backups for CT** | Media doesn't change fast enough to need hourly. CT backup weekly captures config changes. |
| Transport | **SSH + rsync over LAN** | If on same switch, use private IPs. No sensitive data crossing the WAN. |

### Implementation Plan

#### Step 1: Set up Proxmox Backup Server

1. Download PBS ISO or install on Debian: `apt install proxmox-backup-server`
2. Create a datastore: `proxmox-backup-manager datastore create proxmox /datastore/proxmox`
3. Create a media backup datastore: `proxmox-backup-manager datastore create media /datastore/media`
4. Configure authentication (API token or user/pass)

#### Step 2: Configure Proxmox to Back Up the Media CT

In the Proxmox web UI:
1. Select the media CT → Backup → Add
2. Schedule: Weekly (e.g., Sunday 3 AM)
3. Storage: PBS datastore
4. Mode: Snapshot (no downtime)

#### Step 3: Set Up Media rsync to PBS

On the **media CT**, create a backup script:

```bash
#!/bin/bash
# /usr/local/bin/backup-media.sh
# Runs via cron. Syncs /data/media to PBS media mount.

PBS_HOST="10.2.7.x"  # PBS node IP
PBS_MEDIA_MOUNT="/mnt/pbs-media-backup"
RSYNC_DEST="$PBS_MEDIA_MOUNT/$(date +%Y-%m-%d)"

# Mount PBS NFS share (if using NFS export)
# mount -t nfs "$PBS_HOST:/datastore/media" "$PBS_MEDIA_MOUNT"

# Rsync with hard-link rotation (space efficient)
rsync -avh --delete \
  --link-dest="$PBS_MEDIA_MOUNT/latest" \
  /data/media/ "$RSYNC_DEST/"

# Update 'latest' symlink
rm -f "$PBS_MEDIA_MOUNT/latest"
ln -s "$(date +%Y-%m-%d)" "$PBS_MEDIA_MOUNT/latest"
```

Cron: `0 2 * * * /usr/local/bin/backup-media.sh`

#### Step 4: Configure PBS Access

Options for mounting PBS storage on the media CT:
- **NFS export** from PBS → mount on media CT (simplest)
- **SSHFS** from media CT → PBS datastore (no NFS setup needed)
- **SMB/CIFS** if already using a NAS

### Hardware Requirements

**Minimum for PBS node (dedicated):**
- CPU: 2+ cores (any modern x86)
- RAM: 4-8 GB
- Storage: At least 1.5× your media library size (for dedup + incremental overhead)
- Network: 1 Gbps (preferred, 100 Mbps minimum)

**If running PBS as an LXC on the same host:**
- Not true disaster recovery, but protects against data corruption
- Give it 4 GB RAM, 4 cores, and a separate ZFS dataset/disk
- Mount point for the PBS datastore must be on a different physical disk than /data/media

### Recommended Disk Layout (PBS Node)

```
sda (256 GB NVMe) — OS + PBS installation
sdb (4 TB HDD)    — PBS datastore for VM/CT backups
sdc (8 TB HDD)    — PBS datastore for media rsync target
```

Or use a single large ZFS pool if that's all you have.

---

## Part 3: Migration Path

### Phase 1: VPN Integration (Estimated: 30 min)

1. Get NordVPN WireGuard credentials from NordVPN dashboard
2. Add Gluetun to `docker-compose.yml`
3. Reconfigure qBittorrent to use Gluetun's network
4. Test: verify qBittorrent IP is your VPN IP, *arrs are on your home IP
5. Test kill switch: stop Gluetun, verify qBittorrent loses connectivity

### Phase 2: PBS Setup (Estimated: 1-2 hours)

1. Install PBS on second node (or LXC)
2. Create datastores
3. Configure Proxmox backup schedule for the media CT
4. Set up rsync-based media backup
5. Test restore from backup

### Phase 3: Restore Drill (Estimated: 30 min)

1. Simulate a failure by taking the media CT offline
2. Restore from PBS backup
3. rsync latest media files from PBS
4. Bring the stack back up

---

## Open Questions

1. **Do you have a second physical machine** for the PBS node, or should we run it as an LXC on the same Proxmox host?
2. **What's your media library size** (current and projected growth)? This determines PBS storage requirements.
3. **Do you have a dedicated NordVPN IP** for port forwarding, or should we pick a P2P server?
4. **Network setup** — are both nodes on the same switch/VLAN, or do they need to route through the firewall?
5. **Do you want to do a restore drill** after setup to validate the backups actually work?
