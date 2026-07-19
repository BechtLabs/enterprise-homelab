# Proxmox VE

This section documents the Proxmox infrastructure used in the Enterprise Homelab.

# Proxmox VE

## Overview

The Enterprise Homelab is built on Proxmox VE running on a Minisforum MS-01. Proxmox provides the virtualization platform for all virtual machines, Linux containers, and supporting infrastructure services.

## Platform

| Component | Details |
|-----------|---------|
| Host | Minisforum MS-01 |
| Hypervisor | Proxmox VE |
| Network | 10Gb Ethernet |
| Backup | Proxmox Backup Server (PBS) |

## Primary Virtual Machines

| VM ID | Name | Purpose | IP Address |
|------:|------|---------|------------|
| 300 | Ubuntu Docker | Primary Docker host | 192.168.9.200 |
| TBD | PBS | Backup Server | 192.168.9.10 |
| TBD | Kali Linux | Security lab | DHCP |

## Current Goals

- Standardize virtual machine deployments
- Document every service
- Automate backups
- Improve disaster recovery
- Maintain enterprise-level documentation
