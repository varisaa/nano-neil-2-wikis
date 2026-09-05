# USB Camera (Global Shutter Camera) — Settings Log

Device: Arducam Global Shutter USB Camera (`32e4:5b10`)
Stable path: `/dev/v4l/by-id/usb-Global_Shutter_Camera_Global_Shutter_Camera_2602040001-video-index0`
(`/dev/videoN` shifts depending on enumeration order — always use the `by-id` path.)

## Cheatsheet

```bash
DEV=/dev/v4l/by-id/usb-Global_Shutter_Camera_Global_Shutter_Camera_2602040001-video-index0

# See all current values + defaults + ranges
v4l2-ctl -d $DEV --list-ctrls-menus

# Check one control
v4l2-ctl -d $DEV --get-ctrl=power_line_frequency

# Set one/more controls (comma-separated, no spaces)
v4l2-ctl -d $DEV --set-ctrl=power_line_frequency=2

# Reset everything back to factory defaults in one shot
v4l2-ctl -d $DEV --set-ctrl=brightness=128,contrast=65,saturation=80,hue=0,white_balance_automatic=1,gamma=128,gain=128,power_line_frequency=1,sharpness=128,backlight_compensation=56,auto_exposure=3,exposure_dynamic_framerate=0,pan_absolute=0,tilt_absolute=0,focus_automatic_continuous=1,zoom_absolute=100

# List raw supported resolutions/formats/framerates
v4l2-ctl -d $DEV --list-formats-ext
```

### Where a change actually lands

| Change type | Where it lands | Survives unplug/reboot? |
|---|---|---|
| `v4l2-ctl --set-ctrl=...` run by hand in a shell | **In-memory only**, on the UVC driver/camera firmware. Nothing is written to disk. | **No** — resets to the factory `default=` value shown by `--list-ctrls-menus` |
| A control's value baked into a Python script (e.g. the `power_line_frequency=2` call in `simple_camera.py`/`dual_camera.py`) | The **script file itself** (`/home/neil/work/CSI-Camera/*.py`) — the setting isn't persisted on the camera, but the script re-applies it via `subprocess.run(["v4l2-ctl", ...])` at the start of every run | **Effectively yes**, because the code reapplies it every time the script runs — not because the camera remembers it |
| Anything else (udev rule, systemd service, `/etc/modprobe.d`, etc.) | **Not used for this camera** — there is no persistent system-level config for these UVC controls in this setup | N/A |

So: manual `v4l2-ctl` commands are for live experimentation only and vanish on
reboot/unplug. If a change should "stick," it needs to be added to the
Python scripts (as was done for `power_line_frequency`), since that's the
only thing on this machine that actually re-asserts camera settings automatically.

## Baseline / factory defaults

These are the driver-reported `default=` values (`v4l2-ctl --list-ctrls-menus`), i.e.
what the camera comes up as before anything in this doc was changed.

| Control | Default | Range |
|---|---|---|
| brightness | 128 | 0-255 |
| contrast | 65 | 0-128 |
| saturation | 80 | 0-128 |
| hue | 0 | -128 to 128 |
| white_balance_automatic | 1 (on) | bool |
| gamma | 128 | 0-255 |
| gain | 128 | 0-232 |
| power_line_frequency | 1 (50 Hz) | 0=Disabled, 1=50Hz, 2=60Hz |
| white_balance_temperature | 4650 | 2800-6500 (inactive while auto WB is on) |
| sharpness | 128 | 0-255 |
| backlight_compensation | 56 | 16-160 |
| auto_exposure | 3 (Aperture Priority / auto) | 0-3 |
| exposure_time_absolute | 156 | 1-10000 (inactive while auto_exposure=3) |
| exposure_dynamic_framerate | 0 (off) | bool |
| pan_absolute | 0 | -648000 to 648000 |
| tilt_absolute | 0 | -648000 to 648000 |
| focus_absolute | 0 | 0-1023 (inactive while autofocus is on) |
| focus_automatic_continuous | 1 (on) | bool |
| zoom_absolute | 100 | 100-200 |

## Changes made this session

| Date | Control | From → To | Reason | Status |
|---|---|---|---|---|
| 2026-07-21 | `auto_exposure` | 3 → 1 (Manual) | Force manual exposure to test if the camera was working at all under low light (first bring-up test) | **Reverted** — back to default (3) |
| 2026-07-21 | `exposure_time_absolute` | 156 → 10000 | Same low-light bring-up test | **Reverted** — no longer forced (control is inactive again since auto_exposure=3) |
| 2026-07-21 | `gain` | 128 → 232 (max) | Same low-light bring-up test | **Reverted** — back to default (128) |
| 2026-07-21 | `brightness` | 128 → 255 (max) | Same low-light bring-up test | **Reverted** — back to default (128) |
| 2026-07-21 | `power_line_frequency` | 1 (50Hz) → 2 (60Hz) | Anti-flicker: this Jetson is on US 60Hz mains/lighting, camera defaulted to 50Hz | **Kept** — this is a deliberate fix, baked into `simple_camera.py`/`dual_camera.py` so it's reapplied automatically every run (see below) |
| 2026-08-10 | `auto_exposure` | 3 (Aperture Priority) → 1 (Manual) | Outdoor backlit test scene (shaded deck under trees, bright sky beyond): auto-exposure pinned itself at `exposure_time_absolute`'s ceiling (10000) trying to brighten the shaded foreground, blowing the sky to pure white and causing purple fringing at the clipped/overexposed edges (classic sensor blooming/CA). No scene-adaptive metering on this sensor, so auto mode had no better option available. | **Kept** — baked into `simple_camera.py`'s `gstreamer_pipeline_usb_yuyv()` |
| 2026-08-10 | `exposure_time_absolute` | inactive → 800 | Manually picked short-ish exposure once in Manual mode. See "Surprising finding" below - the sky/fringing didn't actually improve much even down to 500, this scene's dynamic range just exceeds the sensor either way. Chosen mainly to reduce sensor blooming a bit and to reduce reliance on the auto algorithm's ceiling-pinning behavior on future backlit scenes. | **Kept** |
| 2026-08-10 | `gain` | 128 → 160 | Compensate midtone brightness for the shorter fixed exposure above | **Kept** |
| 2026-08-10 | `focus_automatic_continuous` | 1 (on) → 0 (off) | Continuous AF wasn't converging as sharp as a manually-picked point on a multi-depth-plane scene (near foliage + mid-distance paper sign) - visibly hazy/soft text edges at whatever point it settled on (~750-950 range observed) | **Kept** |
| 2026-08-10 | `focus_absolute` | inactive → 600 | Found by bracketing manual values (500/600/650/700/750/798/850/900/950) against a fixed real-world target (handwritten text on paper, photographed via screenshot+crop for each value) - 600 was visibly the sharpest; 500 was worse, 650+ visually indistinguishable from the hazy AF-converged result. Only two points (500, 600) got a clean side-by-side comparison - if the subject distance changes meaningfully, re-run the bracket (cheatsheet below). | **Kept** |

### Surprising finding: exposure_time_absolute had almost no visible effect in this scene

Sweeping `exposure_time_absolute` from 10000 (auto's ceiling) down to 500 (a
20x range) produced **no visible brightness change** in the deck/foreground,
and the sky stayed fully blown out at every value. Two things learned:

1. **Controls only apply if set before the device is opened for streaming.**
   `v4l2-ctl --set-ctrl=...` run against a device that `v4l2src`/OpenCV
   already has open for capture is accepted by the driver (readback shows
   the new value) but never reaches the sensor. The device has to be
   reopened (i.e. the capture process restarted) after the control change
   for it to actually take effect. This is *separate* from the "doesn't
   survive reboot" behavior in the table above - this is about the control
   not even applying live, regardless of persistence.
2. Once verified via a proper stop → set-ctrl → restart cycle, the
   remaining flatness across the exposure range suggests this scene's
   dynamic range (deep shade vs. direct sky) just exceeds what this 8-bit
   sensor can hold in one exposure — there's no single exposure value that
   keeps the sky un-clipped *and* the shaded deck well-exposed. Manual
   exposure at least stops the auto-algorithm from perpetually pinning
   itself at the ceiling.

### Second gotcha: combined --set-ctrl of dependent controls can fail outright

Setting a mode-switch and its dependent absolute value in the **same**
`--set-ctrl=` call - e.g.
`--set-ctrl=auto_exposure=1,exposure_time_absolute=800,gain=160,focus_automatic_continuous=0,focus_absolute=600` -
intermittently failed completely on this camera's firmware:

```
Error setting controls: Permission denied
VIDIOC_S_EXT_CTRLS: failed: Permission denied
```

...with **nothing** applied (not even the controls that had no dependency
issue). Splitting into two calls - mode switches first, dependent absolute
values second - fixed it reliably:

```bash
v4l2-ctl -d $DEV --set-ctrl=auto_exposure=1,focus_automatic_continuous=0
v4l2-ctl -d $DEV --set-ctrl=exposure_time_absolute=800,gain=160,focus_absolute=600
```

This is exactly what `simple_camera.py` does now (two `subprocess.run` calls,
in this order) - see below.

## Current live values (as of this doc)

`power_line_frequency` and the five 2026-08-10 tuning controls above are
non-default and baked into `simple_camera.py`'s `--yuyv` path. Everything
else is still factory default:

```
brightness: 128 (default)
contrast: 65 (default)
saturation: 80 (default)
hue: 0 (default)
white_balance_automatic: 1 (default)
gamma: 128 (default)
gain: 160 — CHANGED from default (128)
power_line_frequency: 2 (60Hz) — CHANGED from default (1/50Hz)
white_balance_temperature: 4650 (default, inactive - auto WB still on, looked fine)
sharpness: 128 (default)
backlight_compensation: 56 (default)
auto_exposure: 1 (Manual) — CHANGED from default (3/Aperture Priority)
exposure_time_absolute: 800 — CHANGED from default (156, and was inactive under auto)
exposure_dynamic_framerate: 0 (default)
pan_absolute: 0 (default)
tilt_absolute: 0 (default)
focus_absolute: 600 — CHANGED from default (0, and was inactive under autofocus)
focus_automatic_continuous: 0 — CHANGED from default (1/on)
zoom_absolute: 100 (default)
```

Note: this tuning lives in `gstreamer_pipeline_usb_yuyv()` only (the
`--yuyv` path, 2592x1944). The plain `usb` (MJPG, 1920x1080) path was not
part of this evaluation and is untouched - still factory defaults +
`power_line_frequency` only. If MJPG mode shows the same blown-highlights/AF
softness on your scene, the same values are a reasonable starting point but
haven't been verified there.

## Where the fixes live in code

`simple_camera.py` (`gstreamer_pipeline_usb`, `gstreamer_pipeline_usb_yuyv`) and
`dual_camera.py` (`usb_gstreamer_pipeline`) all run `v4l2-ctl` before building
the GStreamer pipeline, every time, since these settings reset to driver
defaults on unplug/reboot (see "Where a change actually lands" above) and
(per the gotcha above) don't apply at all once the device is already
streaming:

```bash
# All three usb-path pipeline functions (power_line_frequency only):
v4l2-ctl -d <device> --set-ctrl=power_line_frequency=2

# gstreamer_pipeline_usb_yuyv() additionally runs, in this order:
v4l2-ctl -d <device> --set-ctrl=auto_exposure=1,focus_automatic_continuous=0
v4l2-ctl -d <device> --set-ctrl=exposure_time_absolute=800,gain=160,focus_absolute=600
```

`dual_camera.py` and the plain `usb` (MJPG) path in `simple_camera.py` do
**not** have the exposure/focus tuning - only `power_line_frequency`.

## `--profile` - loading a guvcview .gpfl file instead (added 2026-08-10)

`simple_camera.py` now takes `--profile PATH`, pointing at a guvcview
control-profile file (`.gpfl` - File > Save Control Profile in guvcview, or
one already saved under `~/work/camera-settings/`, e.g. `sunglight.gpfl` /
`defaults.gpfl`). When passed, it **replaces** the built-in 2026-08-10
tuning above entirely for that run - only the values in the file are
applied (plus `power_line_frequency`, which is always forced separately).

```bash
python3 simple_camera.py usb --yuyv --profile ~/work/camera-settings/sunglight.gpfl
```

How it works (`GPFL_CONTROL_MAP`, `parse_gpfl_profile`, `apply_v4l2_profile`
in `simple_camera.py`):
1. Parses `ID{0x...}=VAL{...}` lines out of the `.gpfl` file and maps each
   hex control ID to its `v4l2-ctl` name via a lookup table built from this
   camera's own control IDs. `Privacy` (`0x009a0910`) is deliberately
   excluded - this camera errors querying it (`error 32 getting ext_ctrl
   Privacy`) and guvcview saves a nonsense value for it (a stray
   white-balance-Kelvin number), so it's dropped rather than fed to
   `v4l2-ctl`.
2. Drops any dependent value (`white_balance_temperature`,
   `exposure_time_absolute`, `focus_absolute`) whose governing mode control
   is staying in its "auto" state in that same profile - setting those is a
   no-op at best, a noisy `Permission denied` at worst (seen for
   `white_balance_temperature` during testing - harmless, just noisy; fixed
   by skipping it).
3. Applies what's left via `v4l2-ctl`, split into the same two passes as
   the built-in tuning (mode-switch controls first, dependent values
   second) - same firmware quirk, same fix.

Verified 2026-08-10: scrambled brightness/contrast/saturation by hand,
launched with `--profile sunglight.gpfl`, confirmed every value in the file
(`brightness=125, contrast=75, saturation=90, backlight_compensation=57,
auto_exposure=3 (kept auto), focus_automatic_continuous=1 (kept auto)`)
landed correctly and cleanly (no errors) on the device.

Note `sunglight.gpfl` keeps auto exposure and autofocus **on** (unlike the
2026-08-10 built-in tuning, which forces both manual) - it only adjusts
brightness/contrast/saturation/backlight_compensation. Different philosophy,
both valid: pick whichever profile suits the scene, or make a new one via
guvcview and save it under `~/work/camera-settings/`.

## Software that can drive these controls

| Tool | Type | Notes |
|---|---|---|
| `v4l2-ctl` (`v4l-utils`) | CLI | Installed. What this doc's cheatsheet and `simple_camera.py` both use. Scriptable. |
| `qv4l2` (`v4l-utils`) | GUI | Installed 2026-08-10 but **broken on this machine** - segfaults on startup (`InternalMakeCurrentDispatch` GLX assertion, or a plain SIGSEGV once GLX is disabled) regardless of display or GL env vars tried (`LIBGL_ALWAYS_SOFTWARE=1`, `QT_OPENGL=software`, `QT_XCB_GL_INTEGRATION=none`), on both the real `:1` (NVIDIA/Tegra driver) and the pure-software Xvfb screens (`:2`/`:3`/`:4`). Looks like a genuine Qt5-vs-this-system's-GL-stack bug, not a config issue. Use `guvcview` instead. |
| `guvcview` | GUI | Installed 2026-08-10, **works**. Live preview + full sliders for every control above, GTK-based. `DISPLAY=:1 guvcview -d /dev/video0`. **Must use the short `/dev/videoN` path, not the `by-id` path** - guvcview has a fixed-length buffer for the device-path argument and silently truncates long paths (e.g. the `by-id` path got cut to `/dev/v4l/by-id/usb-Global_Shu`, so it fails to open at all). Since `/dev/videoN` numbering can shift, double-check with `v4l2-ctl --list-devices` first if unsure which node is the Global Shutter Camera. Noisy but harmless ALSA/JACK warnings on startup (it also probes for a mic) - ignore them. |
| OpenCV (`cv2.VideoCapture.set(...)`) | Programmatic | Only exposes the common `cv2.CAP_PROP_*` subset, not every UVC control (e.g. `power_line_frequency` isn't reachable this way) - fine for exposure/focus/gain, not exhaustive. |
| GStreamer `v4l2src` `extra-controls` property | Pipeline-level | e.g. `extra-controls="c,focus_automatic_continuous=0,focus_absolute=600"` - sets controls at pipeline-open time, before streaming starts, avoiding the "already streaming" gotcha by construction. Not used in this repo's scripts (they use explicit `v4l2-ctl` calls instead, for clearer logging/error visibility) but a valid alternative. |

## Re-tuning cheatsheet (bracket a control against a fixed real-world target)

For focus specifically (exposure is less useful to bracket - see "Surprising
finding" above), the reliable way to find the sharpest `focus_absolute` for
a *specific* subject distance:

```bash
DEV=/dev/v4l/by-id/usb-Global_Shutter_Camera_Global_Shutter_Camera_2602040001-video-index0

# 1. Stop any running capture process first (controls don't apply live - see above)
pkill -f simple_camera.py

# 2. Disable continuous AF, try a value, with the device idle
v4l2-ctl -d $DEV --set-ctrl=focus_automatic_continuous=0
v4l2-ctl -d $DEV --set-ctrl=focus_absolute=<value>   # try e.g. 500, 600, 700, 800...

# 3. Relaunch and inspect the live window (or capture a frame) at each value
python3 simple_camera.py usb --yuyv &

# 4. Once you've found the sharpest value, bake it into
#    gstreamer_pipeline_usb_yuyv() in simple_camera.py (see above)
```
