# Troubleshooting Log

Real problems hit during the build, and how each was diagnosed and fixed.

## 1. Windows: "An extended error has occurred" mapping the share

**Symptom:** Mapping `\\<ip>\hddstorage` failed with a generic error.

**Diagnosis:**
- `testparm -s | grep '^\['` showed that the share was named `[storage]`, not `hddstorage`. The share name and the underlying folder name are independent.
- `smbclient //localhost/storage -U <user>` returned `NT_STATUS_ACCESS_DENIED`, which pointed to a credentials problem as well.

**Fix:**
- Reset the Samba password with `smbpasswd -a <user>`, since a typo had been made during setup.
- Verified locally with `smbclient` before going back to Windows.
- Cleared cached sessions with `net use * /delete`, then mapped `\\<ip>\storage` with explicit credentials.

**Lesson:** Test the server side in isolation (`smbclient` on localhost) before debugging the client.

## 2. iPhone could read the share but not write to it

**Symptom:** Windows could write, but the iOS Files app's Save button was grayed out.

**Diagnosis:** Server permissions were fine, because Windows could write. iOS checks for Apple's SMB extensions before writing, and plain Samba doesn't advertise them.

**Fix:** Enabled Samba's `vfs_fruit` and `streams_xattr` modules in `[global]` (see [02-file-server.md](02-file-server.md)), restarted `smbd`, and reconnected the iPhone.

## 3. Unprivileged LXC permissions (UID mapping)

**Risk:** Unprivileged containers shift UIDs by 100000, so container UID 1000 appears on the host as UID 101000. Files owned by host root are not writable from inside the container.

**Fix:** `chown -R 101000:101000 /mnt/pve/hddstorage` on the host. Any folder Proxmox itself creates later needs the same treatment.

## 4. "Current guest config does not support taking new snapshots"

**Cause:** Proxmox can't snapshot bind mounts, so the HDD bind mount blocks snapshots for the whole container.

**Resolution:** Use vzdump backups for the container, which skip bind mounts automatically. VMs without bind mounts (Kali, Greenbone) snapshot normally.

## 5. Kali: "You don't have any APT data sources configured"

**Cause:** The installer couldn't use a network mirror during install, so it installed from the ISO only and left `/etc/apt/sources.list` empty.

**Fix:**
```bash
echo "deb http://http.kali.org/kali kali-rolling main contrib non-free non-free-firmware" \
  | sudo tee /etc/apt/sources.list
sudo apt update && sudo apt full-upgrade -y
```
**Prevention:** On the Debian VM install, I deliberately answered **Yes** to "Use a network mirror?"

## 6. Proxmox recreating folders inside the file share

**Cause:** The HDD was registered as Proxmox storage, so Proxmox kept creating `images/`, `dump/`, `template/`, and similar folders in the share.

**Fix:** Removed the storage registration (Datacenter → Storage → Remove). The disk stays mounted through its systemd unit. Then deleted the empty folders from the host, keeping `lost+found`.

## 7. Docker image pulls failing with "connection reset by peer"

**Symptom:** `docker compose pull` for Greenbone repeatedly failed partway through. Errors included `read tcp [2600:...]->[2a0e:...]:443: connection reset by peer` and `timeout awaiting response headers`, and the remaining downloads were cancelled ("Interrupted").

**Diagnosis:** Every failure was on an **IPv6** connection to Greenbone's container registry, and the largest image (`scap-data`, about 1.6 GB) kept being cut off. The IPv6 route to the registry was unreliable.

**Fix:** Disabled IPv6 on the scanner VM, made the setting persistent, and limited Docker's parallel downloads:
```bash
echo -e "net.ipv6.conf.all.disable_ipv6 = 1\nnet.ipv6.conf.default.disable_ipv6 = 1" \
  | sudo tee /etc/sysctl.d/99-disable-ipv6.conf
sudo sysctl --system
echo '{ "max-concurrent-downloads": 2 }' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
docker compose pull      # resumes; finished layers are kept
```

## 8. Greenbone scan found hosts but zero open ports

**Symptom:** The first full LAN scan reported **7 live hosts, 0 ports**, and only a Low "ICMP Timestamp Reply" finding. Each host finished in about 5 minutes. A "clean" result like that looked suspicious, because the Proxmox host is known to have ports 22 and 8006 open.

**Diagnosis (ruling things out layer by layer):**
1. **Report filter?** Showed all severity levels including Log, with minimum QoD set to 0. Still 0 ports, so the results weren't just hidden.
2. **Network path from the VM?** `nc -zv 192.168.1.65 8006` from the Greenbone VM: **open**.
3. **Network path from inside the scanner container?**
   ```bash
   docker compose exec ospd-openvas bash -c "timeout 3 bash -c '</dev/tcp/192.168.1.65/8006' && echo OPEN || echo BLOCKED"
   ```
   **OPEN**, so Docker networking was fine and the fault was inside Greenbone.
4. **Scanner logs:**
   ```
   openvas:  /var/lib/openvas/plugins/plugin_feed_info.inc: Checksum check failed
   ospd-openvas: OpenVAS Scanner failed to load VTs ... returned non-zero exit status 1
   ```

**Root cause:** The scanner verifies its vulnerability tests (VTs) against a signed checksum list before loading them. The containers had first started while feed data was still being copied into place after the interrupted pulls (see #7), so the check failed and most VTs never loaded. Without them, the scanner could detect live hosts but couldn't run port discovery or service checks. The bad state persisted across restarts because it was held in the scanner's **Redis** cache on a Docker volume.

**Fix:** Cleared the Redis volume to force a clean reload:
```bash
cd ~/greenbone
docker compose down
docker volume rm greenbone-community-edition_redis_socket_vol
docker compose up -d
docker compose logs -f ospd-openvas            # wait for "Finished loading VTs" with no errors
docker compose logs openvas | grep -i checksum # no output = clean
```
Then verified with a single-host test scan of `192.168.1.65`, which should show ports 22 and 8006.

**Lesson:** A clean vulnerability scan isn't proof of a clean network. When a scan finds hosts but no services, suspect the scanner first. Check its logs, and test reachability from inside its container, to separate scanner failures from real results.
