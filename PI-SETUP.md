# Raspberry Pi kiosk setup — run for years without babysitting

The board's web page now protects itself (pixel-orbit anti-burn-in, nightly
self-reload at 3 AM, honest offline states). The other half of longevity
lives on the Pi and the monitor. Do these once.

The two failure modes this prevents:

- **Frozen browser** (the Aug 2026 incident: tab froze for 5 days) — fixed by
  a supervised kiosk service that restarts on crash, plus a nightly restart.
- **Panel warp / image persistence in the corners** — caused by heat from the
  backlight running 24/7. A black web page does NOT turn the backlight off;
  only cutting the video signal (or the monitor's own power) does.

---

## 1. Chromium as a supervised kiosk service

Create `~/.config/systemd/user/subway-kiosk.service`:

```ini
[Unit]
Description=Subway board kiosk
After=graphical-session.target

[Service]
ExecStart=/usr/bin/chromium-browser \
  --kiosk --noerrdialogs --disable-infobars \
  --disable-session-crashed-bubble \
  --hide-crash-restore-bubble \
  --check-for-update-interval=31536000 \
  --disk-cache-size=52428800 \
  https://jaredbsampson.github.io/subway-board/
Restart=always
RestartSec=5

[Install]
WantedBy=graphical-session.target
```

Enable it (and remove any old autostart entry for chromium):

```bash
systemctl --user daemon-reload
systemctl --user enable --now subway-kiosk.service
```

Why it matters: `Restart=always` resurrects a crashed browser in 5 seconds,
and `--disable-session-crashed-bubble` / `--hide-crash-restore-bubble` stop
Chromium from parking on a "Restore pages?" dialog after a crash — the
classic way a kiosk stays stuck for days. The small disk cache limits SD-card
wear.

## 2. Nightly browser restart + true display off (crontab)

`crontab -e`, then add — **Wayland (Pi OS Bookworm default)**:

```cron
# Cut the video signal during the board's 2–5 AM black window (backlight off)
0 2 * * *  WAYLAND_DISPLAY=wayland-1 XDG_RUNTIME_DIR=/run/user/1000 wlr-randr --output HDMI-A-1 --off
55 4 * * * WAYLAND_DISPLAY=wayland-1 XDG_RUNTIME_DIR=/run/user/1000 wlr-randr --output HDMI-A-1 --on
# Fresh browser every night while the screen is dark
50 4 * * * XDG_RUNTIME_DIR=/run/user/1000 systemctl --user restart subway-kiosk.service
```

**Older Pi OS / X11** variant:

```cron
0 2 * * *  DISPLAY=:0 xset dpms force off
55 4 * * * DISPLAY=:0 xset dpms force on
50 4 * * * XDG_RUNTIME_DIR=/run/user/1000 systemctl --user restart subway-kiosk.service
```

Check the output name first (`wlr-randr` alone lists outputs; yours may be
`HDMI-A-1` or `HDMI-A-2`). Three hours of backlight-off every night is the
single biggest thing you can do for the panel.

## 3. Monitor settings (the actual warp fix)

- **Hardware brightness ≤ 60%** via the monitor's own buttons. The board's
  phone-controlled "brightness" is a software overlay — it cannot reduce
  backlight heat. Most panel aging is heat × time; 60% backlight roughly
  doubles panel life vs 100%.
- Optional, nicer: most Dell monitors speak DDC/CI, so the Pi can set real
  backlight over the HDMI cable: `sudo apt install ddcutil`, verify with
  `ddcutil detect`, then `ddcutil setvcp 10 50` sets 50% brightness. If you
  tell me the model I can wire the board's brightness slider to this.
- Leave a few inches of ventilation above the monitor; corner warp is heat
  pooling at the bezel.

## 4. Screen orientation / resolution (fixes the sideways-strip)

Wayland: `wlr-randr --output HDMI-A-1 --transform 90 --mode 3840x2160`
(use `--transform normal` for landscape). Make it persist via the desktop's
Screen Configuration tool, or add the wlr-randr line to the kiosk service as
`ExecStartPre=`.

## 5. SD-card longevity

```bash
# Keep logs in RAM instead of grinding the SD card
sudo sed -i 's/#Storage=auto/Storage=volatile/' /etc/systemd/journald.conf
sudo systemctl restart systemd-journald
```

A quality A2-class card or, better, a small USB SSD as the root drive makes
"years" realistic; SD cards are the most common multi-year kiosk casualty.

## 6. One-line health check

```bash
systemctl --user status subway-kiosk.service && vcgencmd measure_temp
```

Pi temperature should stay under ~70 °C; if it runs hotter, add a heatsink or
reposition it out of the monitor's exhaust heat.
