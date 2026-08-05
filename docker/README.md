# Docker Services

This repository documents my production-style homelab, demonstrating modern infrastructure engineering practices including Docker Compose, Proxmox VE, automation, monitoring, Zero Trust networking, and disaster recovery.

## Current Services

| Service | Purpose | Deployment |
|----------|----------|:------:|
| Dockge | Docker Compose management | ✅ |
| Gluetun | WireGuard VPN gateway | ✅ |
| qBittorrent | Torrent client | ✅ |
| Sonarr | TV automation | ✅ |
| Radarr | Movie automation | ✅ |
| Prowlarr | Index management | ✅ |
| SABnzbd | Usenet downloads | ✅ |
| Plex | Media server | ✅ |
| Jellyfin | Alternate media server | ✅ |
| Tautulli | Plex analytics | ✅ |
| Homepage | Dashboard | ✅ |
| Home Assistant | Home automation | ✅ |

---

## Architecture

```text
Internet
    │
    ▼
Cloudflare Tunnel
    │
    ▼
Proxmox VE
    │
    ▼
Ubuntu Docker VM
    │
    ├── Dockge
    ├── Homepage
    ├── Home Assistant
    ├── Gluetun
    │     └── qBittorrent
    ├── Sonarr
    ├── Radarr
    ├── Prowlarr
    ├── SABnzbd
    ├── Plex
    ├── Jellyfin
    ├── Tautulli
    └── Beszel Agent
```

---

## Documentation

| Service | Documentation |
|----------|---------------|
| Gluetun + qBittorrent | [Migration Guide](gluetun-qbittorrent.md) |
| Dockge | *(Coming Soon)* |
| Plex | *(Coming Soon)* |
| Home Assistant | *(Coming Soon)* |
| Frigate | *(Coming Soon)* |
| Homepage | *(Coming Soon)* |

---

## Current Objectives

- Standardize all Docker Compose projects
- Manage services through Dockge
- Document every service deployment
- Maintain GitHub as infrastructure documentation
- Improve disaster recovery with documented rebuild procedures
