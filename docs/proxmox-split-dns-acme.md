# Proxmox HTTPS with Let's Encrypt, Cloudflare DNS, and Pi-hole Split DNS

## Overview

This guide documents how to configure a Proxmox VE server with a trusted Let's Encrypt certificate while keeping local management traffic on the LAN.

The design uses:

- Proxmox VE built-in ACME support
- Let's Encrypt
- Cloudflare DNS
- Cloudflare API token authentication
- Pi-hole local DNS
- DNS-01 certificate validation
- Split DNS for local access

The result is a Proxmox interface accessible locally using:

```text
https://proxmox.example.com:8006
```

without browser certificate warnings.

> **Security Note:** All hostnames, domains, IP addresses, account names, and other environment-specific values in this document are examples. Replace them with values appropriate for your environment.

---

## Architecture

### Local Access

```text
LAN Client
    |
    v
Pi-hole DNS
    |
    | proxmox.example.com
    | -> 192.168.10.10
    v
Proxmox VE
    |
    v
TCP 8006 / HTTPS
```

Because Pi-hole resolves the hostname directly to the private Proxmox address, local traffic remains on the LAN.

### Remote Access

If Cloudflare Access and Cloudflare Tunnel are also being used, remote access can follow a separate path:

```text
Remote Client
      |
      v
Cloudflare Access
      |
      v
Cloudflare Tunnel
      |
      v
Proxmox VE
```

This allows remote access without exposing Proxmox TCP port 8006 directly to the Internet.

---

# 1. Why ACME?

A default Proxmox installation uses a self-signed TLS certificate.

The encryption itself works, but browsers do not trust the certificate authority that issued it.

This commonly produces warnings such as:

```text
Your connection is not private
```

Rather than disabling certificate validation or accepting permanent browser exceptions, Proxmox can obtain a trusted certificate directly from Let's Encrypt.

Proxmox includes native support for ACME.

---

# 2. Register a Proxmox ACME Account

Navigate to:

```text
Datacenter
  -> ACME
  -> Accounts
  -> Add
```

Example configuration:

```text
Account Name: default
ACME Directory: Let's Encrypt V2
```

Provide a valid email address and accept the Let's Encrypt Terms of Service.

Register the account.

A successful registration should end with:

```text
TASK OK
```

---

# 3. Create a Cloudflare API Token

The DNS-01 challenge requires Proxmox to temporarily create a DNS TXT record.

A dedicated Cloudflare API token should be used instead of the Cloudflare Global API Key.

Create a token with the following permissions:

```text
Zone -> DNS  -> Edit
Zone -> Zone -> Read
```

Restrict the token to only the required DNS zone:

```text
Zone Resources

Include
Specific zone
example.com
```

A descriptive token name could be:

```text
proxmox-acme
```

Do not place the actual token in documentation, scripts, Git repositories, screenshots, or configuration examples.

---

# 4. Configure the Cloudflare DNS Challenge Plugin

In Proxmox navigate to:

```text
Datacenter
  -> ACME
  -> Challenge Plugins
  -> Add
```

Example:

```text
Plugin ID: cloudflare-dns
Validation Delay: 30
DNS API: Cloudflare Managed DNS
```

Use API-token authentication rather than the legacy Global API Key.

Example configuration:

```text
CF_Account_ID=<CLOUDFLARE_ACCOUNT_ID>
CF_Token=<CLOUDFLARE_API_TOKEN>
```

Leave legacy Global API Key fields empty when using token authentication.

For example:

```text
CF_Email=
CF_Key=
```

---

# 5. Configure the Proxmox ACME Domain

Select the Proxmox node and navigate to:

```text
System
  -> Certificates
```

Under **ACME**, add a domain.

Example:

```text
Domain: proxmox.example.com
Type: DNS
Plugin: cloudflare-dns
```

The DNS challenge is important because the Proxmox server does not need to be publicly reachable for Let's Encrypt validation.

---

# 6. Request the Certificate

Select:

```text
Order Certificates Now
```

Proxmox will begin the ACME process.

During validation, it temporarily creates a TXT record similar to:

```text
_acme-challenge.proxmox.example.com
```

The general process is:

```text
Proxmox
   |
   | Cloudflare API
   v
Create temporary TXT record
   |
   v
Let's Encrypt checks DNS
   |
   v
Domain ownership validated
   |
   v
Certificate issued
   |
   v
TXT record removed
   |
   v
Certificate installed
```

Successful output should contain messages similar to:

```text
Status is 'valid'
All domains validated!

Downloading certificate
Setting pveproxy certificate and key
Restarting pveproxy
```

At this point Proxmox is serving the Let's Encrypt certificate.

---

# 7. Configure Pi-hole Split DNS

For local access, configure Pi-hole to resolve the Proxmox hostname directly to its private IP address.

Navigate to:

```text
Pi-hole
  -> Local DNS
  -> DNS Records
```

Create:

```text
Domain:
proxmox.example.com

IP:
192.168.10.10
```

The resulting local DNS behavior should be:

```text
proxmox.example.com
        |
        v
192.168.10.10
```

This prevents local clients from unnecessarily leaving the LAN to reach the Proxmox server.

---

# 8. Verify IPv4 Resolution

From a client using Pi-hole for DNS:

```powershell
nslookup -type=A proxmox.example.com <PIHOLE_IP>
```

Expected result:

```text
Name:    proxmox.example.com
Address: 192.168.10.10
```

Another useful test is:

```powershell
Resolve-DnsName proxmox.example.com -Type A
```

The hostname should resolve to the private LAN address.

---

# 9. Watch for AAAA / IPv6 Leakage

An interesting issue can occur when split DNS overrides only the IPv4 `A` record.

For example:

```text
A     -> 192.168.10.10
AAAA  -> Public IPv6 address
```

Some browsers and operating systems may prefer IPv6.

That can result in one browser successfully reaching Proxmox while another browser times out.

Check IPv6 resolution explicitly:

```powershell
Resolve-DnsName proxmox.example.com -Type AAAA
```

or:

```powershell
nslookup -type=AAAA proxmox.example.com <PIHOLE_IP>
```

If the local service is intended to be IPv4-only, public AAAA records should not leak through the local DNS override.

---

# 10. Pi-hole v6 Solution

The following approach applies to Pi-hole v6.

Check the installed FTL version:

```bash
pihole-FTL --version
```

Review the current DNS configuration:

```bash
pihole-FTL --config dns
```

Pi-hole may contain a local host record similar to:

```text
192.168.10.10 proxmox.example.com
```

However, an unanswered AAAA query may still be forwarded to an upstream DNS provider.

A dnsmasq local-zone directive can prevent that behavior.

Example:

```bash
sudo pihole-FTL --config misc.dnsmasq_lines \
'["local=/proxmox.example.com/"]'
```

Restart Pi-hole FTL:

```bash
sudo systemctl restart pihole-FTL
```

---

# 11. Verify the AAAA Fix

Test again:

```powershell
nslookup -type=AAAA proxmox.example.com <PIHOLE_IP>
```

For an IPv4-only local service, the desired result is:

```text
No IPv6 address (AAAA) records available for proxmox.example.com
```

Then verify IPv4 still works:

```powershell
nslookup -type=A proxmox.example.com <PIHOLE_IP>
```

Expected:

```text
Name:    proxmox.example.com
Address: 192.168.10.10
```

The important result is:

```text
A     -> Private LAN address
AAAA  -> No public answer
```

---

# 12. Flush Client DNS Cache

Clients may continue using cached DNS records after the server-side configuration has been corrected.

On Windows:

```powershell
ipconfig /flushdns
```

Completely close and reopen the browser.

Then connect to:

```text
https://proxmox.example.com:8006
```

The browser should now connect directly to the Proxmox server using the trusted Let's Encrypt certificate.

---

# Troubleshooting

## One Browser Works but Another Times Out

Do not immediately assume the browser is the problem.

Compare IPv4 and IPv6 DNS resolution:

```powershell
Resolve-DnsName proxmox.example.com -Type A
Resolve-DnsName proxmox.example.com -Type AAAA
```

If IPv4 points to the LAN while IPv6 points somewhere else, the browsers may simply be choosing different network paths.

---

## Test IPv4 Directly

```bash
curl -4 -I https://proxmox.example.com:8006
```

A response from Proxmox confirms:

- DNS resolution works
- TCP 8006 is reachable
- TLS negotiation succeeds
- The Proxmox web service is responding

Note that Proxmox may return an HTTP error for a `HEAD` request while still proving that connectivity and TLS are functioning.

---

## Test IPv6 Directly

```bash
curl -6 -I https://proxmox.example.com:8006
```

If IPv6 is not configured for the local service, this should not resolve to an unrelated public destination.

---

# Security Considerations

## Use Least-Privilege API Tokens

The Cloudflare token only needs:

```text
Zone -> DNS  -> Edit
Zone -> Zone -> Read
```

Restrict it to the specific DNS zone used for ACME.

Avoid using the Global API Key.

---

## Do Not Publish Secrets

Never commit:

```text
Cloudflare API tokens
Cloudflare Global API Keys
Account IDs unnecessarily
Private keys
ACME account keys
Tunnel credentials
Real internal hostnames/IP addresses unnecessarily
```

Use placeholders in public documentation.

---

## Do Not Expose Proxmox Port 8006

A trusted TLS certificate does not make direct Internet exposure desirable.

For remote administration, use an appropriate secure access method such as:

```text
VPN
Zero-trust access
Cloudflare Access/Tunnel
Other authenticated private-access solution
```

---

# Final Architecture

## Inside the LAN

```text
Client
  |
  | DNS query
  v
Pi-hole
  |
  | proxmox.example.com
  | -> 192.168.10.10
  v
Proxmox VE :8006
  |
  v
Let's Encrypt TLS
```

## Outside the LAN

One possible architecture:

```text
Remote Client
      |
      v
Cloudflare Access
      |
      v
Cloudflare Tunnel
      |
      v
Proxmox VE
```

No direct inbound connection to TCP 8006 is required.

---

# Key Takeaways

1. Proxmox can manage its own Let's Encrypt certificates using ACME.

2. A reverse proxy is not required solely to obtain a trusted certificate for the Proxmox management interface.

3. Cloudflare DNS-01 validation allows certificate issuance without exposing Proxmox to the Internet.

4. Pi-hole split DNS keeps local management traffic on the LAN.

5. Always test both `A` and `AAAA` DNS records when troubleshooting inconsistent browser behavior.

6. A public AAAA response can cause IPv6-capable clients to take a completely different path than IPv4 clients.

7. Fix the DNS architecture rather than disabling IPv6 as a troubleshooting shortcut.

8. Dedicated, least-privilege API tokens are preferable to account-wide API keys.

---

## Environment Used

This procedure was tested with:

```text
Proxmox VE 9.x
Pi-hole 6.x
Cloudflare DNS
Let's Encrypt ACME v2
Windows client
```

Exact versions and network addresses are intentionally omitted from this public documentation.
