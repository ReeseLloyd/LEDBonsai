# LEDBonsai Kiosk — Raspberry Pi Setup Guide

A step-by-step guide to reproduce this kiosk display setup on a fresh Raspberry Pi.

## Hardware

- Raspberry Pi 3B (or newer)
- External HDMI display
- MicroSD card (8GB+)
- Power supply

## 1. Flash the OS

Use [Raspberry Pi Imager](https://www.raspberrypi.com/software/) to flash:
- **OS:** Raspberry Pi OS Desktop (64-bit)
- During flash, configure:
  - Hostname: `ledbonsai`
  - Username: `ledbonsai`
  - Enable SSH
  - Set WiFi credentials

## 2. First Boot & SSH

After booting, find the Pi IP address from your router and connect:

```bash
ssh ledbonsai@<ip-address>
```

Auto-login to the desktop is pre-configured by the imager. Confirm with:

```bash
grep autologin /etc/lightdm/lightdm.conf
```

You should see `autologin-user=ledbonsai` and `autologin-session=rpd-labwc`.

## 3. Set Timezone

```bash
sudo timedatectl set-timezone America/Denver
```

Adjust to your local timezone. This ensures cron jobs run at the correct local time.

## 4. Chromium Kiosk Autostart

Create the labwc user autostart file. This replaces the default desktop session
(panel, taskbar) with a direct Chromium kiosk launch.

```bash
mkdir -p ~/.config/labwc
cat > ~/.config/labwc/autostart << 'AUTOSTART'
/usr/bin/kanshi &
sleep 3
chromium --kiosk --ozone-platform=wayland --noerrdialogs --disable-infobars \
  --disable-translate --check-for-update-interval=31536000 \
  --password-store=basic \
  'http://reeselloyd.com/ledbonsai.html?screensaver=1&ssinterval=300&focus=1' &
AUTOSTART
```

**Flag notes:**
- `--kiosk` — fullscreen, no browser UI
- `--ozone-platform=wayland` — forces the Wayland backend. Without it Chromium can fall back to X11 and fail to start when launched outside the desktop session (e.g. by the watchdog in Step 7 or a manual SSH relaunch)
- `--password-store=basic` — suppresses the "Choose password for new keyring" prompt on first boot
- `--check-for-update-interval=31536000` — disables update checks (1 year in seconds)
- `ssinterval=300` — animation speed (lower = faster)

## 5. Hide the Mouse Cursor

The cursor is managed by the Wayland compositor (labwc), so X11 tools like
`unclutter` do not work. Create a transparent XCursor theme instead:

```bash
python3 << 'PYTHON'
import struct, pathlib

def make_xcursor(nom_size=24):
    magic = b"Xcur"
    file_hdr = struct.pack("<III", 16, 0x10000, 1)
    toc = struct.pack("<III", 0xfffd0002, nom_size, 28)
    img = struct.pack("<IIIIIIIII", 36, 0xfffd0002, nom_size, 1, 1, 1, 0, 0, 50)
    img += struct.pack("<I", 0x00000000)
    return magic + file_hdr + toc + img

base = pathlib.Path.home() / ".local/share/icons/blank-cursor/cursors"
base.mkdir(parents=True, exist_ok=True)
(base / "left_ptr").write_bytes(make_xcursor())
names = [
    "default","arrow","X_cursor","right_ptr","watch","crosshair","text",
    "pointer","move","cell","help","wait","progress","not-allowed",
    "context-menu","no-drop","copy","alias","grab","grabbing","all-scroll",
    "col-resize","row-resize","n-resize","e-resize","s-resize","w-resize",
    "ne-resize","nw-resize","se-resize","sw-resize","ew-resize","ns-resize",
    "nesw-resize","nwse-resize","zoom-in","zoom-out"
]
for name in names:
    link = base / name
    if not link.exists():
        link.symlink_to("left_ptr")
(base.parent / "index.theme").write_text("[Icon Theme]\nName=blank-cursor\n")
print("Done")
PYTHON
```

Activate the theme in the labwc environment:

```bash
cat >> ~/.config/labwc/environment << 'ENV'
XCURSOR_THEME=blank-cursor
XCURSOR_SIZE=24
ENV
```

## 6. Display Scheduling

The display turns on at 5AM and off at 10PM using `wlopm` (Wayland output power manager).
Note: `vcgencmd display_power` does NOT work with the KMS driver on newer Pi OS, and
`wlr-randr --off/--on` destroys the output mode config causing an unrecoverable black screen —
use `wlopm` instead, which sends a DPMS signal without touching the mode.

### Install wlopm

```bash
sudo apt-get install -y wlopm
```

### Wrapper scripts

```bash
cat > ~/display-off.sh << 'SCRIPT'
#!/bin/bash
export WAYLAND_DISPLAY=wayland-0
export XDG_RUNTIME_DIR=/run/user/$(id -u ledbonsai)
wlopm --off HDMI-A-1
SCRIPT
chmod +x ~/display-off.sh

cat > ~/display-on.sh << 'SCRIPT'
#!/bin/bash
export WAYLAND_DISPLAY=wayland-0
export XDG_RUNTIME_DIR=/run/user/$(id -u ledbonsai)
wlopm --on HDMI-A-1
SCRIPT
chmod +x ~/display-on.sh
```

> **Note:** Confirm your output name first:
> `WAYLAND_DISPLAY=wayland-0 XDG_RUNTIME_DIR=/run/user/$(id -u) wlr-randr`
> It may differ from `HDMI-A-1` on different hardware.

## 7. Chromium Watchdog

Restarts Chromium automatically if it crashes:

```bash
cat > ~/chromium-watchdog.sh << 'SCRIPT'
#!/bin/bash
if ! pgrep -x chromium > /dev/null; then
    export WAYLAND_DISPLAY=wayland-0
    export XDG_RUNTIME_DIR=/run/user/$(id -u)
    export XDG_SESSION_TYPE=wayland
    chromium --kiosk --ozone-platform=wayland --noerrdialogs --disable-infobars \
      --disable-translate --check-for-update-interval=31536000 \
      --password-store=basic \
      'http://reeselloyd.com/ledbonsai.html?screensaver=1&ssinterval=300&focus=1' &
fi
SCRIPT
chmod +x ~/chromium-watchdog.sh
```

> **Why `XDG_SESSION_TYPE=wayland` and `--ozone-platform=wayland`?** cron runs the
> watchdog *outside* the labwc desktop session, so the session environment is absent.
> Without these, Chromium defaults to the X11 backend, fails with
> `Missing X server or $DISPLAY`, and exits — so the watchdog (or a manual relaunch
> over SSH) can't actually bring the kiosk back, and the screen stays black until a
> reboot. Forcing Wayland makes recovery work.

## 8. WiFi Watchdog

Checks connectivity every 30 minutes and attempts to reconnect if offline.
Logs failures only to `~/wifi-watchdog.log` (auto-trimmed to 500 lines).

Install `wtype` (used for sending keystrokes to the Wayland session):

```bash
sudo apt-get install -y wtype
```

Create the watchdog script:

```bash
cat > ~/wifi-watchdog.sh << 'SCRIPT'
#!/bin/bash
LOGFILE="/home/ledbonsai/wifi-watchdog.log"
PING_HOST="1.1.1.1"
WIFI_CONN="DeadLeg-5G"

log() {
    echo "$(date "+%Y-%m-%d %H:%M:%S") $1" >> "$LOGFILE"
}

# Keep log under 500 lines
if [ "$(wc -l < "$LOGFILE" 2>/dev/null)" -gt 500 ]; then
    tail -400 "$LOGFILE" > "${LOGFILE}.tmp" && mv "${LOGFILE}.tmp" "$LOGFILE"
fi

if ping -c 3 -W 5 "$PING_HOST" > /dev/null 2>&1; then
    exit 0
fi

log "No connectivity detected. Cycling WiFi radio..."
nmcli radio wifi off
sleep 5
nmcli radio wifi on
sleep 15

if ping -c 3 -W 5 "$PING_HOST" > /dev/null 2>&1; then
    log "Reconnected after radio cycle."
    exit 0
fi

log "Still offline. Forcing connection up: $WIFI_CONN"
nmcli connection up "$WIFI_CONN"
sleep 10

if ping -c 3 -W 5 "$PING_HOST" > /dev/null 2>&1; then
    log "Reconnected after forcing connection up."
else
    log "Still offline after all attempts. Manual intervention may be needed."
fi
SCRIPT
chmod +x ~/wifi-watchdog.sh
```

> **Note:** Update `WIFI_CONN` to match your network name. Check with `nmcli connection show --active`.

To send keystrokes to Chromium from an SSH session (e.g. to trigger in-page actions):

```bash
ssh ledbonsai@<ip> 'WAYLAND_DISPLAY=wayland-0 XDG_RUNTIME_DIR=/run/user/$(id -u) wtype n'
```

## 9. Cron Jobs

```bash
(crontab -l 2>/dev/null; cat << 'CRON'
0 5 * * * /home/ledbonsai/display-on.sh
0 22 * * * /home/ledbonsai/display-off.sh
*/5 * * * * /home/ledbonsai/chromium-watchdog.sh
*/30 * * * * /home/ledbonsai/wifi-watchdog.sh
CRON
) | crontab -
```

Weekly reboot at 4AM Sunday (before the 5AM display-on, so it goes unnoticed):

```bash
(sudo crontab -l 2>/dev/null; echo "0 4 * * 0 /sbin/reboot") | sudo crontab -
```

## 10. Reboot & Verify

```bash
sudo reboot
```

After rebooting, verify from another machine:

```bash
# Chromium is running
ssh ledbonsai@<ip> 'pgrep -a chromium'

# Cron jobs are in place
ssh ledbonsai@<ip> 'crontab -l && sudo crontab -l'

# Timezone is correct
ssh ledbonsai@<ip> 'timedatectl'
```

## Key Files Reference

| File | Purpose |
|------|---------|
| `~/.config/labwc/autostart` | Kiosk startup — Chromium launch |
| `~/.config/labwc/environment` | Cursor theme and env vars |
| `~/.local/share/icons/blank-cursor/` | Transparent cursor theme |
| `~/display-on.sh` | Turn HDMI on |
| `~/display-off.sh` | Turn HDMI off |
| `~/chromium-watchdog.sh` | Restart Chromium if crashed |
| `~/wifi-watchdog.sh` | Reconnect WiFi if offline |
| `~/wifi-watchdog.log` | WiFi watchdog failure log |

## Cron Schedule Summary

| Time | Job |
|------|-----|
| 5:00 AM daily | Display on |
| 10:00 PM daily | Display off |
| Every 5 minutes | Chromium watchdog |
| Every 30 minutes | WiFi watchdog |
| 4:00 AM Sunday | System reboot |

## Troubleshooting

**Black screen after boot (affects Pi OS Lite only):**
Add `hdmi_force_hotplug=1` to `/boot/firmware/config.txt`.
The Desktop version handles HDMI initialisation automatically.

**Keyring prompt on first boot:**
Ensure `--password-store=basic` is in the Chromium flags in both
`~/.config/labwc/autostart` and `~/chromium-watchdog.sh`.

**Watchdog won't restart Chromium (black screen after a crash or `pkill`):**
The watchdog runs from cron, outside the desktop session, so it must force the
Wayland backend. Confirm `export XDG_SESSION_TYPE=wayland` and
`--ozone-platform=wayland` are present in `~/chromium-watchdog.sh` (Step 7).
Without them Chromium falls back to X11 and exits with
`Missing X server or $DISPLAY`. A reboot always recovers (the labwc autostart has
the full session environment), but the watchdog should be able to recover on its own.

**Cursor still visible:**
`unclutter` and `unclutter-xfixes` are X11-only and do not work on Wayland.
Use the transparent XCursor theme method in Step 5.

**Display scheduling not working:**
Confirm the Wayland socket name: `ls /run/user/$(id -u)/wayland-*`
It should be `wayland-0`. Update the scripts if different.

**wlr-randr output name differs from HDMI-A-1:**
Run `WAYLAND_DISPLAY=wayland-0 XDG_RUNTIME_DIR=/run/user/$(id -u) wlr-randr`
to list available outputs and update display-on.sh and display-off.sh accordingly.
Also check output name via `WAYLAND_DISPLAY=wayland-0 XDG_RUNTIME_DIR=/run/user/$(id -u) wlopm`.

**Screen black in the morning (display stuck at 0x0, Chromium still running):**
Do NOT use `wlr-randr --off/--on` for scheduling. It destroys the output mode config and
`wlr-randr --on` cannot recover it — the output stays at `0x0 px (current)` and appears black.
Use `wlopm --off/--on` instead (Step 6). `wlopm` sends a DPMS signal that blanks and wakes the
screen without touching the mode, so it recovers cleanly every time.

To manually restore the display over SSH when it's stuck:
```bash
WAYLAND_DISPLAY=wayland-0 XDG_RUNTIME_DIR=/run/user/$(id -u ledbonsai) wlopm --on HDMI-A-1
```
