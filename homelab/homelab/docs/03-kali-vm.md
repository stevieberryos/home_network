# 03: Kali Linux VM

A full KVM virtual machine for penetration-testing and network-analysis tooling, used only against networks I own.

## Why a VM (Not a Container)

Network tools like nmap need low-level access to network interfaces. A VM gives Kali its own kernel and strong isolation from the host.

## VM Configuration

| Setting | Value | Reason |
|---|---|---|
| Disk | 40 GB on `local-lvm`, Discard + SSD emulation | Thin provisioning reclaims deleted space |
| CPU | 2 cores, type `host` | Passes through the real CPU's feature set |
| RAM | 4 GB | Powered off when idle to free memory for Greenbone |
| NIC | VirtIO on `vmbr0`, Proxmox firewall **off** | The host firewall would distort scan results |
| Guest agent | Enabled | Clean shutdowns, IP shown in the UI, consistent backups |

## Installation

- Downloaded the Kali 64-bit installer ISO to `local` through **Download from URL**.
- Installed with the default Xfce desktop and default tool collection, using guided partitioning on the virtual disk.

## Post-Install

The installer couldn't reach a network mirror, so `/etc/apt/sources.list` was left empty. I added the official rolling repository:
```bash
echo "deb http://http.kali.org/kali kali-rolling main contrib non-free non-free-firmware" \
  | sudo tee /etc/apt/sources.list
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent
sudo systemctl start qemu-guest-agent
```
Then I took a `clean-install` snapshot as a known-good rollback point.

## Remote Access (Planned)

- LAN: `sudo systemctl enable --now ssh`, with key-based authentication only.
- Off-site: Tailscale (WireGuard-based) instead of forwarding SSH or RDP ports on the router.
