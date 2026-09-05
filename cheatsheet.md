# Jetson Orin Nano — Command Cheatsheet

Quick-reference commands for VNC virtual desktops, camera capture, and
network connections on this machine (`ubuntu-neil`). See also:
`ai-context/plans/headless-vnc-ipad-iphone-setup.md` and
`ai-context/camera-cam0-cam1-debug.md` for full background/rationale.

## VNC virtual desktops

| Display | Purpose | VNC port | Boot state |
|---|---|---|---|
| `:1` | Real GDM/GNOME session (physical monitor) | 5903 (manual only, not a systemd unit) | Always up (primary session) |
| `:2` | iPad-sized virtual desktop | 5901 | **Enabled** — starts at boot |
| `:3` | iPhone-sized virtual desktop | 5902 | **Enabled** — starts at boot |
| `:4` | Generic 1280x720 virtual desktop | 5904 | **Enabled** — starts at boot |

Each virtual desktop is 3 systemd units: `xvfb-<name>[-portrait|-landscape]`,
`desktop-<name>` (openbox+tint2), `x11vnc-<name>`.

```bash
# Check what's running/enabled right now
systemctl list-unit-files | grep -E 'xvfb|desktop-i|desktop-d|x11vnc'
systemctl list-units --all --type=service | grep -E 'xvfb|desktop-i|desktop-d|x11vnc'

# All of :2/:3/:4 now auto-start at boot (see table above). Stop one manually if not needed:
sudo systemctl stop xvfb-desktop4 desktop-desktop4 x11vnc-desktop4

# Manually start VNC on the real monitor session (:1, port 5903) — not persistent
DISPLAY=:1 x11vnc -display :1 -rfbport 5903 -forever -shared -rfbauth /home/neil/.vnc/passwd &

# Run a GUI app on a specific virtual desktop from an SSH/console session
DISPLAY=:2 <command>   # iPad desktop
DISPLAY=:3 <command>   # iPhone desktop
DISPLAY=:4 <command>   # generic desktop
```

**Important:** `nvarguscamerasrc` (CSI camera / Argus) requires a real
GPU/EGL context. It works with `DISPLAY` completely unset (headless) or on
`:1` (real GPU session) — it does **not** work on `:2`/`:3`/`:4` (Xvfb, pure
software framebuffer, 0% GPU). Run camera capture/AI scripts headless
(no `DISPLAY` set), and use `:1` only for playback/viewing afterward.

## Camera commands

Viewer scripts live in `/home/neil/work/CSI-Camera/`. Camera map:
- `sensor-id=0` → imx296 (CSI, cam0)
- `sensor-id=1` → imx477 (CSI, cam1)
- USB "Global Shutter Camera" → always address via its stable `by-id` path,
  **not** `/dev/videoN` (that number shifts around depending on enumeration
  order/what else is plugged in):
  `/dev/v4l/by-id/usb-Global_Shutter_Camera_Global_Shutter_Camera_2602040001-video-index0`

```bash
# Single camera viewer (needs DISPLAY=:1 or unset+headless test)
python3 simple_camera.py 0            # CSI cam0 (imx296)
python3 simple_camera.py 1            # CSI cam1 (imx477)
python3 simple_camera.py usb          # USB camera (uses the by-id path by default)
python3 simple_camera.py usb --yuyv   # uncompressed YUYV @ 2592x1944 - needs SuperSpeed (USB3) cable

# Same, but with a saved guvcview control profile instead of the built-in
# tuning (see ai-context/usb-camera-settings-log.md "--profile" section)
python3 simple_camera.py usb --yuyv --profile ~/work/camera-settings/sunglight.gpfl

# Trio viewer — all 3 cameras side by side ("Trio Cameras" window)
DISPLAY=:1 python3 dual_camera.py

# Headless raw GStreamer test (no OpenCV window, good for confirming a sensor works)
gst-launch-1.0 nvarguscamerasrc sensor-id=0 num-buffers=5 ! \
  'video/x-raw(memory:NVMM),width=1920,height=1080,framerate=30/1' ! \
  nvvidconv ! fakesink

# List USB/V4L2 devices
v4l2-ctl --list-devices

# Check/set USB camera exposure if image looks black (low light, not broken)
DEV=/dev/v4l/by-id/usb-Global_Shutter_Camera_Global_Shutter_Camera_2602040001-video-index0
v4l2-ctl -d $DEV --list-ctrls-menus
v4l2-ctl -d $DEV --set-ctrl=auto_exposure=1,exposure_time_absolute=10000,gain=232,brightness=255
v4l2-ctl -d $DEV --set-ctrl=auto_exposure=3   # back to auto

# GUI control panel (live preview + sliders) - alternative to v4l2-ctl
guvcview -d /dev/video0   # NOTE: short /dev/videoN path only - guvcview
                          # truncates/fails on the long by-id path above.
                          # qv4l2 is installed but segfaults on this
                          # machine regardless of args - don't bother.
```

`simple_camera.py usb --yuyv` (2592x1944) has manual exposure/gain/focus
tuning baked in as of 2026-08-10 - it's no longer factory-default auto
exposure/autofocus. **Controls only apply if set before the capture process
opens the device** - `v4l2-ctl` writes against an already-streaming device
are silently accepted but never reach the sensor; stop the process first.
Full settings baseline/change log/gotchas: `ai-context/usb-camera-settings-log.md`.

## Recording / playback scripts

Live in `/home/neil/work/scripts/`, reachable as "a click" everywhere:
`:1` has a Desktop icon + GNOME app grid entry; `:2`/`:3`/`:4` have entries
in the right-click root menu (openbox).

| Script | What it does |
|---|---|
| `record_camera.py` | Click to start (pick camera via popup), click again to stop. Headless - no preview window. Records fragmented mp4 (`qtmux fragment-duration=1000`) so it stays openable while still recording. |
| `record_camera_preview.py` | Same toggle, plus a live preview window (`tee`'d off the same pipeline). USB: works on any display. CSI/Trio: preview window only on `:1` (needs real GPU/EGL); on `:2`/`:3`/`:4` it silently falls back to headless-only. |
| `view_recordings.sh` | If a recording is active, opens that (still-growing) file directly in `mpv` - a few seconds behind live. Otherwise opens a file picker rooted at `recordings/`. |

USB camera picker offers two modes: **default** is uncompressed YUYV at max
resolution (2592x1944, needs the SuperSpeed/USB3 cable), "other" is MJPG at
1920x1080 (works over USB2 too, lighter on the encoder).

Recordings saved to `/home/neil/work/scripts/recordings/`. Encoding is
`x264enc` (software) - this system's `nvvideo4linux2` GStreamer plugin only
ships the hardware **decoder** (`nvv4l2decoder`), no hardware encoder.

Playback uses `mpv` (not Totem/GNOME - lighter weight, and controllable via
`mpv-mpris`), with different args per display:
- `:1` (real GPU monitor session): plain `mpv` with defaults - uses the
  real NVIDIA-driven GL-backed VO, tries hardware decode (`v4l2m2m-copy`)
  and falls back to software cleanly if it can't (seen on some recordings -
  NAL unit format mismatch - mpv just degrades gracefully, no crash).
- `:2`/`:3`/`:4` (Xvfb, no GPU/EGL): `mpv --vo=x11 --hwdec=no` - plain/minimal,
  the only thing proven to work without a GPU.

Note: `nvgstplayer-1.0` and GStreamer's `nveglglessink` were tried first for
`:1` and both **crashed** (segfaults) in this environment - abandoned in
favor of mpv, which handles the same hardware gracefully.

### Global hotkeys (work regardless of window focus, even if a preview window covers the whole screen)

| Key | Action |
|---|---|
| F5 | Stop the active recording (no-op if nothing's recording - safe to mash) |
| F6 | Rewind mpv playback 10s |
| F7 | Play/pause mpv playback |
| F8 | Fast-forward mpv playback 10s |

F6-F8 control mpv via MPRIS (`playerctl -p mpv ...`, enabled by the
`mpv-mpris` plugin), so they work even if mpv's window isn't focused. Wired
up two ways: an openbox keybind in `~/.config/openbox/rc.xml` (`:2`/`:3`/`:4`)
and a GNOME custom keybinding via
`gsettings` (`:1`, under `org.gnome.settings-daemon.plugins.media-keys`).

## WiFi / network connections

```bash
# Current status
nmcli device status
nmcli -f NAME,TYPE,DEVICE,STATE connection show
ip -4 addr
ip route

# Set a static IP on a connection profile (use its current DHCP values)
sudo nmcli connection modify <profile-name> \
  ipv4.method manual \
  ipv4.addresses <ip>/<prefix> \
  ipv4.gateway <gateway-ip> \
  ipv4.dns "<dns1> <dns2>"

# Apply immediately without reboot
sudo nmcli connection down <profile-name> && sudo nmcli connection up <profile-name>

# Revert a profile back to DHCP
sudo nmcli connection modify <profile-name> ipv4.method auto ipv4.addresses "" ipv4.gateway "" ipv4.dns ""

# Bring a connection up/down manually
nmcli connection up "<profile-name>"
nmcli connection down "<profile-name>"

# Stop a profile from auto-reconnecting (e.g. Bluetooth phone tether)
nmcli connection modify "<profile-name>" connection.autoconnect no
```

### Known profiles on this machine

| Profile | Type | Static IP | Autoconnect |
|---|---|---|---|
| `Wired connection 2` | ethernet (`enx341b2284ce7d`, USB dongle) | Static — `192.168.1.170/24`, gw `192.168.1.254` | yes |
| iPad via USB-C (native gadget) | ethernet gadget (`l4tbr0`, built-in USB-C flash/recovery port) | Jetson `192.168.55.1` (fixed, built into `nv-l4t-usb-device-mode.service`), iPad auto-DHCPs to `192.168.55.100` — no manual iPad config needed | n/a (always-on system service) |
| `varisaa-fiber` | wifi (`wlP1p1s0`) | Static — `192.168.1.235/24`, gw `192.168.1.254` | yes |
| `Jacob-Work-iPhone Network` | bluetooth PAN tether (`bnep0`) | Static — `172.20.10.3/28`, gw `172.20.10.1` | **no** (disabled — was auto-reconnecting) |
| `Jacob-Work-iPhone` | wifi (iPhone hotspot over WiFi, `wlP1p1s0`) | Static — `172.20.10.2/28`, gw `172.20.10.1` | yes — auto fallback if `varisaa-fiber` is out of range |

Note: iPhone Personal Hotspot (WiFi or Bluetooth) always shares the phone's
**cellular** connection — the local link type (BT vs WiFi) only affects
speed/latency to the phone itself, not whether internet is available.
Bluetooth tethering is much lower throughput/higher latency than WiFi —
fine for SSH/terminal, not great for camera/video streaming.
