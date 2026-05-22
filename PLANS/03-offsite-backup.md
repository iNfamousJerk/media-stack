# Off-Site Backup Plan (Brother's House)

> **Status:** Draft / Proposed  
> **Date:** 2026-05-22

---

## Architecture

```
┌─ YOUR HOUSE ───────────────────────┐     ┌─ BROTHER'S HOUSE ─────────────┐
│                                    │     │                               │
│  Proxmox Host                      │     │  Cheap Enterprise PC (PBS)    │
│  ├── Media CT (Docker stack)       │     │  ├── PBS datastore            │
│  ├── Local PBS LXC (3 HDDs)        │     │  │   (CT backups only)        │
│  │   ├── 2×1TB ZFS → CT backups   │─────┼──┤                            │
│  │   └── 2TB → media rsync         │     │  └── (optional media drive)   │
│  └── Auto-ripper CT (future)       │     │                               │
│                                    │     │                               │
│         ◄── Tailscale VPN ──►      │     │                               │
└────────────────────────────────────┘     └───────────────────────────────┘
```

## Why This Works Well

| Need | Solution |
|------|----------|
| **Zero config networking** | Tailscale — WireGuard mesh VPN, no port forwarding, NAT traversal |
| **Small, critical backups off-site** | PBS datastore sync — CT backups are tiny (configs, DBs, OS) |
| **Large media stays local** | 2TB HDD handles this — no internet upload bottleneck |
| **True disaster recovery** | Fire/theft/power surge at your house → buy a new box, restore CT from brother's PBS |

## Connectivity: Tailscale

Tailscale is perfect here — free for personal use, works through any router/NAT, no open ports needed:

```
Your house:  tailscale up     → 100.x.x.1
Brother's:   tailscale up     → 100.x.x.2
              ↓
Both PBS instances communicate over the encrypted WireGuard tunnel
```

Install once, forget it exists.

## What Gets Synced (and What Doesn't)

### ✅ CT Backups (synced daily — tiny, ~1-5 GB)
These are your Docker configs, *arr databases, Jellyfin metadata, OS state. If your house burns down, you buy a new PC, restore the CT from brother's PBS, and you're back in business. The *arr databases alone are worth their weight in gold (years of curation).

### ❌ Media Files (NOT synced — too large)
Your 4K movies and TV shows stay local on the 2TB HDD. If your house burns down, you lose the media files but not the *curation* — the *arr databases at brother's house know *what* you had, and Radarr/Sonarr can re-download everything. You lose the bytes, not the catalog.

### Optional: New-Only Media Sync
If your internet upload can handle it, set up a separate rclone sync for "new additions only" — just new media that arrived in the last 24 hours (small enough to transfer overnight). This way even your most recent acquisitions survive a fire.

## Implementation

### Step 1: Set Up Brother's PBS Machine

The enterprise PC we were discussing — install Proxmox + PBS on it:

```bash
# Install Proxmox VE (free)
# Download ISO from proxmox.com, install like any other server

# Then install PBS inside a CT or as a VM:
# Or install PBS directly on the metal if it's dedicated to backups
```

Storage — even a single 1-2 TB HDD in his machine is plenty for CT backups. If he has a spare drive, even better.

### Step 2: Install Tailscale on Both Sides

**On your Proxmox host** (or PBS LXC):
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
# → Authenticate via browser
# → Note your Tailscale IP (100.x.x.x)
```

**On brother's PBS machine:**
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
# → Authenticate via browser
```

Now both can ping each other on their 100.x.x.x addresses.

### Step 3: Configure PBS Remote Sync

PBS has built-in sync — it can pull backups from one PBS to another. This is the cleanest approach:

**On brother's PBS** (pull mode — he pulls from you):
```bash
# Add your local PBS as a remote
proxmox-backup-manager remote add local-pbs \
  --host <your-tailscale-ip>:8007 \
  --fingerprint <your-pbs-fingerprint>

# Create a sync job (runs daily, pulls new CT backups)
proxmox-backup-manager sync create \
  --remote local-pbs \
  --remote-datastore proxmox \
  --local-datastore proxmox \
  --schedule "0 3 * * *"
```

PBS handles encryption, dedup, and compression automatically over the wire.

### Step 4: Verify

**Check sync status:**
```bash
proxmox-backup-manager sync list
proxmox-backup-manager sync log <sync-id>
```

**Simulate a disaster:**
1. "Lose" your Proxmox host (shut it down)
2. Go to brother's house
3. Install Proxmox on a new machine
4. Add brother's PBS as a storage
5. Restore the media CT → all configs, DBs, Docker Compose intact
6. Buy new HDDs for media → *arrs re-download over time

## Network Requirements

| Side | What's Needed | Why |
|------|---------------|-----|
| Your house | **10 Mbps+ upload** (preferred) | PBS sync of CT backups is small, but faster is better |
| Brother's house | **Any internet** | Pull mode — he initiates the connection |
| Both | **Tailscale** (free) | No open ports, no static IPs needed |

PBS sync is very bandwidth-efficient — CT backups are dedup'd and compressed. A nightly sync of ~1 GB is totally feasible on a 10 Mbps upload.

## Hardware for Brother's PC

Since you were already looking for a cheap enterprise PC (~$100-150):

| Spec | Minimum | Nice |
|------|---------|------|
| CPU | 2 cores | 4 cores |
| RAM | 4 GB | 8 GB |
| Storage | 500 GB HDD | 2 TB HDD or more |
| Network | Any ethernet | Gigabit |

Even a $60 Dell Optiplex 3020 with a 500GB HDD handles this job easily — it's just a PBS sync target.

## Cost Breakdown

| Item | Est. Cost |
|------|-----------|
| Enterprise PC (brother's house) | $60-150 |
| Storage for it | Free (whatever he's got) or ~$30 for a used 1TB HDD |
| Tailscale | Free |
| **Total** | **~$60-150** |

That's full off-site disaster recovery for the cost of dinner out. Can't beat it.

## Summary: What Survives What

| Disaster | Media Files | Configs/Curation | Recovery |
|----------|-------------|------------------|----------|
| Accidental delete | ❌ Gone | ✅ Safe | Restore from local HDD backup |
| Disk failure | ✅ On ZFS mirror | ✅ On ZFS mirror | Replace drive, resilver |
| House fire | ❌ Gone | ✅ At brother's | Buy new PC, restore from his PBS |
| Ransomware | ✅ On local PBS | ✅ At brother's | Restore from before infection |
