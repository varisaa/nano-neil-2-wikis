# CSI sensor-mode=0 hardcoded to a stale device-tree overlay's mode table

**Status: root-caused and fixed.** `sensor-mode` now auto-selects
(`-1`) instead of hardcoding an index tied to one specific overlay.

## Symptom

Launching CAM0 through the real production script chain
(`camera-launch-cam0.sh` → `camera-launch.sh` → `recorder/gst_recorder.py`)
crashed immediately instead of starting a preview:

```
ARGUS_ERROR: Error generated. .../gstnvarguscamerasrc.cpp, execute: 969 Frame Rate specified is greater than supported
GST_ARGUS: Running with following settings:
   Camera index = 1
   Camera mode  = 0
   Output Stream W = 4032 H = 3040
   Frame Rate = 21.000000
...
ARGUS_ERROR: Error generated. .../gstnvarguscamerasrc.cpp, execute: 1231 InvalidState.
GST_ARGUS: Cleaning up
```

Triggered by:
`camera-launch.sh 1 --csi-profile settings/csi/default.csiprofile`
(equivalently, `camera-launch-cam0.sh --csi-profile ...` once that script
existed — see Related).

## Root cause

`gst_recorder.py`'s `csi_camera()` function builds the `nvarguscamerasrc`
element with a hardcoded `sensor-mode=0` and hardcoded caps
`width=3840,height=2160,framerate=30/1`, based on a comment asserting
"sensor-mode=0 is the IMX477 driver's 3840x2160@30 mode." Both literals
sit directly in the element string (previously ~line 237), outside the
normal `.csiprofile`-overridable `CSI_PROPERTIES`/`settings` mechanism —
a `.csiprofile` cannot touch `sensor-mode` at all today, by design or
oversight.

That assumption was true under the production device-tree overlay
(`imx296-cam0-imx477-cam1-custom.dtbo`, the hand-authored combined overlay
documented in the 2026-08-23 camera handover) at the time it was written
(2026-09-14, per the code comment). It stopped being true the moment a
*different* overlay was loaded: `sensor-mode` is an index into a table
defined by each sensor's device-tree node (`mode0{}/mode1{}/...` child
nodes), not a fixed constant owned by the kernel driver. The driver
(`imx477_v2.0.6`) is shared code; the mode table is DT data.

On 2026-09-17, CAM1's board was being debugged for a suspected connector
fault, and the board was temporarily booted with a different overlay —
`tegra234-p3767-camera-p3768-imx477-dual.dtbo` (extlinux label "CSI
Camera IMX477 dual - CAM0+CAM1 (cable fault test 2026-09-17)") — to run
both ports as plain dual-IMX477 for isolation testing. Under *this*
overlay, Argus reports:

```
GST_ARGUS: Available Sensor modes :
GST_ARGUS: 4032 x 3040 FR = 21.000000 fps ...
GST_ARGUS: 3840 x 2160 FR = 29.999999 fps ...
GST_ARGUS: 1920 x 1080 FR = 59.999999 fps ...
```

i.e. mode 0 is now 4032×3040@21fps (mode 1 is the 3840×2160@30fps the
code actually wants). Requesting 30fps while forcing mode 0 fails
outright with "Frame Rate specified is greater than supported," which
Argus escalates to `InvalidState` and tears the whole pipeline down.

This is a distinct gotcha from, but compounds with, the already-documented
"`sensor-id` is an Argus enumeration index, not a fixed physical mapping"
caveat in `wikis/camera-cam0-cam1-debug.md` — that one is about *which*
sensor answers to a given `sensor-id`; this one is about what a given
*mode index* means for whichever sensor is there, under whichever overlay
is currently loaded.

## Not the cause (ruled out)

- Not a hardware, cable, or lens fault — CAM1's board itself was
  separately, exhaustively verified working that same night (raw V4L2
  capture, Argus capture, 5x and alternating stress tests, all clean)
  once properly lens-mounted. This crash is unrelated to that investigation.
- Not specific to CAM0 as a physical port — the same hardcoded line runs
  for CAM1 too; it simply hadn't been exercised against this particular
  overlay's mode table yet when CAM1 was tested via the real launch
  scripts.

## Fix

Changed `sensor-mode=0` to `sensor-mode=-1` (Argus's "auto-select the
mode that best matches the requested caps" value) in
`recorder/gst_recorder.py`'s `csi_camera()`. The caps
(`width=3840,height=2160,framerate=30/1`) are unchanged — they already
correctly express the actual intent ("give me 4K30"); only the previously
hardcoded index is now resolved dynamically against whichever mode table
the currently-loaded overlay actually defines. This makes the pipeline
resilient to overlay changes instead of silently assuming one specific
overlay's mode ordering forever.

Also added `camera-launch-cam0.sh` (mirroring the existing
`camera-launch-cam1.sh`) and a `CAM0_I2C=10-001a` entry in
`machines/nano-neil-2/machine.env`, so launching CAM0 no longer requires
manually resolving its current Argus `sensor-id` by hand — the same
enumeration-order problem `camera-launch-cam1.sh` already solved for
CAM1, just not previously mirrored for CAM0.

## Found but not fixed in this pass

- The same stale "sensor-mode=0 → 3840x2160@30" assumption is echoed as
  prose/comments in `docs/recorder.md:15`, `tools/bench_gst.sh:77`, and
  `gst_recorder.py`'s own module docstring — none of these are
  executable, so left alone here, but worth a documentation pass.
- A separate, adjacent latent bug in the same function: the
  `width=3840,height=2160` literal in the caps filter does not pick up
  the `width, height` variables computed a few lines earlier from
  `flip_method` (portrait vs. landscape) — so a portrait-oriented CSI
  capture would still request landscape-dimension caps. Not touched here;
  out of scope of this specific crash.
- CAM0 is not wired into `camera-profile-menu.sh`'s dynamic CSI profile
  menu. Deferred because the existing `.csiprofile` files are all
  written/tuned for CAM1's IMX477 characteristics specifically, and
  CAM0's production sensor is a different chip (IMX296, single fixed
  mode) where those presets may not apply as-is.

## Follow-up 2026-09-18: `-1` is not enough under the dual overlay

With `sensor-mode=-1`, Argus correctly picks mode 1 (3840x2160@30) under the dual overlay, but 4K and native
capture still fail (`INVALID_SETTINGS`, `PIXEL_SHORT_LINE`). Reading the sensor's registers shows the
installed `nv_imx477.ko` programs its tables in the old two-mode order, so "mode 1" (DT: 4K) actually sends
1920x1080. Not memory (the earlier guess: failed identically at 4.2 GB available) and not a cable. See
`csi-dual-overlay-mode-table-mismatch.md` for the evidence and fix options. The `-1` change is still right
(and correct under `imx477-C`); it just cannot fix an overlay/driver mismatch.

## Related

- `wikis/investigations/csi-dual-overlay-mode-table-mismatch.md` — the follow-up above.
- `wikis/camera-cam0-cam1-debug.md` — the sensor-id enumeration-order
  caveat this finding compounds with.
- `wikis/camera-handover-2026-08-23.md` — documents the production
  hand-authored overlay this script's original mode-0 assumption was
  correct under.
