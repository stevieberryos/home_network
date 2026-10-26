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
