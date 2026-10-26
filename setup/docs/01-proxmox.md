# 01: Proxmox VE Host

Proxmox VE is the bare-metal operating system. It is a Debian-based hypervisor that runs KVM virtual machines and LXC containers, all managed from a web UI.

## Installation

1. Enabled hardware virtualization (VT-x/AMD-V) in the BIOS.
2. Flashed the Proxmox VE ISO to a USB stick and installed it to the 256 GB SSD.
3. Reserved a fixed IP for the host in the router (DHCP reservation).
4. Logged into the web UI at `https://<proxmox-ip>:8006`.

## Storage Layout

| Storage ID | Type | Location | Holds |
|---|---|---|---|
| `local` | Directory | SSD (`/var/lib/vz`) | ISO images, container templates, backups |
| `local-lvm` | LVM-thin | SSD | VM and container virtual disks |
| HDD | ext4, mounted at `/mnt/pve/hddstorage` | 1 TB HDD | Network file share (bind-mounted into the file server) |

**Why two kinds of SSD storage?**
- `local` is a regular folder, so it holds *files* like ISOs and templates.
- `local-lvm` is a thin-provisioned block pool that Proxmox divides into virtual disks. Space is only consumed as guests actually write data.

## Adding the HDD

1. **Node → Disks**: identified the HDD and wiped old partitions.
2. **Disks → Directory → Create**: formatted it as ext4 and named it `hddstorage`, which mounts it at `/mnt/pve/hddstorage`.
3. Removed the Proxmox storage *registration* afterward (Datacenter → Storage → Remove). The disk stays mounted through its systemd mount unit, but Proxmox no longer creates its own `images/`, `dump/`, and `template/` folders inside the file share.

## Practices

- **Start at boot** is enabled for always-on guests (the file server and Greenbone).
- ISOs are deleted from `local` after installs to conserve SSD space.
- Guests have the QEMU guest agent installed. It enables clean shutdowns, IP reporting in the UI, and consistent backups through filesystem freeze.
