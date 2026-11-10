# Wheel Income Ledger: Proxmox Install Walkthrough

How I deployed my self-hosted trade ledger on my Proxmox homelab, with an explanation of what every command does and why it is needed.

**What this builds:** a small Debian container (LXC) on Proxmox that runs Docker, which in turn runs the Wheel Income Ledger, a FastAPI and SQLite web app for tracking closed options trades. The app is reachable from any device on the home network at `http://<container-ip>:8000`.

**Placeholders used below.** Replace these with your own values. Real addresses, usernames and share names are deliberately left out of this document.

| Placeholder | Meaning |
|---|---|
| `<container-ip>` | IP address of the new ledger container |
| `<samba-server-ip>` | IP address of the Samba file server container |
| `<share-name>` | Name of the Samba share |
| `<samba-username>` | Samba account used to reach the share |

## Contents

1. [Architecture](#1-architecture)
2. [Create the container in the Proxmox GUI](#2-create-the-container-in-the-proxmox-gui)
3. [Find the Samba share details](#3-find-the-samba-share-details)
4. [Pull the project from the Samba share](#4-pull-the-project-from-the-samba-share)
5. [Install Docker](#5-install-docker)
6. [Unpack the project](#6-unpack-the-project)
7. [Prepare the data folder](#7-prepare-the-data-folder)
8. [Build and start the app](#8-build-and-start-the-app)
9. [Verify it works](#9-verify-it-works)
10. [Day-to-day operations](#10-day-to-day-operations)
11. [Security notes](#11-security-notes)

## 1. Architecture

```
Proxmox VE host
├── Samba LXC            file server; used here to hand the project zip to the new container
└── wheel-ledger LXC     unprivileged Debian 12 container
    └── Docker
        └── wheel-ledger container (Python 3.12, FastAPI, uvicorn) on port 8000
            └── /data/ledger.db   SQLite file, stored on the LXC's disk via a bind mount
```

Why two layers of container: the LXC gives the app its own IP address, resource limits and snapshot/backup unit inside Proxmox. Docker inside it makes the app itself reproducible, because the `Dockerfile` pins the Python version and dependencies, so rebuilding always produces the same environment.

## 2. Create the container in the Proxmox GUI

No commands in this step; everything is done in the Proxmox web interface.

### 2.1 Download the template

`local` storage > **CT Templates** > **Templates** > `debian-12-standard` > **Download**.

A template is a prebuilt root filesystem. LXC containers share the host's kernel, so they do not boot an installer the way a VM does. They start from a template instead.

### 2.2 Create CT

| Tab | Setting | Why |
|---|---|---|
| General | Hostname `wheel-ledger`, root password set | Name shown in Proxmox and on the network |
| General | **Unprivileged container** ticked | Root inside the container maps to an unprivileged user on the host, so a compromise of the container does not hand over the host |
| General | **Nesting** ticked | Lets the container create its own namespaces, which Docker needs |
| Template | Debian 12 | Stable base with long support |
| Disks | 8 GB | Enough for Docker, one Python image and the database |
| CPU | 1 core | The app is a single lightweight process |
| Memory | 1024 MB, swap 512 MB | Mostly needed while Docker builds the image |
| Network | Bridge `vmbr0`, IPv4 DHCP | Puts the container on the LAN like any other device |

### 2.3 Options

- **Features > keyctl** ticked. Docker uses the kernel keyring; in an unprivileged container that system call is blocked unless this feature is on. Changing it requires being logged in as `root@pam`.
- **Start at boot > Yes**. The ledger comes back by itself after a host reboot.

Then **Start** the container and open its **Console**. Every command from section 4 onward is run there, as root.

## 3. Find the Samba share details

I needed the share name and username to pull the zip. Two places to look.

### On the Windows computer that has the share mapped

```
net use
```

Lists every network connection the computer currently has. The **Remote** column shows mapped drives as `\\<samba-server-ip>\<share-name>`, which gives both the server address and the share name.

```
cmdkey /list
```

Lists the logins saved in Windows Credential Manager. The entry whose target matches the Samba server shows the saved **User**, which is the Samba username. Passwords are not displayed.

### On the Samba container itself

```bash
testparm -s 2>/dev/null | grep '^\['
```

- `testparm` validates Samba's configuration file (`/etc/samba/smb.conf`) and prints the effective settings.
- `-s` skips the "press enter" prompt so it can be piped.
- `2>/dev/null` hides the status messages it writes to standard error.
- `grep '^\['` keeps only lines that start with `[`, which are the section headers. `[global]`, `[homes]` and `[printers]` are built in; any other name is a share.

```bash
pdbedit -L
```

Lists the accounts in Samba's own user database. Samba keeps its passwords separately from Linux system passwords, so this is the authoritative list of who can log in to a share.

```bash
smbpasswd <samba-username>
```

Only needed if the password is forgotten. Sets a new Samba password for that account. Every device that connects to the share then needs the new password.

## 4. Pull the project from the Samba share

First I copied `wheel-ledger.zip` onto the share from my desktop using the mapped drive. Then, in the ledger container's console:

```bash
apt update && apt install -y smbclient
```

- `apt update` refreshes the list of available packages. A fresh template has a stale or empty list, so installs fail without it.
- `&&` runs the second command only if the first succeeded.
- `apt install -y smbclient` installs a command-line Samba client. `-y` answers "yes" to the confirmation prompt.

`smbclient` works like an FTP client for Windows shares. It runs entirely as a normal program, so it avoids mounting the share, which an unprivileged container is not allowed to do.

```bash
cd /root
smbclient //<samba-server-ip>/<share-name> -U <samba-username> -c 'get wheel-ledger.zip'
```

- `cd /root` moves to root's home directory so the download lands there.
- `//<samba-server-ip>/<share-name>` is the share to connect to (the Linux spelling of `\\server\share`).
- `-U <samba-username>` is the account to log in with. It prompts for the password, so the password never appears in shell history.
- `-c '...'` runs the quoted commands and exits instead of opening an interactive prompt.
- `get wheel-ledger.zip` downloads that file into the current directory.

If the zip is in a subfolder of the share: `-c 'cd <folder>; get wheel-ledger.zip'`.

## 5. Install Docker

```bash
apt install -y curl unzip ca-certificates
```

- `curl` downloads files and makes HTTP requests. Used to fetch the Docker installer and later to test the app.
- `unzip` extracts the project archive.
- `ca-certificates` is the set of trusted certificate authorities, so HTTPS downloads can be verified.

```bash
curl -fsSL https://get.docker.com | sh
```

Downloads Docker's official install script and runs it.

- `-f` makes curl fail on an HTTP error instead of saving an error page.
- `-s` hides the progress meter; `-S` still shows errors.
- `-L` follows redirects.
- `| sh` feeds the downloaded script straight to the shell.

The script detects Debian, adds Docker's package repository and signing key, and installs the Docker engine plus the `docker compose` plugin. Piping a script from the internet into a shell means trusting its source completely; this one is published by Docker. The cautious alternative is to download it to a file, read it, then run it.

```bash
docker --version && docker compose version
```

Confirms both the engine and the compose plugin installed.

## 6. Unpack the project

```bash
cd /root && unzip wheel-ledger.zip && cd wheel-ledger
```

Extracts the archive into `/root/wheel-ledger` and moves into it. Later commands must run from this folder because `docker compose` looks for `docker-compose.yml` in the current directory.

What is inside:

| Path | Purpose |
|---|---|
| `app/main.py` | HTTP routes (FastAPI) |
| `app/db.py` | SQLite storage |
| `app/schwab.py` | Parser that turns a Schwab export into closed trades |
| `app/static/index.html` | The web page, one file, no build step |
| `Dockerfile` | Recipe for the app's image |
| `docker-compose.yml` | How to run that image: port, volume, restart policy, settings |
| `requirements.txt` | Python dependencies |
| `tests/` | Parser tests |

## 7. Prepare the data folder

```bash
mkdir -p data && chown 1000:1000 data
```

- `mkdir -p data` creates the folder that will hold the database. `-p` means no error if it already exists.
- `chown 1000:1000 data` gives ownership to user ID 1000 and group ID 1000.

Why this matters: the `Dockerfile` creates a non-root user with ID 1000 and runs the app as that user, which limits the damage if the app is ever compromised. `docker-compose.yml` maps `./data` on the LXC to `/data` inside the Docker container. File permissions are checked by numeric ID, so the folder must be owned by 1000 or the app cannot create `ledger.db` and fails with "permission denied".

Keeping the database in a mapped folder, outside the Docker container, is what lets the app be rebuilt or replaced without losing any trades.

## 8. Build and start the app

```bash
docker compose up -d --build
```

- `docker compose` reads `docker-compose.yml` in the current folder.
- `up` creates and starts the services defined there.
- `--build` builds the image from the `Dockerfile` first, instead of looking for an existing one.
- `-d` (detached) runs it in the background and returns the prompt.

What the build does, following the `Dockerfile`:

1. Starts from the official `python:3.12-slim` image.
2. Installs FastAPI (the web framework) and uvicorn (the server that runs it) from `requirements.txt`.
3. Copies the `app/` folder in.
4. Creates the unprivileged user and the `/data` folder.
5. Sets the start command: `uvicorn app.main:app --host 0.0.0.0 --port 8000`.

What `docker-compose.yml` adds at run time:

- `ports: "8000:8000"` publishes the app's port 8000 on the LXC's port 8000. The left number is the outside port and can be changed if 8000 is taken.
- `volumes: ./data:/data` is the bind mount from section 7.
- `restart: unless-stopped` restarts the app after a crash or a reboot, unless it was stopped on purpose.
- `LEDGER_EXCLUDE_TICKERS` lists tickers whose share sales start unticked on the import screen.

## 9. Verify it works

```bash
curl http://localhost:8000/api/health
```

Requests the app's health endpoint from inside the LXC. A working install answers `{"ok":true}`. This proves the container is running, the port is published and the app started without errors.

```bash
hostname -I
```

Prints the LXC's IP addresses. The first one is `<container-ip>`. Docker adds its own internal addresses (172.x.x.x) to the list; ignore those.

Then, from any device on the network, open `http://<container-ip>:8000`.

First use:

1. **Import Schwab file** and choose a transaction export (Schwab: History > Export > JSON).
2. Tick the trades to keep and press **Import selected**.
3. Set the capital base in the Monthly target panel.

Finally, add a DHCP reservation for the container in the router so `<container-ip>` does not change.

## 10. Day-to-day operations

All run from `/root/wheel-ledger` in the LXC console.

| Command | What it does |
|---|---|
| `docker compose ps` | Shows whether the app is running and which ports are published |
| `docker compose logs` | Prints the app's output; the first place to look when something fails |
| `docker compose logs -f` | Same, but keeps following new lines (Ctrl+C to stop) |
| `docker compose restart` | Restarts the app without rebuilding |
| `docker compose down` | Stops and removes the app container; the database in `data/` is untouched |
| `docker compose up -d --build` | Rebuilds and starts; run this after replacing the project files with a new version |

**Updating:** put the new zip on the Samba share, repeat the `smbclient` line from section 4, run `unzip -o wheel-ledger.zip` (the `-o` overwrites existing files without asking), then `docker compose up -d --build`.

**Backing up:** the whole ledger is the single file `data/ledger.db`. Either copy it somewhere safe or back up the entire LXC with Proxmox's built-in backup, which captures the app and its data together.

## 11. Security notes

- The app has no login. Anyone who can reach port 8000 can read and change the ledger, so it stays on the LAN only. No port forwarding.
- Remote access goes through a VPN (WireGuard or Tailscale), which extends the LAN instead of exposing the app.
- The LXC is unprivileged and the app runs as a non-root user inside Docker, so there are two layers between the app and root on the Proxmox host.
- Schwab exports contain account details and a full transaction history. They are excluded from version control by the project's `.gitignore` and should not be committed to this repository.
