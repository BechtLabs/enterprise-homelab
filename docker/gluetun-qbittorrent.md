# Gluetun + qBittorrent Migration

## Objective

Migrate qBittorrent from a standalone container to a secure Docker deployment protected by Gluetun.

---

## Goals

- Route all BitTorrent traffic through WireGuard
- Prevent IP leaks using Gluetun firewall
- Maintain compatibility with Sonarr and Radarr
- Preserve download paths
- Simplify future maintenance using Dockge

---

## Environment

| Component | Value |
|-----------|-------|
| Host | Ubuntu Docker VM |
| Hypervisor | Proxmox |
| Docker Manager | Dockge |
| VPN | Gluetun |
| Torrent Client | LinuxServer qBittorrent |

---

## Container Layout

Internet
↓

WireGuard VPN

↓

Gluetun

↓

qBittorrent

↓

Sonarr / Radarr

---

## Ports

| Port | Purpose |
|------|----------|
|8080|qBittorrent WebUI|
|8082|Secondary UI|
|6881 TCP|BitTorrent|
|6881 UDP|BitTorrent DHT|

---

## Download Paths

Movies

/data/downloads/torrents/movies

TV

/data/downloads/torrents/tv

Incomplete

/data/downloads/torrents/incomplete

---

## Lessons Learned

- Docker volume mappings must match Sonarr exactly.
- qBittorrent category paths are critical.
- Health checks immediately reveal mapping problems.
- Dockge makes Compose management significantly easier than Portainer for multi-container stacks.

---

## Status

✅ Working

- Gluetun healthy
- qBittorrent healthy
- Sonarr healthy
- Radarr healthy
- Prowlarr connected
