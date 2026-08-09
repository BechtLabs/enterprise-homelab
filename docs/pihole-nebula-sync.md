# Pi-hole DNS Redundancy with Nebula Sync

## Overview

My home lab uses two Pi-hole v6 DNS servers to provide redundant DNS filtering and local name resolution.

Rather than maintaining two independent Pi-hole configurations, I deployed Nebula Sync to synchronize configuration changes from the primary Pi-hole to the secondary Pi-hole.

| Role | IP Address | Platform |
|---|---|---|
| Primary Pi-hole | 192.168.9.4 | Raspberry Pi 4 |
| Secondary Pi-hole | 192.168.9.14 | Proxmox LXC |
| Nebula Sync | 192.168.9.200 | Ubuntu Docker VM |

Network clients are configured with both Pi-hole servers as DNS resolvers.

---

## Architecture

```text
                         +----------------------+
                         | Primary Pi-hole      |
                         | Raspberry Pi 4       |
                         | 192.168.9.4          |
                         +----------+-----------+
                                    |
                                    | Configuration Sync
                                    v
                         +----------------------+
                         | Nebula Sync          |
                         | Ubuntu Docker VM     |
                         | 192.168.9.200        |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         | Secondary Pi-hole    |
                         | Proxmox LXC          |
                         | 192.168.9.14         |
                         +----------------------+

DNS Clients
    |
    +--------> 192.168.9.4
    |
    +--------> 192.168.9.14
```

This design separates DNS availability from configuration synchronization. Clients can query either Pi-hole, while Nebula Sync maintains consistent configuration between them.

---

## Pi-hole Environment

Pi-hole versions at deployment:

- Pi-hole Core: v6.4.3
- FTL: v6.7
- Web Interface: v6.6

Both Pi-hole servers use Cloudflare DNSSEC-enabled upstream resolvers.

Local DNS domain:

```text
lan
```

Example local DNS records include:

```text
homarr.bechtfamily.org
plex.bechtfamily.org
radarr.bechtfamily.org
sonarr.bechtfamily.org
```

These records allow internal services to be accessed by hostname rather than IP address.

---

## Nebula Sync Deployment

Nebula Sync runs as a Docker container on the Ubuntu Docker VM at:

```text
192.168.9.200
```

Docker Compose directory:

```text
/opt/docker/compose/nebula-sync
```

Synchronization direction:

```text
Primary:   http://192.168.9.4
Replica:   http://192.168.9.14
```

The Raspberry Pi installation at `.4` is treated as the authoritative Pi-hole configuration.

Changes made to the primary can then be synchronized to the secondary Pi-hole at `.14`.

---

## Credential Management

Pi-hole v6 application passwords are used by Nebula Sync to authenticate to the Pi-hole API.

Credentials are not stored directly in `compose.yaml`.

They are stored separately:

```text
secrets/primary.txt
secrets/replicas.txt
```

The secret files were assigned restrictive filesystem permissions.

Example:

```bash
chmod 400 secrets/primary.txt secrets/replicas.txt
```

Application passwords and other credentials are intentionally excluded from this repository.

---

## Pi-hole API Configuration

Nebula Sync requires permission to modify Pi-hole configuration through the API.

On the Pi-hole servers, application passwords were configured through:

```text
Settings
  -> Web Interface / API
  -> Configure application password
```

The following Pi-hole v6 API option was also enabled:

```text
webserver.api.app_sudo
```

This allows trusted applications authenticated with an application password to modify Pi-hole configuration.

This permission should only be enabled for applications that are trusted to make configuration changes.

---

## Authentication Testing

Before troubleshooting Nebula Sync itself, API authentication was tested directly against each Pi-hole.

Example API test:

```bash
curl -s -o /tmp/pihole-auth.json -w "HTTP %{http_code}\n" \
  -X POST http://PIHOLE-IP/api/auth \
  -H "Content-Type: application/json" \
  --data "{\"password\":\"$PIPASS\"}"
```

Successful authentication returns:

```text
HTTP 200
```

Both Pi-hole servers were independently verified before another synchronization attempt was made.

---

## Initial Problem

The first Nebula Sync attempts failed during authentication:

```text
Authenticating clients...
Failed to invalidate session
Sync failed
unexpected status code: 401
```

The HTTP `401 Unauthorized` response indicated that Nebula Sync could reach the Pi-hole API, but the supplied application credential was not being accepted.

Instead of changing multiple components at once, each Pi-hole was tested independently.

The primary Pi-hole at `.4` successfully returned:

```text
HTTP 200
```

The secondary Pi-hole at `.14` continued returning:

```text
HTTP 401
```

This isolated the problem to authentication on the secondary Pi-hole.

---

## Root Cause

The issue was related to the Pi-hole application password lifecycle.

Generating a new application password in the Pi-hole interface does not by itself make that credential active. The new password must also be explicitly enabled or used to replace the currently configured application password.

The working process was:

1. Generate a new Pi-hole application password.
2. Copy the generated password.
3. Enable or replace the active application password in Pi-hole.
4. Store the same credential in the appropriate Nebula Sync secret file.
5. Test authentication directly against `/api/auth`.
6. Confirm an `HTTP 200` response.
7. Run Nebula Sync again.

After correcting the secondary credential, both Pi-hole servers returned:

```text
HTTP 200
```

---

## Successful Synchronization

A manual synchronization was run from the Ubuntu Docker VM:

```bash
cd /opt/docker/compose/nebula-sync
sudo docker compose run --rm nebula-sync
```

The successful run produced:

```text
Starting nebula-sync v0.11.2
Running sync mode=full replicas=1
Authenticating clients...
Syncing teleporters...
Syncing configs...
Running gravity...
Invalidating sessions...
Sync completed
```

This confirmed successful synchronization from the primary Pi-hole to the secondary.

---

## DNS Validation

DNS resolution was tested independently against both servers.

Primary:

```bash
nslookup google.com 192.168.9.4
```

Secondary:

```bash
nslookup google.com 192.168.9.14
```

Local Pi-hole name resolution was also tested:

```bash
nslookup pi.hole 192.168.9.4
nslookup pi.hole 192.168.9.14
```

Both Pi-hole servers successfully responded to DNS queries.

---

## Security Considerations

The deployment follows several basic security practices:

- Pi-hole application passwords are not committed to GitHub.
- Credentials are separated from the Docker Compose configuration.
- Secret files use restrictive filesystem permissions.
- API configuration changes require application-password authentication.
- Pi-hole DNS services remain internal to the home network.
- No public DNS port forwarding is required.
- The secondary Pi-hole provides DNS redundancy if the primary server becomes unavailable.

---

## Troubleshooting Approach

This deployment reinforced a troubleshooting approach I use throughout my home lab:

1. Verify basic network connectivity.
2. Test each component independently.
3. Validate authentication outside the application.
4. Change one variable at a time.
5. Retest after each change.
6. Document the root cause and resolution.

In this case, directly testing the Pi-hole API made it possible to distinguish a Nebula Sync problem from a Pi-hole authentication problem and quickly identify which server was rejecting the credentials.

---

## Result

The final environment provides two functioning Pi-hole DNS servers with a centralized configuration workflow:

```text
Primary Pi-hole
      |
      | configuration changes
      v
  Nebula Sync
      |
      v
Secondary Pi-hole
```

The primary Raspberry Pi remains the authoritative configuration source, while the Proxmox-hosted Pi-hole provides a redundant DNS resolver.

Nebula Sync eliminates the need to manually reproduce Pi-hole configuration changes across both servers.
