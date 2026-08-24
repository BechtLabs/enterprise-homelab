# Self-Hosted Recursive DNS with Technitium on Proxmox

## Project Overview

This project documents the deployment of a self-hosted recursive DNS resolver using **Technitium DNS Server** running inside an unprivileged **Proxmox LXC container**.

The goal was to add a dedicated DNS resolution layer behind an existing Pi-hole environment while maintaining control over DNS recursion, caching, and DNSSEC validation.

Rather than forwarding all DNS requests to a public resolver such as Cloudflare, Google, or Quad9, Technitium was configured to perform recursive DNS resolution directly against the DNS hierarchy.

## Architecture

```text
Clients
   |
   v
Pi-hole
   |
   v
Technitium DNS
192.168.9.15
   |
   +--> Root DNS Servers
   |
   +--> TLD DNS Servers
   |
   +--> Authoritative DNS Servers
```

Technitium acts as the recursive resolver while Pi-hole remains the planned front-end filtering layer.

## Environment

* Hypervisor: Proxmox VE
* Deployment Type: LXC
* Container ID: 107
* Operating System: Debian 13
* Container Type: Unprivileged
* CPU: 1 Core
* RAM: 1 GB
* Disk: 8 GB
* Storage: Local NVMe LVM-Thin
* Static IP: `192.168.9.15/24`
* Gateway: `192.168.9.1`
* Web Interface: `http://192.168.9.15:5380`
* DNS Service: TCP/UDP 53

## Deployment Method

Technitium DNS Server was deployed using the Proxmox VE Community Scripts project.

The LXC was configured with:

```text
CPU:        1 Core
RAM:        1024 MB
Disk:       8 GB
OS:         Debian 13
Type:       Unprivileged
IPv4:       192.168.9.15/24
Gateway:    192.168.9.1
IPv6:       Disabled
```

The container disk was placed on local NVMe storage rather than NFS-backed NAS storage.

This was intentional so DNS availability would not depend on a NAS or network-mounted storage target.

## Why Recursive DNS?

A traditional Pi-hole configuration often forwards DNS queries to a public resolver:

```text
Client
  |
  v
Pi-hole
  |
  v
Cloudflare / Google / Quad9
```

In that architecture, the upstream resolver performs the actual DNS lookup.

With recursive DNS, Technitium performs the lookup itself:

```text
Client
  |
  v
Technitium
  |
  +--> Root Server
  |
  +--> .com TLD Server
  |
  +--> Authoritative DNS Server
```

This removes the requirement for a single public recursive DNS provider and provides greater control over DNS resolution.

## Forwarder Configuration

Technitium was intentionally configured with:

```text
Forwarders: None
```

No Cloudflare, Google, Quad9, or other upstream recursive resolver is configured.

This allows Technitium to perform full recursive DNS resolution.

## DNSSEC

DNSSEC validation was enabled within Technitium.

DNSSEC helps protect DNS responses against forged or modified records by validating cryptographic signatures in the DNS chain.

Configuration:

```text
DNSSEC Validation: Enabled
EDNS UDP Payload: 1232
EDNS Client Subnet: Disabled
```

## Validation Testing

### Direct DNS Test

A direct DNS query was sent from a MacBook to the Technitium server:

```bash
dig @192.168.9.15 google.com
```

Successful response:

```text
status: NOERROR
SERVER: 192.168.9.15#53
```

This confirmed that Technitium was reachable from another LAN device and responding correctly on port 53.

## DNS Cache Test

The first lookup for `google.com` required recursive resolution.

A subsequent lookup returned much faster, demonstrating successful DNS caching.

Example:

```text
Initial lookup: approximately 79 ms
Cached lookup:  approximately 8 ms
```

## DNSSEC Validation Test

A signed domain was queried using:

```bash
dig @192.168.9.15 cloudflare.com +dnssec
```

The response included:

```text
status: NOERROR
flags: qr rd ra ad
```

The `ad` flag indicates:

```text
Authenticated Data
```

This confirms that Technitium successfully validated the DNSSEC chain.

## Broken DNSSEC Test

A deliberately misconfigured DNSSEC domain was queried:

```bash
dig @192.168.9.15 dnssec-failed.org
```

Expected result:

```text
status: SERVFAIL
ANSWER: 0
```

Technitium also reported a DNSSEC validation failure.

This confirms that Technitium is not simply accepting invalid DNS records and that DNSSEC enforcement is functioning correctly.

## Verified Capabilities

The following functionality has been successfully tested:

* [x] Proxmox LXC deployment
* [x] Static IPv4 addressing
* [x] DNS service reachable over LAN
* [x] Recursive DNS resolution
* [x] No external DNS forwarders
* [x] DNS caching
* [x] DNSSEC validation
* [x] Valid DNSSEC domains accepted
* [x] Invalid DNSSEC domains rejected
* [x] Web-based DNS management
* [x] Local NVMe storage for infrastructure independence

## Planned Pi-hole Integration

The next phase will place Technitium behind the existing Pi-hole filtering layer:

```text
LAN Clients
     |
     v
Pi-hole
     |
     v
Technitium DNS
192.168.9.15
     |
     v
Recursive DNS
     |
     v
Root / TLD / Authoritative DNS
```

Pi-hole will continue to provide:

* DNS filtering
* Ad blocking
* Client-level statistics
* Blocklist management

Technitium will provide:

* Recursive DNS resolution
* DNS caching
* DNSSEC validation
* Local DNS infrastructure
* Future local zone management

The first integration test will use only one Pi-hole instance while leaving the secondary Pi-hole on its existing upstream DNS configuration for rollback and redundancy.

## Security Considerations

Technitium was deployed as an unprivileged LXC container.

Additional decisions included:

* No DNS service exposed directly to the Internet
* No public port forwarding
* DNS service restricted to the internal network
* Strong Administrator password configured
* DNSSEC validation enabled
* External forwarding disabled
* IPv6 disabled for the initial deployment
* DNS container stored on local NVMe rather than NAS-backed storage

## Skills Demonstrated

This project demonstrates hands-on experience with:

* Proxmox VE
* Linux containers
* LXC networking
* DNS architecture
* Recursive DNS
* DNSSEC
* DNS caching
* Infrastructure design
* Static IP planning
* Linux services
* Network troubleshooting
* Security-focused deployment
* Validation using `dig`
* Layered DNS architecture

## Current Status

Technitium DNS Server is operational and fully validated as a standalone recursive resolver.

```text
Technitium DNS:   Operational
IPv4 Address:     192.168.9.15
Recursive DNS:    Verified
DNSSEC:           Verified
Caching:          Verified
Forwarders:       None
Pi-hole Link:     Pending
```

## Next Steps

* [ ] Configure one Pi-hole instance to use `192.168.9.15` as its upstream DNS server
* [ ] Verify end-to-end client resolution through Pi-hole and Technitium
* [ ] Confirm DNSSEC behavior through Pi-hole
* [ ] Monitor query and cache statistics
* [ ] Integrate the secondary Pi-hole
* [ ] Create local DNS zones
* [ ] Add architecture screenshots
* [ ] Document rollback procedure
* [ ] Add monitoring and backup strategy
