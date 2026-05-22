# Media Stack

A self-hosted media automation stack running entirely in Docker Compose. Inspired by Marius' RS Stack (Automation Avenue).

## 🧩 Services

| Service | Port | Purpose |
|---------|------|---------|
| **qBittorrent** | `8080` | Torrent client |
| **Radarr** | `7878` | Movie automation |
| **Sonarr** | `8989` | TV show automation |
| **Prowlarr** | `9696` | Indexer management |
| **Lidarr** | `8686` | Music automation |
| **Bazarr** | `6767` | Subtitle management |
| **Jellyfin** | `8096` | Media server |
| **FlareSolverr** | `8191` | Cloudflare bypass for indexers |
| **Cross-seed** | `2468` | Cross-seed automation |

## 📁 Folder Structure

```
/data/
├── torrents/
│   ├── movies/
│   ├── music/
│   └── tv/
└── media/
    ├── movies/
    ├── music/
    └── tv/
```

This layout enables **hard links** — files appear in both the torrent and media folders without taking up double disk space. No slow copying.

## 🚀 Quick Start

### 1. Prerequisites

- Linux server (Debian/Ubuntu recommended)
- Docker and Docker Compose installed
- At least 50 GB free disk (more if storing media locally)

### 2. Clone & Prepare

```bash
git clone https://github.com/iNfamousJerk/media-stack.git
cd media-stack
```

### 3. Create Folder Structure

```bash
sudo mkdir -p /data/{torrents/{movies,music,tv},media/{movies,music,tv}}
sudo chown -R 1000:1000 /data
sudo chmod -R 755 /data
```

### 4. Configure Environment

Edit `docker-compose.yml` and update these variables at the top:

| Variable | Default | Description |
|----------|---------|-------------|
| `PUID` | `1000` | Your user ID (run `id -u`) |
| `PGID` | `1000` | Your group ID (run `id -g`) |
| `TZ` | `America/New_York` | Your timezone |
| `DNS_SERVERS` | `1.1.1.1, 1.0.0.1` | Secure DNS (Cloudflare) |

### 5. Launch

```bash
sudo docker compose up -d
```

### 6. Configure Services

| Service | URL | Setup Required |
|---------|-----|----------------|
| **qBittorrent** | `http://<host>:8080` | Login with `admin` / temp password from logs, then change password. Add categories: `movies`, `tv`, `music`. Set default save path to `/data/torrents`. |
| **Prowlarr** | `http://<host>:9696` | Add indexers. Configure Radarr/Sonarr/Lidarr apps with API keys. |
| **Radarr** | `http://<host>:7878` | Set root folder to `/data/media/movies`. Enable hard links. Add qBittorrent as download client. |
| **Sonarr** | `http://<host>:8989` | Set root folder to `/data/media/tv`. Enable hard links. Add qBittorrent as download client. |
| **Lidarr** | `http://<host>:8686` | Set root folder to `/data/media/music`. Enable hard links. Add qBittorrent as download client. |
| **Bazarr** | `http://<host>:6767` | Connect to Radarr and Sonarr via API keys. |
| **Jellyfin** | `http://<host>:8096` | Create admin account. Add media libraries pointing to `/data/media/{movies,tv,music}`. |

## 🔗 Hard Links

This stack uses **hard links** instead of file copies. When a torrent completes:

1. qBittorrent saves to `/data/torrents/{category}/`
2. Radarr/Sonarr/Lidarr creates a hard link to `/data/media/{category}/`
3. The file appears in both places but only uses **one copy** of disk space

Verify hard links are working:
```bash
ls -li /data/torrents/movies/
ls -li /data/media/movies/
# If inode numbers match, hard links are active
```

## 🔐 Secure DNS

All containers use Cloudflare DNS (`1.1.1.1`, `1.0.0.1`) instead of a VPN tunnel. This is the current best practice from server wiki/trash guides — simpler and avoids VPN connectivity issues.

Verify DNS is working:
```bash
docker exec radarr nslookup google.com
```

## 🛠️ Useful Commands

```bash
# View logs
docker compose logs -f

# Restart all services
docker compose restart

# Stop all services
docker compose down

# Update all images
docker compose pull && docker compose up -d

# Check container status
docker ps
```

## 📐 Plans & Upcoming

| Document | Description |
|----------|-------------|
| [VPN + Backup Architecture](PLANS/01-vpn-and-backup-architecture.md) | NordVPN via Gluetun + Proxmox Backup Server with existing HDDs |
| [Auto-Ripping Machine](PLANS/02-auto-ripper.md) | USB Blu-ray drive + ARM for automated disc ripping into Jellyfin |
| [Off-Site Backup](PLANS/03-offsite-backup.md) | Enterprise PC at brother's house + Tailscale + PBS sync for DR |

## 📚 Resources

- [Trash Guides](https://trash-guides.info) — Configuration best practices
- [Servarr Wiki](https://wiki.servarr.com) — Official documentation
- [Automation Avenue](https://youtube.com/@automationavenue) — Original RS Stack video
