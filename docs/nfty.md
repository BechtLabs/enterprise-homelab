# Self-Hosted ntfy Notification Server

## Overview

ntfy was deployed as a self-hosted notification service for the homelab. It provides a centralized method for infrastructure services, monitoring applications, and scripts to send alerts to subscribed devices.

The initial integration uses **Uptime Kuma → ntfy**, allowing monitored service failures and recoveries to generate notifications.

### Architecture

```text
Monitored Services
       │
       ▼
  Uptime Kuma
       │
       │ HTTP
       ▼
     ntfy
192.168.9.200:8094
       │
       ├──────── Local Web Access
       │
       ▼
Cloudflare Tunnel
       │
       │ HTTPS
       ▼
ntfy.bechtfamily.org
       │
       ▼
Browser / Mobile Clients
```

No inbound router ports are opened.

---

## Host

ntfy runs as a Docker container on the Ubuntu Docker VM.

```text
Host: Ubuntu Docker VM
IP:   192.168.9.200
Port: 8094
```

Docker and container management are handled through **Portainer**.

---

## Docker Compose

ntfy was deployed as a Portainer Stack.

```yaml
services:
  ntfy:
    image: binwiederhier/ntfy:latest
    container_name: ntfy
    command:
      - serve
    environment:
      - TZ=America/New_York
    volumes:
      - /opt/ntfy/cache:/var/cache/ntfy
      - /opt/ntfy/etc:/etc/ntfy
    ports:
      - "8094:80"
    restart: unless-stopped
```

### Persistent Storage

Host directories are mapped into the container:

```text
HOST                         CONTAINER
/opt/ntfy/cache      →       /var/cache/ntfy
/opt/ntfy/etc        →       /etc/ntfy
```

This separates persistent ntfy data from the container itself so the container can be recreated or updated without relying on its internal filesystem.

---

## Port Selection

The original deployment attempted to use TCP port `8085`.

Deployment failed because the port was already assigned to:

```text
IT Tools → 8085
```

Port `8095` was also investigated but was already in use by Music Assistant.

Listening services were reviewed with:

```bash
sudo ss -tulpn | grep LISTEN
```

Port `8094` was available and was selected for ntfy.

Final mapping:

```text
192.168.9.200:8094 → ntfy container port 80
```

This was a useful reminder to verify existing Docker and host ports before assigning ports to new services.

---

## Initial Testing

The ntfy web interface was verified locally at:

```text
http://192.168.9.200:8094
```

A topic named:

```text
homelab
```

was created.

A test notification was published with:

```bash
curl -d "ntfy is working on VM 200" \
  http://192.168.9.200:8094/homelab
```

The notification immediately appeared in the subscribed `homelab` topic.

This verified:

```text
Client → ntfy server → topic → subscriber
```

---

## Cloudflare Tunnel

ntfy was then published through the existing Cloudflare Tunnel infrastructure.

### Public Hostname

```text
ntfy.bechtfamily.org
```

### Tunnel Origin

```text
Protocol: HTTP
Origin:   192.168.9.200:8094
```

The internal ntfy service remains HTTP.

Cloudflare provides HTTPS for the external connection:

```text
Internet
    │
    │ HTTPS
    ▼
ntfy.bechtfamily.org
    │
    ▼
Cloudflare
    │
    │ Tunnel
    ▼
http://192.168.9.200:8094
    │
    ▼
ntfy
```

No router port forwarding is required.

Cloudflare Access is also enabled, providing an authentication challenge before allowing browser access to the externally published ntfy interface.

---

## Why HTTPS Matters

When ntfy was initially accessed directly through:

```text
http://192.168.9.200:8094
```

the web interface displayed:

```text
Notifications not supported
Notifications are only supported over HTTPS.
```

After accessing ntfy through:

```text
https://ntfy.bechtfamily.org
```

the browser had a secure HTTPS connection suitable for browser notification functionality.

---

# Uptime Kuma Integration

Uptime Kuma was selected as the first service to integrate with ntfy.

Because Uptime Kuma and ntfy are both available inside the homelab, Kuma communicates directly with the local ntfy endpoint rather than sending notification traffic through Cloudflare.

### Uptime Kuma ntfy Configuration

```text
Notification Type:     Ntfy
Friendly Name:         ntfy - Homelab
ntfy Topic:            homelab
Server URL:            http://192.168.9.200:8094
Authentication Method: None
```

The topic is configured separately from the server URL.

Uptime Kuma effectively publishes to:

```text
http://192.168.9.200:8094/homelab
```

A test notification was successfully delivered to ntfy.

---

## Notification Flow

The resulting monitoring path is:

```text
Application / Server
        │
        ▼
   Uptime Kuma
        │
        │ Detects DOWN/UP state
        ▼
      ntfy
        │
        ▼
  homelab topic
        │
        ▼
Subscribed Devices
```

ntfy does not perform the monitoring itself.

Instead:

* **Uptime Kuma detects the event**
* **ntfy transports the notification**
* **Subscribed clients receive the alert**

This separation allows ntfy to become a common notification platform for many different homelab systems.

---

## Potential Future Integrations

The same ntfy server can eventually receive notifications from:

```text
Uptime Kuma
Home Assistant
Proxmox VE
Proxmox Backup Server
Synology NAS
Docker maintenance scripts
Speedtest Tracker
Network monitoring
Backup scripts
Security monitoring
```

Additional topics could also be introduced later:

```text
homelab
network
servers
docker
nas
backups
security
```

For now, the single `homelab` topic keeps the notification architecture simple.

---

## Security Improvements

The current deployment successfully establishes the notification infrastructure, but additional ntfy security configuration is planned.

Future improvements include:

* Enable ntfy authentication
* Create ntfy user accounts
* Configure topic access controls
* Restrict anonymous publishing
* Configure the external base URL
* Evaluate mobile push notification configuration

Cloudflare Access protects the externally exposed web interface, while ntfy authentication will provide application-level control over publishing and subscribing.

These are separate security layers and serve different purposes.

---

## Useful Commands

Check whether ntfy is running:

```bash
docker ps | grep ntfy
```

View logs:

```bash
docker logs ntfy
```

Follow logs:

```bash
docker logs -f ntfy
```

Check the local ntfy endpoint:

```bash
curl http://192.168.9.200:8094/v1/health
```

Send a test notification:

```bash
curl -d "Homelab test notification" \
  http://192.168.9.200:8094/homelab
```

Check listening ports:

```bash
sudo ss -tulpn | grep LISTEN
```

Restart ntfy:

```bash
docker restart ntfy
```

---

## Result

ntfy is now operating as the centralized notification transport for the homelab.

Current verified path:

```text
Uptime Kuma
     ↓
ntfy @ 192.168.9.200:8094
     ↓
homelab topic
     ↓
Cloudflare Tunnel / Local Access
     ↓
Subscriber
```

The deployment provides a foundation for consolidating infrastructure alerts into a single self-hosted notification platform while maintaining the homelab design principle of **no directly exposed inbound ports**.
