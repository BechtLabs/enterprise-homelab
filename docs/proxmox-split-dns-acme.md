# Proxmox Local HTTPS with Pi-hole, Cloudflare, and ACME

## Goal

Create a clean HTTPS experience for Proxmox using the same hostname both inside and outside the home network:

`proxmox.bechtfamily.org`

The objectives were:

* Use Cloudflare Access and Cloudflare Tunnel for remote access
* Keep Proxmox traffic local while at home
* Eliminate browser certificate warnings
* Avoid exposing Proxmox ports directly to the Internet
* Avoid adding Nginx solely for certificate management
* Preserve IPv6 rather than disabling it as a workaround

---

## Final Architecture

### Remote Access

```text
Browser
   |
   v
https://proxmox.bechtfamily.org
   |
   v
Cloudflare Access
   |
   v
Cloudflare Tunnel
   |
   v
https://192.168.9.6:8006
   |
   v
Proxmox
```

Cloudflare Tunnel is configured with:

```text
Hostname:
proxmox.bechtfamily.org

Origin:
https://192.168.9.6:8006
```

Because Proxmox originally used its self-signed certificate, the Cloudflare Tunnel origin configuration used:

```text
No TLS Verify = ON
```

Cloudflare still provides a valid browser-facing HTTPS certificate.

---

## Local Access Design

Pi-hole provides split DNS inside the LAN.

Local DNS record:

```text
proxmox.bechtfamily.org -> 192.168.9.6
```

This means devices on the home network connect directly to Proxmox instead of traversing Cloudflare.

```text
Home Client
   |
   v
Pi-hole DNS
   |
   v
192.168.9.6
   |
   v
Proxmox :8006
```

---

## Initial Problem

After adding the Pi-hole local DNS record, IPv4 resolution worked correctly:

```powershell
nslookup -type=A proxmox.bechtfamily.org 192.168.9.4
```

Result:

```text
Name:    proxmox.bechtfamily.org
Address: 192.168.9.6
```

However, Microsoft Edge timed out while Firefox worked.

Further investigation showed that IPv6 lookups were still being forwarded upstream.

```powershell
Resolve-DnsName proxmox.bechtfamily.org
```

Returned:

```text
AAAA 2606:4700:3031::ac43:ac5e
AAAA 2606:4700:3033::6815:3fe5
A    192.168.9.6
```

The IPv6 addresses belonged to Cloudflare.

This created an inconsistent path:

```text
IPv4 -> 192.168.9.6 -> Local Proxmox
IPv6 -> Cloudflare -> Wrong path for :8006
```

Some browsers preferred IPv6 and timed out.

---

## Pi-hole DNS Issue

The primary Pi-hole was:

```text
192.168.9.4
```

Initially, DNS requests to this Pi-hole timed out.

Testing showed:

```powershell
nslookup google.com 192.168.9.4
```

failed, while the secondary Pi-hole worked:

```text
192.168.9.14
```

The Pi-hole interface setting was changed from:

```text
Allow only local requests
```

to:

```text
Permit all origins
```

After the change:

```powershell
nslookup google.com 192.168.9.4
```

worked normally.

This Pi-hole remains protected behind the LAN firewall and is not directly exposed to the Internet.

---

## Proxmox ACME / Let's Encrypt Certificate

Rather than using Nginx as a certificate proxy, Proxmox's built-in ACME support was used.

### ACME Account

In Proxmox:

```text
Datacenter
  -> ACME
  -> Accounts
  -> Add
```

Configuration:

```text
Account Name: default
ACME Directory: Let's Encrypt V2
```

The account registered successfully.

---

## Cloudflare API Token

A dedicated Cloudflare API token was created:

```text
proxmox-acme-dns01
```

Permissions:

```text
Zone -> DNS  -> Edit
Zone -> Zone -> Read
```

Resource restriction:

```text
Include
Specific zone
bechtfamily.org
```

No client IP restriction was configured because the token must continue working during automatic certificate renewal.

A dedicated token is preferred over the Cloudflare Global API Key.

---

## Proxmox Cloudflare DNS Challenge Plugin

In Proxmox:

```text
Datacenter
  -> ACME
  -> Challenge Plugins
  -> Add
```

Configuration:

```text
Plugin ID: cloudflare-dns
Validation Delay: 30
DNS API: Cloudflare Managed DNS
```

Credential method:

```text
CF_Account_ID=<Cloudflare Account ID>
CF_Token=<dedicated API token>
```

The following legacy fields were left blank:

```text
CF_Email=
CF_Key=
```

---

## ACME Domain Configuration

On the Proxmox node:

```text
pve
  -> System
  -> Certificates
  -> ACME
```

Domain added:

```text
Domain: proxmox.bechtfamily.org
Type: DNS
Plugin: cloudflare-dns
```

Then:

```text
Order Certificates Now
```

The ACME process successfully:

1. Created `_acme-challenge.proxmox.bechtfamily.org`
2. Waited for DNS propagation
3. Validated ownership with Let's Encrypt
4. Removed the temporary TXT record
5. Downloaded the certificate
6. Installed the certificate into `pveproxy`
7. Restarted `pveproxy`

Relevant successful output:

```text
Status is 'valid', domain 'proxmox.bechtfamily.org' OK!

All domains validated!

Downloading certificate
Setting pveproxy certificate and key
Restarting pveproxy
```

---

## Local AAAA Leak

Pi-hole contained the correct local IPv4 record:

```text
proxmox.bechtfamily.org -> 192.168.9.6
```

But AAAA requests continued upstream.

Testing:

```powershell
nslookup -type=AAAA proxmox.bechtfamily.org 192.168.9.4
```

returned Cloudflare IPv6 addresses.

The same behavior was observed with:

```text
plex.bechtfamily.org
```

---

## Pi-hole v6 Fix

Pi-hole FTL version:

```bash
pihole-FTL --version
```

Result:

```text
v6.7
```

Current DNS configuration was reviewed using:

```bash
pihole-FTL --config dns
```

The existing local host configuration included:

```text
192.168.9.6 proxmox.bechtfamily.org
```

To prevent unanswered AAAA queries for the local hostname from being forwarded to Cloudflare, the following dnsmasq directive was added through Pi-hole v6:

```bash
sudo pihole-FTL --config misc.dnsmasq_lines '["local=/proxmox.bechtfamily.org/"]'
```

Then Pi-hole FTL was restarted:

```bash
sudo systemctl restart pihole-FTL
```

---

## Verification

### IPv4

```powershell
nslookup -type=A proxmox.bechtfamily.org 192.168.9.4
```

Result:

```text
Name:    proxmox.bechtfamily.org
Address: 192.168.9.6
```

### IPv6

```powershell
nslookup -type=AAAA proxmox.bechtfamily.org 192.168.9.4
```

Result:

```text
No IPv6 address (AAAA) records available for proxmox.bechtfamily.org
```

This is the desired behavior.

---

## Windows DNS Cache

After the DNS fix:

```powershell
ipconfig /flushdns
```

Microsoft Edge was completely closed and reopened.

The following URL then worked normally:

```text
https://proxmox.bechtfamily.org:8006
```

with:

* No timeout
* No certificate warning
* Trusted Let's Encrypt certificate
* Local LAN routing

---

## Final Result

### At Home

```text
proxmox.bechtfamily.org
        |
        v
Pi-hole
        |
        v
192.168.9.6
        |
        v
Proxmox :8006
        |
        v
Let's Encrypt HTTPS
```

Traffic stays entirely on the LAN.

### Away From Home

```text
proxmox.bechtfamily.org
        |
        v
Cloudflare Access
        |
        v
Cloudflare Tunnel
        |
        v
Proxmox
```

No inbound Proxmox ports are exposed.

---

## Lessons Learned

A local Pi-hole A record alone does not necessarily prevent AAAA queries from being forwarded upstream.

When using split DNS with Cloudflare-proxied hostnames, verify both record types:

```powershell
Resolve-DnsName hostname
Resolve-DnsName hostname -Type A
Resolve-DnsName hostname -Type AAAA
```

Different browsers may select different address families, which can make an IPv6 DNS leak appear to be a browser-specific problem.

The clean solution was to fix DNS rather than:

* Disable IPv6
* Ignore certificate warnings
* Add unnecessary reverse proxies
* Expose Proxmox directly to the Internet
* Modify browser security settings

---

## Next Steps

Apply the same local-DNS treatment to:

```text
plex.bechtfamily.org
radarr.bechtfamily.org
sonarr.bechtfamily.org
homarr.bechtfamily.org
```

Then review old infrastructure associated with:

```text
192.168.9.50
```

and determine whether the previous Nginx Proxy Manager configuration and its Cloudflare API token can be retired.
