# 04: Greenbone Community Edition (OpenVAS) on Docker

An always-on vulnerability scanner that runs scheduled weekly scans of the home network.

## Architecture

```
Proxmox → Debian 13 VM (greenbone) → Docker Engine → ~20 Greenbone containers
                                                      └─ nginx serves the web UI (HTTPS)
```

Greenbone is distributed as about 20 single-purpose containers: PostgreSQL, Redis, gvmd, the scanner, feed-data loaders, nginx, and others. Docker Compose orchestrates them. Running Docker inside a dedicated VM keeps container upgrades simple while still isolating the scanner from the hypervisor.

## VM Configuration

| Setting | Value |
|---|---|
| OS | Debian 13 netinst. Headless: no desktop, only SSH server and standard utilities |
| Disk | 60 GB on `local-lvm`, Discard + SSD emulation |
| CPU | 2 cores, type `host` |
| RAM | 8 GB (4 GB minimum, ballooning) |
| NIC | VirtIO on `vmbr0`, firewall off |
| Options | Start at boot, QEMU guest agent |

The root password was left blank during install, so the first user gets `sudo` automatically.

## Docker

```bash
sudo apt update && sudo apt install -y curl qemu-guest-agent
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER      # log out and back in
docker run --rm hello-world        # sanity check
```

## Greenbone Deployment

Uses the official Greenbone Community Containers compose file:
```bash
mkdir -p ~/greenbone && cd ~/greenbone
curl -f -O -L https://greenbone.github.io/docs/latest/_static/compose.yaml

# Expose the web UI to the LAN (it binds to localhost by default)
sed -i 's/127.0.0.1:443:443/443:443/' compose.yaml

docker compose pull
docker compose up -d
docker compose exec -u gvmd gvmd gvmd --user=admin --new-password='<password>'
docker compose ps
```

Web UI: `https://<greenbone-ip>`. The first start loads the vulnerability feed, which can take an hour or more. Check **Administration → Feed Status** before scanning.

## Scheduled Scanning

| Object | Configuration |
|---|---|
| Schedule | Weekly, off-hours (Sunday 3:00 AM, America/Chicago) |
| Target | `192.168.1.0/24` |
| Task | Target above + **Full and fast** scan config + weekly schedule |

## Feed Updates (cron)

New vulnerability tests ship as updated container images. Pull them weekly, before the scan:
```cron
0 0 * * 0 docker compose -f /home/<user>/greenbone/compose.yaml pull && docker compose -f /home/<user>/greenbone/compose.yaml up -d
```

## Operating Notes

- Only networks I own are scanned.
- Scans can occasionally hang fragile IoT devices, which is why they run during off-hours.
- Findings are triaged High → Medium first. Low and Log entries are informational.
