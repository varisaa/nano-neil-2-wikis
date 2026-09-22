# A stray `v4l2-ctl` holding /dev/videoN fakes a CSI hardware fault (`INVALID_SETTINGS`, `PIXEL_SHORT_LINE`)

**Status: root-caused and resolved (2026-09-22).** Both CAM0 and CAM1 are healthy: 4K30 and
1080p60, zero link errors, neutral colour. No cable, connector or overlay work was needed. The
apparent CAM0 hardware fault was self-inflicted by a debugging process left running in the same
session.

This is the **third** distinct cause of the same `PIXEL_SHORT_LINE` signature on this machine, after
`csi-dual-overlay-mode-table-mismatch.md` and the genuine 2026-09-16/17 cable faults. It is the one
the existing triage rule did not cover.

## Symptom

Testing both CSI ports over SSH (Argus via `nvarguscamerasrc`):

```
CAM0 (sensor-id 0, i2c 10-001a, /dev/video1)   4K30    FAIL (INVALID_SETTINGS)
CAM0                                           1080p60 OK, 60 s soak, 0 errors
CAM1 (sensor-id 1, i2c 9-001a, /dev/video0)    4K30    OK
CAM1                                           1080p60 OK, 60 s soak, 0 errors
```

Failing CAM0 4K runs completed Argus setup, started captures, then died ~0.27 s in. `journalctl -u
nvargus-daemon` showed, on every frame:

```
PIXEL_SHORT_LINE: A line ends with fewer pixels than expected.
ChanselFault 0x000200
FALCON_ERROR 0x00000e
SCF: Error ResourceError: Error 15 Received for sensor N ..!
 (in src/services/capture/FusaCaptureViCsiHw.cpp, function waitCsiFrameEnd(), line 664)
```

while the **kernel log stayed completely empty** — no `corr_err`, no `PD_CRC_ERR`, no I2C timeouts,
no `Error writing mode`.

This looks exactly like marginal CSI signal integrity, and was initially (wrongly) diagnosed as a
marginal CAM0 ribbon contact — the per-lane rate roughly doubles from 1080p60 (~0.62 Gbps/lane) to
4K30 (~1.24 Gbps/lane) on 2 lanes, which makes "fine at the low rate, breaks at the high rate" a very
tempting story.

## Root cause

An earlier step in the same session tried to identify which `/dev/videoN` belongs to which Argus
`sensor-id` by starting an Argus stream and then probing each node to see which one reported busy.
That probe was written as a bare `v4l2-ctl ... --stream-mmap --stream-count=1` with **no `timeout`**.
Against a node whose VI channel was already claimed, it blocked instead of returning `EBUSY`, the
wrapper script was killed, and the probe was orphaned:

```
PID 2469  v4l2-ctl -d /dev/video1 --set-fmt-video=width=1920,height=1080,pixelformat=RG10 \
                   --stream-mmap --stream-count=1 --stream-to=/dev/null
```

It held `/dev/video1` (CAM0) for the rest of the session with the **VI capture channel configured for
a 1920-pixel line width**. Every subsequent Argus 4K30 request on CAM0 programmed the sensor for 3840
pixels per line while the VI channel still expected 1920 — so every line was "short", exactly the
`PIXEL_SHORT_LINE` condition.

It accounts for every detail:

- **only CAM0 failed** — only CAM0's node was held;
- **CAM0 1080p60 was flawless** — it matched the stuck 1920-wide configuration;
- **the kernel log was empty** — nothing was electrically wrong;
- **nothing helped** — `systemctl restart nvargus-daemon` does not evict a V4L2 client holding a VI
  channel, and the fault is upstream of every Argus-side knob.

## Fix

```bash
sudo fuser -v /dev/video0 /dev/video1     # identify the holder
sudo kill -9 2469
sudo systemctl restart nvargus-daemon
```

CAM0 delivered 4K30 immediately afterwards. Verification, same boot, nothing else changed:

| Check | CAM0 | CAM1 |
|---|---|---|
| 4K30, 20 stills | 20/20, ~2.19 MB each | — |
| 4K30, 300 frames, wall time | **17.0 s** | **16.8 s** |
| 4K30, 40 full-quality stills | 40/40, 3840x2160 | — |
| `PIXEL_SHORT_LINE` / `PD_CRC_ERR` / `ChanselFault` / `ResourceError` | 0 | 0 |
| kernel `corr_err` / I2C timeouts | 0 | 0 |

## Ruled out (all while the stray holder was still running, all red herrings)

- `systemctl restart nvargus-daemon` before the run.
- Explicit `sensor-mode=0`; 4K at 21 fps instead of 30; `aelock=true` with fixed
  `exposuretimerange`/`gainrange`.
- Property combinations: bare pipeline, `ee-mode=0`, `ee-mode=0 tnr-mode=2 tnr-strength=1`.
- Device tree. A full recursive diff of the two sensor nodes under
  `/proc/device-tree/bus@0/cam_i2cmux/` shows only `lane_polarity` (CAM0 6, CAM1 0),
  `tegra_sinterface` (`serial_b` vs `serial_c`), `name`, `devnode` and phandles. Both declare an
  identical two-mode table (mode0 3840x2160@30, mode1 1920x1080@60), and Argus advertises identical
  modes for both and selects mode 0 for both.
- The mode-table mismatch of `csi-dual-overlay-mode-table-mismatch.md`. That was fixed on 2026-09-18
  by `imx477-dual-2mode-custom.dtbo`, still the default boot label, and it would affect **both**
  ports, not one.
- Raw V4L2 link quality: `bypass_mode=0` 1080p, 60 frames, ran at a steady 30.00 fps with zero kernel
  errors on **both** ports.

## Updated triage for `PIXEL_SHORT_LINE`

`PIXEL_SHORT_LINE` means "the VI expected more pixels per line than arrived". That is a **width /
configuration** mismatch far more often than a cable fault. In order of likelihood on this machine:

1. **A stray process holding that port's VI channel at another width** — this page. Check
   `sudo fuser -v /dev/video*` **first**; it costs one command.
2. **Overlay / driver mode-table mismatch** — `csi-dual-overlay-mode-table-mismatch.md`. Affects
   **both** ports at the same modes.
3. **A genuine link fault** — cable, connector, sensor board. Normally also shows `PD_CRC_ERR` and
   kernel-side `corr_err` / `discarding frame`, plus I2C mode-write timeouts when bad. Neither
   appeared here.

The older rule in `docs/csi_resolution_check.md` ("both ports fail = mode table, one port fails =
cable/connector") is still useful but is missing case 1, which presents as a single-port failure.

## Two techniques worth keeping

**Map Argus `sensor-id` to a physical port from the device tree, not by probing.** The busy-node
probe that caused this whole mess does not even work — Argus locks the whole VI, so *both* nodes read
busy. Instead, module order in `tegra-camera-platform` **is** Argus enumeration order:

```bash
for m in /proc/device-tree/tegra-camera-platform/modules/module*; do
  echo "$m -> $(tr -d '\0' < $m/drivernode0/sysfs-device-tree)"
done
```

On nano-neil-2 under `JetsonIO-dual2mode`:

| Argus sensor-id | DT node | i2c | /dev/video |
|---|---|---|---|
| 0 | `cam_i2cmux/i2c@0/rbpcv3_imx477_a@1a` | `10-001a` = CAM0 | video1 |
| 1 | `cam_i2cmux/i2c@1/rbpcv3_imx477_c@1a` | `9-001a` = CAM1 | video0 |

This does not depend on how many sensors bound, so it also sidesteps the enumeration trap described
in `camera-cam0-cam1-debug.md`.

**Tell "no image" from "real image" without looking at the picture.** A covered lens at maximum AE
gain produces structureless noise, which does *not* JPEG-compress — frames come out large and
suspiciously constant (~1.44 MB at 1080p here), easily misread as rich detail. The std of 8x8
block means separates them cleanly: **~4 = no image, ~50-60 = real scene**. This bit us on
2026-09-22 when CAM0's lens cover was still on during the first soak.

## Related

- `csi-dual-overlay-mode-table-mismatch.md` — same signature, different cause (cause 2 above).
- `csi-sensor-mode-hardcoded-assumption.md` — a mode *index* only means something relative to the
  loaded overlay.
- `../camera-cam0-cam1-debug.md` — the CAM0/CAM1 port mapping and the "sensor-id is enumeration
  order" warning.
- `docs/csi_resolution_check.md` in water-polo-recordings — the triage rule updated above.
- Test images and full session write-up on the hub:
  `water-polo-detection/data/bench-test/2026-09-22-nano-neil-2-cam0-cam1/`.
