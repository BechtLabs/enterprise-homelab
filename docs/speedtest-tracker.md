# Speedtest Tracker

Speedtest Tracker provides automated internet speed testing and historical performance tracking for my home network.

## Environment

* **Platform:** Proxmox VE
* **Docker Host:** Ubuntu VM
* **Docker Host IP:** `192.168.9.200`
* **Container Management:** Portainer
* **Deployment Method:** Docker Compose via Portainer Stack
* **Application:** Speedtest Tracker
* **Container Image:** `lscr.io/linuxserver/speedtest-tracker:latest`
* **Web Port:** `8765`

## Why Portainer Stacks?

Docker is the engine that runs the container.

Docker Compose provides the configuration describing how the application should run.

Portainer provides a graphical interface for managing Docker.

A Portainer **Stack** allows the Docker Compose configuration to be managed directly through Portainer.

For this homelab, using Stacks provides several advantages:

* Application configuration is documented as YAML.
* Containers can be recreated easily.
* Ports, volumes, environment variables, and restart policies are visible in one place.
* Persistent application data remains separate from the disposable container.
* Deployments are easier to troubleshoot and reproduce.
* The Compose configuration can also be used outside Portainer if necessary.

The general model is:

```text
Proxmox
└── Ubuntu Docker VM (.200)
    │
    ├── Docker
    │   └── Speedtest Tracker
    │
    └── Portainer
        └── Stack
            └── Docker Compose configuration
```

## Persistent Storage

Speedtest Tracker configuration and database files are stored on the Docker host:

```text
/opt/speedtest-tracker/config
```

The container maps this directory to:

```text
/config
```

This separates persistent data from the container itself.

If the container is removed or recreated, the historical data and configuration remain on the Docker host.

## Port Mapping

The container's internal web server listens on port `80`.

Port `8765` is used on the Docker host:

```text
192.168.9.200:8765
        ↓
Container port 80
```

This avoids consuming port 80 on the Docker host and reduces conflicts with other services.

## Docker Compose

```yaml
services:
  speedtest-tracker:
    image: lscr.io/linuxserver/speedtest-tracker:latest
    container_name: speedtest-tracker
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - APP_KEY=REDACTED
      - DB_CONNECTION=sqlite
      - SPEEDTEST_SCHEDULE=0 */6 * * *
      - DISPLAY_TIMEZONE=America/New_York
      - PRUNE_RESULTS_OLDER_THAN=0
    volumes:
      - /opt/speedtest-tracker/config:/config
    ports:
      - 8765:80
    restart: unless-stopped
```

> **Security:** The actual `APP_KEY` is intentionally excluded from this repository. Do not commit application keys, passwords, API tokens, or other secrets to GitHub.

## Speed Test Schedule

Tests are configured to run every six hours:

```text
0 */6 * * *
```

This provides four measurements per day without generating excessive tests.

## Local Access

Speedtest Tracker is available from the LAN at:

```text
http://192.168.9.200:8765
```

HTTP is currently used for direct LAN access. HTTPS can be provided separately through the homelab reverse proxy if desired.

## Deployment

In Portainer:

1. Open **Stacks**.
2. Select **Add stack**.
3. Name the stack `speedtest-tracker`.
4. Paste the Docker Compose configuration.
5. Supply the application's `APP_KEY`.
6. Deploy the stack.
7. Verify the container reaches the running state.
8. Open `http://192.168.9.200:8765`.
9. Sign into the administration interface.
10. Run a manual speed test to verify operation.

## Design Principle

Containers should be considered disposable.

Application configuration and data should be persistent and reproducible.

For this deployment:

```text
Container       → Disposable
Compose YAML    → Reproducible configuration
/config         → Persistent application data
Portainer       → Management interface
GitHub          → Documentation / configuration history
```

This approach makes the service easier to maintain, update, troubleshoot, and rebuild.
