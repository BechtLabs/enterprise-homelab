## qBittorrent + Gluetun VPN Kill Switch Verification

Validated the Docker networking and VPN fail-closed behavior for my qBittorrent deployment.

### Architecture

```text
qBittorrent
     │
     │ network_mode: container:gluetun
     ▼
Gluetun
     │
     ├── Firewall / Kill Switch
     ▼
WireGuard VPN
     │
     ▼
Internet
```

qBittorrent does not have an independent Docker network path. Its network namespace is provided by the Gluetun container.

### Verification Performed

* Confirmed qBittorrent is attached directly to Gluetun's network namespace.
* Confirmed the public IP presented through Gluetun differs from the ISP WAN IP.
* Started a legal Linux distribution torrent and verified active transfer through the VPN.
* Stopped the Gluetun container during the active transfer.
* Confirmed qBittorrent immediately lost network connectivity.
* Confirmed DNS resolution from qBittorrent also failed while Gluetun was offline.
* Restarted Gluetun and confirmed VPN connectivity and torrent traffic recovered.
* Added `restart: unless-stopped` to qBittorrent.
* Recreated the qBittorrent container while preserving persistent configuration and data mounts.
* Rebooted the Docker host and verified both Gluetun and qBittorrent automatically recovered.
* Reconfirmed qBittorrent remained attached exclusively to Gluetun after reboot.

### Persistent qBittorrent Configuration

```yaml
restart: unless-stopped
network_mode: "container:gluetun"
```

Persistent storage:

```text
/opt/docker/appdata/qbittorrent -> /config
/mnt/data                       -> /data
```

### Result

**VPN kill switch: PASS**

**Container restart recovery: PASS**

**Host reboot recovery: PASS**

**Direct ISP fallback detected: NO**

The important lesson was not simply confirming that the VPN was connected. The real test was intentionally taking the VPN container offline while traffic was active and verifying that qBittorrent failed closed rather than silently falling back to the host's normal Internet connection.
