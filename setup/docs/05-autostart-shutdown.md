# Proxmox Scheduled Shutdown & Auto-Wake

Automates a nightly shutdown of the Proxmox VE host and a scheduled power-on the next morning, using `cron` for the shutdown and the motherboard's real-time clock (RTC) alarm for the wake-up.

**Example schedule used:** shut down at **12:00 AM**, wake at **7:30 AM**, every day.

> **Status:** Implemented and later removed. The removal steps are at the bottom.

---

## How it works

1. At the scheduled time, `cron` runs a small script on the Proxmox host.
2. The script uses `rtcwake` to write a wake-up time into the motherboard's RTC alarm.
3. The script then performs a clean `shutdown`. Proxmox stops all VMs and containers gracefully first.
4. At the alarm time, the motherboard powers the machine back on. Proxmox boots and starts any guests set to **Start at boot**.

The wake alarm is set only when the script runs. A normal shutdown from the web UI or the `shutdown` command does **not** schedule a wake-up.

---

## Prerequisites

### BIOS settings
- **RTC wake** enabled. The setting may be called "Resume by RTC Alarm," "Power On by RTC," or "Wake on RTC." Leave any fixed time fields in the BIOS blank or at 0 so the alarm set by Linux is used.
- **ErP / EuP (deep power saving)** disabled. This mode cuts standby power, which prevents RTC wake and Wake-on-LAN.
- *(Recommended)* **Restore on AC Power Loss → Power On**, so the server comes back after an outage that drains the UPS.

### Host timezone
Cron runs in the host's local time, so check it:
```bash
timedatectl
timedatectl set-timezone America/Chicago   # if needed
```

---

## Step 1 — Test RTC wake support

```bash
rtcwake -m no -s 300 && shutdown -h now
```
This sets an alarm 5 minutes out and shuts down cleanly. If the server powers itself back on after about 5 minutes, the hardware supports RTC wake. If it doesn't, recheck the BIOS settings above.

---

## Step 2 — Create the script

```bash
nano /usr/local/bin/sleep-until.sh
```

```bash
#!/bin/bash
# Usage: sleep-until.sh "07:30"
# Sets the RTC alarm for the next occurrence of the given time, then shuts down.
WAKE=$(date +%s -d "tomorrow $1")
# If the wake time hasn't passed yet today, wake today instead
[ $(date +%s -d "today $1") -gt $(date +%s) ] && WAKE=$(date +%s -d "today $1")
/usr/sbin/rtcwake -m no -t "$WAKE"
/sbin/shutdown -h now
```

```bash
chmod +x /usr/local/bin/sleep-until.sh
```

**Why a Unix timestamp?** Passing `-t <epoch seconds>` lets `rtcwake` handle the conversion between UTC and local time, so it doesn't matter how the hardware clock is configured.

---

## Step 3 — Schedule it with cron

```bash
crontab -e
```
On first run, pick **1 (nano)**. Add this line at the bottom:

```
0 0 * * * /usr/local/bin/sleep-until.sh 07:30
```

| Field | Value | Meaning |
|---|---|---|
| minute | `0` | on the hour |
| hour | `0` | midnight |
| day of month | `*` | every day |
| month | `*` | every month |
| day of week | `*` | every weekday (use `1-5` for Mon–Fri only) |

Save with **Ctrl+O** and **Enter**, then exit with **Ctrl+X**. Verify:
```bash
crontab -l
```

---

## Step 4 — Make guests start on boot

In the Proxmox web UI, for each VM or container that should come back automatically (e.g., the Samba LXC):

**Guest → Options → Start at boot → Yes**

Use **Start/Shutdown order** to set a startup delay if one guest depends on another.

---

## Shutting down early (and still waking on schedule)

Run the script manually instead of a plain shutdown:
```bash
/usr/local/bin/sleep-until.sh 07:30
```
Before midnight, the script sees that 7:30 today has passed and sets the alarm for 7:30 tomorrow. You can pass any time for a one-off wake, e.g. `sleep-until.sh 09:00`.

Optional shortcut:
```bash
echo "alias sleepnow='/usr/local/bin/sleep-until.sh 07:30'" >> ~/.bashrc && source ~/.bashrc
```

---

## Alternatives considered

- **BIOS-only power-on schedule:** Simple, but you can only change it in BIOS setup and it can't vary by day.
- **Wake-on-LAN:** Enable WoL in the BIOS and on the NIC, then send a magic packet from another device on the network, such as a desktop or a pfSense router. Better for on-demand wake than for a fixed schedule.

---

## Removing the automation

### 1. Remove the cron job
```bash
crontab -e
```
Delete the `sleep-until.sh` line (**Ctrl+K** in nano), save, and exit. If that's the only job, you can remove the whole crontab instead:
```bash
crontab -r
```
Verify with `crontab -l`. It should show no `sleep-until.sh` line, or print `no crontab for root`.

### 2. Clear any pending wake alarm
```bash
rtcwake -m disable
rtcwake -m show      # should report: alarm: off
```

### 3. Optional cleanup
```bash
rm /usr/local/bin/sleep-until.sh
sed -i '/alias sleepnow=/d' ~/.bashrc && unalias sleepnow 2>/dev/null
```

### 4. BIOS
The RTC wake setting can stay enabled. With nothing setting an alarm, it has no effect. If a fixed power-on time was entered directly in the BIOS, clear it or disable the setting.
