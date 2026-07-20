# Time Machine Backup to Synology DS923+

## Overview

This document describes how Time Machine is configured to back up my Mac Studio to a Synology DS923+ running DSM 7.3.2.

## Environment

| Component | Value |
|-----------|-------|
| NAS | Synology DS923+ |
| DSM Version | 7.3.2 |
| Backup Protocol | SMB |
| Client | Apple Mac Studio |
| Backup Software | Apple Time Machine |
| Destination | TimeMachine Shared Folder |

## Configuration

### Shared Folder

- Name: **TimeMachine**
- SMB Enabled
- Dedicated backup share
- 3.22 TB quota

### User Account

A dedicated **timemachine** user was created with read/write access only to the TimeMachine shared folder.

This follows the principle of least privilege and keeps administrative credentials separate from backup operations.

## Backup Process

1. Create a dedicated shared folder.
2. Enable SMB.
3. Enable Bonjour Time Machine broadcast.
4. Select the TimeMachine shared folder.
5. Configure a dedicated backup account.
6. Connect macOS Time Machine to the NAS.

## Current Status

- ✅ Initial backup completed successfully
- ✅ Incremental backups enabled
- ✅ Backups stored on Synology DS923+
- ✅ Network backups protected by a storage quota

## Lessons Learned

- Use a dedicated backup account instead of an administrator account.
- Configure a storage quota to prevent backups from consuming all available disk space.
- Test restore functionality after the initial backup completes.
- Time Machine backups are incremental after the first full backup, significantly reducing backup times.

## Future Improvements

- Configure Snapshot Replication for additional protection.
- Include the Time Machine share in the NAS backup strategy.
- Periodically verify restore capability.
