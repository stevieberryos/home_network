# 02: Network File Server (Samba in LXC)

A lightweight LXC container running Samba shares the 1 TB HDD to Windows and iOS clients on the LAN.

## Why a Container

- Samba is a simple Linux service, so a full VM would waste RAM. The container idles at well under 1 GB.
- TrueNAS was considered and rejected for now. With a single disk, its ZFS redundancy features don't apply, and it wants about 8 GB of RAM.

## Build Steps

### 1. Template and container
- Downloaded the `turnkey-fileserver` template to `local` (CT Templates).
- Created **CT 100** (`fileserver`): unprivileged, 8 GB root disk on `local-lvm`, 1 GB RAM, DHCP on `vmbr0` with a router reservation.
- Ran the TurnKey first-boot wizard: set the Samba root password, skipped the Hub/backup steps, and installed security updates.

### 2. Bind-mount the HDD into the container
Run on the Proxmox host:
```bash
pct set 100 -mp0 /mnt/pve/hddstorage,mp=/mnt/hddstorage
```

### 3. Share user
Run inside the container:
```bash
adduser <user>
smbpasswd -a <user>
id <user>          # UID 1000
```

### 4. Fix ownership for unprivileged UID mapping
An unprivileged container's UID 1000 maps to UID **101000** on the host (a 100000 offset). Run on the host:
```bash
chown -R 101000:101000 /mnt/pve/hddstorage
```

### 5. Share definition (`/etc/samba/smb.conf`)
TurnKey ships a guest-accessible `[storage]` share. I repointed it at the HDD and required authentication:
```ini
[storage]
   comment = Public Share
   path = /mnt/hddstorage
   browseable = yes
   read only = no
   create mask = 0644
   directory mask = 0755
   valid users = <user>
```

### 6. Apple (iOS) compatibility, in `[global]`
```ini
vfs objects = fruit streams_xattr recycle
fruit:metadata = stream
fruit:model = MacSamba
fruit:posix_rename = yes
fruit:veto_appledouble = no
fruit:wipe_intentionally_left_blank_rfork = yes
fruit:delete_empty_adfiles = yes
```
Without these settings, the iOS Files app treated the share as read-only. The `fruit` module implements Apple's SMB extensions, and `streams_xattr` stores Apple metadata as ext4 extended attributes.

Validate and apply:
```bash
testparm -s > /dev/null && systemctl restart smbd
```

## Clients

| Client | How |
|---|---|
| Windows | File Explorer → Map network drive → `\\<fileserver-ip>\storage`, with "Connect using different credentials" checked |
| iPhone | Files → Connect to Server → `smb://<fileserver-ip>/storage` → Registered User |
| Android | An SMB-capable file manager (CX File Explorer, Material Files) |

## Notes

- **Snapshots are unavailable.** Proxmox can't snapshot a container that has a bind mount. Container-level backups (vzdump) are used instead, and these automatically exclude the bind-mounted HDD.
- **Start at boot** is enabled.
