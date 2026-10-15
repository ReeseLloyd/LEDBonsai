# LEDBonsai Lite — Pi Zero 2 W Offline Kiosk Setup

How the Pi Zero 2 W desk display was actually built: LEDBonsai in **Lite mode** (`?lite=1`) on a
Waveshare 4inch DPI LCD (C) 720x720 touch panel, running **fully offline** once deployed.
Companion to `LEDBonsai-PiSETUP.md` (Pi 3B / HDMI / labwc / web-hosted), which remains valid for that setup.

| | Pi 3B guide | This guide |
|---|---|---|
| OS | Desktop 64-bit | **Lite, 32-bit** (Bookworm) |
| Display | HDMI | **DPI on the GPIO header** (Waveshare 4inch DPI LCD (C)) |
| Compositor | labwc (desktop session) | **cage** (single-app kiosk compositor) |
| App source | `http://reeselloyd.com/...` | **Local file**, no network needed |
| Rendering | DOM | **Canvas, software-rendered** (`--disable-gpu`) |
| Reliability | cron watchdogs, weekly reboot | restart loop, overlay (read-only) filesystem, uptime-based daily reboot |

Everything below was done and verified on the real hardware unless marked **(not yet verified)**.

---

## Hardware

- Raspberry Pi Zero 2 W (512 MB RAM, about 425 MB usable), 40-pin header fitted
- **Waveshare 4inch DPI LCD (C)**, 720x720 IPS with I2C capacitive touch (sold under the XYGStudy name).
  It plugs onto the 40-pin header. This is **DPI**, not HDMI or DSI: it uses most GPIO pins, so an RTC
  module cannot share the header (no real-time clock; see Step 8).
- MicroSD card, 5 V supply

## 1. Flash the OS

[Raspberry Pi Imager](https://www.raspberrypi.com/software/): **Raspberry Pi OS Lite, 32-bit** (Bookworm).
Hostname `deskbonsai`, user `deskbonsai`, SSH enabled, home WiFi configured.
(32-bit uses less RAM than 64-bit, which matters with Chromium on 512 MB.)

```bash
ssh deskbonsai@deskbonsai.local
sudo apt update && sudo apt full-upgrade -y
sudo timedatectl set-timezone America/Denver   # adjust
```

## 2. Panel: Waveshare overlays

Per Waveshare's wiki for the 4inch DPI LCD (C) (Bookworm/KMS):

1. Download `4DPIC-DTBO.zip` from `https://files.waveshare.com/upload/8/8a/4DPIC-DTBO.zip` (5,302 bytes).
   Unzip and copy these into `/boot/firmware/overlays/` (`/boot/overlays` is a symlink to it):
   `vc4-kms-DPI-4inch.dtbo`, `waveshare-4dpic-3b-4b.dtbo`, `waveshare-4dpic-3b.dtbo`,
   `waveshare-4dpic-4b.dtbo`, `waveshare-4dpi.dtbo`.
2. Append to `/boot/firmware/config.txt` (the existing `dtoverlay=vc4-kms-v3d` line stays):

   ```
   dtoverlay=vc4-kms-DPI-4inch
   dtoverlay=waveshare-4dpic-3b-4b
   dtoverlay=waveshare-4dpic-3b
   dtoverlay=waveshare-4dpic-4b
   dtoverlay=waveshare-4dpi
   ```

   Waveshare's page also lists `waveshare-touch-4dpi`. On this Pi touch **does** work with it
   enabled (the Goodix controller registers as a pointer), and tap-to-advance relies on it.
3. Rotate 180 degrees (the panel was mounted upside down): prepend to `/boot/firmware/cmdline.txt`
   (keep it a single line):

   ```
   video=DPI-1:720x720M@60,rotate=180
   ```

Reboot. `cat /sys/class/drm/card0-DPI-1/status` should say `connected`. The Waveshare page lists the
Pi 3B and 4B but not the Zero 2 W; it works fine on this board.

## 3. Install the kiosk stack

```bash
sudo apt install -y --no-install-recommends cage chromium zram-tools
```

(`wlopm` was installed too but is not used, since there is no display schedule; see Step 9.)
Chromium's dependencies make this slow on the Zero (tens of minutes). Let it finish.

## 4. Copy the app

```bash
scp ledbonsai.html deskbonsai@deskbonsai.local:ledbonsai.html
```

The kiosk loads `file:///home/deskbonsai/ledbonsai.html?lite=1&glow=0`; nothing is fetched from the network.
`ssinterval` defaults to 300 s (5 minutes) in lite mode.

`glow=0` turns off the canvas bloom. On this panel the colors looked oversaturated with the bloom on, and
looked right with it off. Remove `&glow=0` from the profile to bring the glow back.

## 5. Autologin and kiosk launch

```bash
sudo raspi-config nonint do_boot_behaviour B2     # console autologin
```

`~/.bash_profile` launches cage + Chromium from the tty1 autologin and restarts it if it exits:

```bash
# LEDBonsai kiosk: on the tty1 autologin, run cage + Chromium and restart if it exits
if [ "$(tty)" = "/dev/tty1" ]; then
    export XCURSOR_THEME=blank-cursor XCURSOR_SIZE=24
    while true; do
        cage -- chromium --no-memcheck --disable-gpu --enable-features=NetworkServiceInProcess2 --kiosk \
            --ozone-platform=wayland \
            --noerrdialogs --disable-infobars --disable-session-crashed-bubble \
            --no-first-run --disable-translate \
            --password-store=basic \
            --disable-background-networking --disable-component-update --disable-sync \
            --check-for-update-interval=31536000 --disk-cache-dir=/dev/null --disk-cache-size=1 \
            --js-flags=--max-old-space-size=128 \
            "file:///home/deskbonsai/ledbonsai.html?lite=1&glow=0" 2>/dev/null
        sleep 3
    done
fi
[ -f ~/.bashrc ] && . ~/.bashrc
```

**Why each non-obvious flag is there (all found by hitting the problem on this Pi):**

| Flag | Reason |
|---|---|
| `--no-memcheck` | Raspberry Pi's Chromium wrapper shows a blocking "less than 1GB of RAM" dialog otherwise |
| `--disable-gpu` | The Zero's GPU only supports OpenGL ES 2.0; with GPU rendering enabled the screen stayed black. Software rendering is fine for a canvas drawn once |
| `--enable-features=NetworkServiceInProcess2` | **Required.** Without it Chromium's separate network process crashes at launch and the initial page load silently never commits (blank screen forever). Running the network service in the browser process fixes it and saves RAM |
| `--disable-background-networking`, `--disable-component-update`, `--disable-sync`, `--check-for-update-interval` | Stop Chromium reaching for the internet |
| `--disk-cache-dir=/dev/null --disk-cache-size=1` | Nothing to cache; keeps the RAM-backed overlay clean |
| `2>/dev/null` | Chromium logs constantly while offline; do not let that fill RAM. Remove it temporarily to debug |

## 6. Hide the mouse cursor

The touch controller shows up as a pointer, so a cursor appears. Three layers hide it:

1. The app sets `cursor: none` in lite mode (v1.36).
2. A transparent XCursor theme at `~/.local/share/icons/blank-cursor/` (generated by the Python snippet in
   Step 5 of `LEDBonsai-PiSETUP.md`; same method).
3. The **default** theme is pointed at it, because Chromium loads the default theme, not `XCURSOR_THEME`:

   ```bash
   mkdir -p ~/.icons/default ~/.local/share/icons/default
   for d in ~/.icons/default ~/.local/share/icons/default; do
       printf "[Icon Theme]\nName=Default\nInherits=blank-cursor\n" > "$d/index.theme"
   done
   ln -sfn ~/.local/share/icons/blank-cursor ~/.icons/blank-cursor
   ```

   Layers 1 and 2 alone left the cursor visible; layer 3 is what finished it.

## 7. Trim services, swap and uptime

```bash
sudo systemctl disable --now apt-daily.timer apt-daily-upgrade.timer
sudo systemctl disable --now bluetooth hciuart ModemManager triggerhappy triggerhappy.socket
sudo systemctl disable --now dphys-swapfile      # swap lives in zram instead (zram-tools default: 50% RAM, lz4)
```

Daily reboot, scheduled by **uptime** (24 hours after boot, not a wall-clock time), so it works with no RTC and
no NTP. This clears any slow browser memory growth. It started as 7 days and was shortened to 24 hours, since the
Pi does nothing else and a reboot only costs about 1.5 minutes of dark screen:

```bash
sudo tee /etc/systemd/system/ledbonsai-reboot.service >/dev/null <<EOF
[Unit]
Description=LEDBonsai scheduled reboot
[Service]
Type=oneshot
ExecStart=/usr/bin/systemctl reboot
EOF
sudo tee /etc/systemd/system/ledbonsai-reboot.timer >/dev/null <<EOF
[Unit]
Description=Reboot 24 hours after boot
[Timer]
OnBootSec=24h
[Install]
WantedBy=timers.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable ledbonsai-reboot.timer
```

## 8. Time (offline, no RTC)

The DPI panel occupies the header, so a DS3231 RTC isn't practical. Offline the clock is wrong after every
power loss. LEDBonsai only uses relative timers, so it doesn't matter to the app, and the display is simply
left on (no schedule). The only visible effect is log timestamps.

## 9. Display schedule: not used

The panel is always on. A schedule would need a correct clock (Step 8) and a way to switch the DPI
backlight (a PWM line on GPIO 18 per Waveshare); neither is set up. **(not implemented)**

## 10. WiFi for home and work

Home WiFi comes from Imager. Add the work network (replace the placeholders) so the Pi joins whichever is in range:

```bash
sudo nmcli connection add type wifi con-name work ssid "WORK_SSID" \
    wifi-sec.key-mgmt wpa-psk wifi-sec.psk "WORK_PASSWORD" \
    connection.autoconnect-priority 5
```

(Higher priority wins when both are visible.) The Pi never *needs* the network to run; this is for maintenance access only.

## 11. Read-only filesystem (overlay)

An unattended Pi that may be unplugged will eventually corrupt a normal SD card. The overlay makes the
root filesystem read-only (writes go to RAM and vanish on reboot):

```bash
sudo raspi-config nonint enable_overlayfs      # installs overlayroot, adds overlayroot=tmpfs to cmdline
sudo reboot
```

Check it: `grep -c overlayroot /proc/cmdline` prints 1, and `findmnt /` shows `overlay` (**verified**: the real SD card
appears read-only at `/media/root-ro`, and writes land in a RAM layer of about 210 MB). The first run installs the
`overlayroot` package and rebuilds the initramfs, which takes several minutes on the Zero.
Needs more than 256 MB RAM (raspi-config refuses otherwise). `/boot/firmware` is left writable on purpose;
nothing writes to it, and read-only `/boot` would only complicate maintenance.

**Maintenance mode** (helper scripts in `~`, created before enabling the overlay):

| Script | What it does |
|---|---|
| `~/maint-start.sh` | Disables the overlay and reboots. Root is writable; change anything, install, edit |
| `~/maint-finish.sh` | Re-enables the overlay and reboots. Changes are now locked in |

Anything you change **while the overlay is on is lost on reboot**: always run `maint-start.sh` first.

## 12. Updating the app

```bash
ssh deskbonsai@deskbonsai.local '~/maint-start.sh'        # reboots into writable mode; wait ~1.5 min
scp ledbonsai.html deskbonsai@deskbonsai.local:ledbonsai.html
ssh deskbonsai@deskbonsai.local '~/maint-finish.sh'       # locks it again and reboots
```

(Edits to `.bash_profile`, `config.txt` and friends follow the same pattern.)

## Behavior

- A new tree every 5 minutes (counted from when a tree finishes drawing).
- **Tap or click anywhere**: the screen blanks, then the next tree appears when ready (about 5 s on the Zero).
- Cold boot to first tree: about 1.5 to 2 minutes.
- Daily reboot by uptime (24 h after boot), plus an automatic restart loop if Chromium or cage exits.

## Offline rehearsal checklist

1. Overlay on (Step 11), WiFi **off** (`sudo nmcli radio wifi off`) or out of range, reboot.
2. Confirm a tree appears and keeps cycling. Let it run a day or two.
3. Pull the power without shutting down. Boot again; confirm it recovers by itself.
4. Re-enable WiFi and check `free -m`, `uptime`. **(not yet verified)**

Status: the overlay is on and the kiosk comes back by itself after a clean reboot (**verified**). A full run at the office
with no WiFi at all (cold start, then cycling trees as normal) also **worked as expected**. The deliberate hard
power pull (step 3) has not been run as a separate test.

## Troubleshooting

**Black screen at boot, cursor present.** The Chromium page never loaded. Check `NetworkServiceInProcess2`
is in the profile (Step 5). If you need to see what the browser is doing, temporarily add
`--remote-debugging-port=9222` and query `http://127.0.0.1:9222/json` over SSH (the page's `title` should be
"LEDBonsai v1.xx"; an empty title with a `file://` URL means it never committed). Remove the port afterwards.

**White screen.** Chromium is drawing but the page did not load; same cause as above.

**"It is not recommended to run Chromium on devices with less than 1GB of RAM" dialog.** Add `--no-memcheck`.

**Panel dark / `DPI-1` disconnected.** Check the five overlay lines and that the `.dtbo` files are in
`/boot/firmware/overlays/`; `dmesg | grep -i -E "dpi|panel"`.

**Upside down.** Change `rotate=180` to `0` (or remove it) in `cmdline.txt`.

**Cursor visible.** Confirm Step 6, especially the default-theme `index.theme`.

**Changes disappear after a reboot.** The overlay is on. Use `~/maint-start.sh` first (Step 11).

**Trees too small or large.** Tune the `LITE_*` constants near the top of the script in `ledbonsai.html`.

## Key files

| File | Purpose |
|------|---------|
| `~/ledbonsai.html` | The app (local copy) |
| `~/.bash_profile` | Kiosk launch loop |
| `~/maint-start.sh`, `~/maint-finish.sh` | Toggle the overlay for maintenance |
| `/boot/firmware/config.txt` | Panel overlays (backup: `config.txt.bak-pre-panel`) |
| `/boot/firmware/cmdline.txt` | Rotation, `overlayroot=tmpfs` (backup: `cmdline.txt.bak-pre-panel`) |
| `/etc/systemd/system/ledbonsai-reboot.{service,timer}` | Daily reboot by uptime |
| `~/.icons/default/`, `~/.local/share/icons/` | Transparent cursor theme |

## Ideas for later tuning

Noted, deliberately not changed yet. Revisit after a week or so of running at the office:

1. **Gentler glow.** The canvas bloom made colors look oversaturated on this panel, so the kiosk runs with
   `glow=0`. A reduced bloom (lower `LITE_BLOOM_PASSES` and layer alphas in `ledbonsai.html`) might keep a little
   softness without the vividness. Optional saturation/gamma URL controls for lite mode are another route.
2. **Prefetch the next tree.** A tap currently blanks the screen for about 5 s while the Zero generates and scores
   12 candidates. Computing the next tree during idle time would make a tap nearly instant.
3. **Check long-run memory.** After a few days, compare `free -m` and `uptime` on the Pi to see how Chromium
   behaves between the daily reboots, then adjust the reboot timer if it is too frequent or not frequent enough.
4. **Hard power-pull test.** Pull the power without a shutdown (overlay on) and confirm it comes back by itself
   in about 2 minutes. The office run covered a cold start with no WiFi, but not this specific case.
