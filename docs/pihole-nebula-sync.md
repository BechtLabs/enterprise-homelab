# Pi-hole High Availability with Nebula Sync

## Overview

My home network uses two Pi-hole v6 DNS servers for redundancy.

| Role | IP Address | Platform |
|---|---|---|
| Primary Pi-hole | 192.168.9.4 | Raspberry Pi 4 |
| Secondary Pi-hole | 192.168.9.14 | Proxmox LXC |
| Nebula Sync | 192.168.9.200 | Ubuntu Docker VM |

Both Pi-hole servers are provided to network clients as DNS servers.

Nebula Sync synchronizes the configuration from the primary Pi-hole to
the secondary Pi-hole.

## Architecture

Primary Pi-hole (.4)
        |
        | Configuration synchronization
        v
Nebula Sync (.200)
        |
        v
Secondary Pi-hole (.14)

DNS clients can use either .4 or .14.

## Pi-hole Versions

At deployment:

- Pi-hole Core: v6.4.3
- FTL: v6.7
- Web Interface: v6.6

## DNS Configuration

Both Pi-hole servers use Cloudflare DNSSEC upstream resolvers.

Local DNS domain:

lan

Example local DNS records include:

- homarr.bechtfamily.org
- plex.bechtfamily.org
- radarr.bechtfamily.org
- sonarr.bechtfamily.org

## Nebula Sync

Nebula Sync runs as a Docker container on the Ubuntu Docker VM.

Compose directory:

/opt/docker/compose/nebula-sync

Primary:

http://192.168.9.4

Replica:

http://192.168.9.14

Pi-hole application passwords are stored using Docker secrets and are
not stored directly in compose.yaml.

Secret files:

secrets/primary.txt
secrets/replicas.txt

The secret files are restricted to the Nebula Sync container user.

## Pi-hole API Configuration

Application passwords are used instead of normal Pi-hole administrator
passwords.

On the replica, the following Pi-hole v6 option is enabled:

webserver.api.app_sudo

This permits Nebula Sync to modify configuration through the Pi-hole API.

## Synchronization

Nebula Sync performs a full synchronization using Pi-hole's Teleporter
functionality.

The first successful manual synchronization produced:

Authenticating clients...
Syncing teleporters...
Syncing configs...
Running gravity...
Invalidating sessions...
Sync completed

## Testing

DNS resolution was tested independently against both Pi-hole servers.

Examples:

nslookup google.com 192.168.9.4
nslookup google.com 192.168.9.14

Pi-hole local DNS was also verified:

nslookup pi.hole 192.168.9.4
nslookup pi.hole 192.168.9.14

Both servers successfully returned IPv4 and IPv6 DNS records.

## Troubleshooting Notes

Initial Nebula Sync attempts returned:

HTTP 401 Unauthorized

The problem was traced to Pi-hole application passwords that had been
generated but had not yet been activated.

The correct sequence is:

1. Generate the Pi-hole application password.
2. Copy the generated password.
3. Click "Enable new app password" or "Replace app password".
4. Store the activated password in the Nebula Sync secret.
5. Test the credential against the Pi-hole API.

A successful API authentication test returns:

HTTP 200

After both Pi-hole API credentials returned HTTP 200, Nebula Sync
completed successfully.

## Security

- Pi-hole application passwords are not stored in compose.yaml.
- Credentials are stored in Docker secret files.
- Secret files use restrictive filesystem permissions.
- Pi-hole DNS remains accessible only from the local network.
- No Internet-facing DNS ports are required.
