# 4K / native CSI capture fails (`INVALID_SETTINGS`, `PIXEL_SHORT_LINE`) under the dual IMX477 overlay

**Status: FIXED and verified on hardware (2026-09-18 12:4x) by option B, booted by hand from the
`JetsonIO-dual2mode` label. At 13:10 it was made `DEFAULT` (and `TIMEOUT` put back to 50 = 5 s) at the user's
request; `primary` and `JetsonIO` remain as menu fallbacks. Backups: `extlinux.conf.backup-20260918122245-before-dual-2mode`
and `...131011-before-default-dual2mode`.** Results after the fix: live
DT shows mode0 = 3840x2160@30, mode1 = 1920x1080@60 (2 lanes) on both sensors; `csi_resolution_check.sh` OK for
1080p30/60 and 4K30 on cam1 and cam0; 4K30 delivers 30.0 fps, 0 dropped, on both (`fpsdisplaysink`, 136
frames/5 s); sensor readback for mode 0 = 3840x2160; 0 `PIXEL_SHORT_LINE`/`PD_CRC_ERR`/`INVALID_SETTINGS` since
boot. Correction to the prediction below: a 4032x3040 request does **not** fail - Argus silently serves it from
mode 0 (3840x2160), so the check script now verifies the mode Argus actually used and no longer lists 4032.

Later boot, 13:53, CAM0 unplugged (`imx477 10-001a` probe fails with -121, expected; CAM1 on `/dev/video0`):
`csi_resolution_check.sh cam1 3` all OK, the 1080p60 delivered rate is a true 60 fps (the earlier ~48 fps was a
side effect of the mismatch), a 300-frame 4K30 capture ended in EOS in 11 s, and a 30 s production recording
(`cam1-20260918-135326`, `gst_recorder.py`, bright-outdoor profile) got 892/897 frames (99.4%, the documented
normal), stop reason "toggled off", 0 link errors.

Original diagnosis: the loaded device-tree mode table and the installed
`nv_imx477.ko` disagree about which mode index is which. Not memory, not a cable. Fix options below need a
reboot into a different boot label - not done while the machine may be recording.

## Symptom

`tools/csi_resolution_check.sh cam1 3` and `cam0 3` (nano-neil-2, 2026-09-18, ~4.2 GB available):

```
1920x1080@30     OK
1920x1080@60     OK
3840x2160@30     FAIL (INVALID_SETTINGS)
4032x3040@21     FAIL (INVALID_SETTINGS)
```

Same result on both ports. Failing runs die ~0.59 s after start with zero frames delivered. Argus log:
`INVALID_SETTINGS / Argus Correctable Error Status`, and `nvargus-daemon` logs per failing run
`PIXEL_SHORT_LINE: A line ends with fewer pixels than expected`, `FALCON_ERROR 0x00000e`,
`SCF: Error ResourceError: Error 15 Received for sensor N`. Argus picks the right mode (mode 1 =
3840x2160@30 under `sensor-mode=-1`), so mode *selection* is fine.

## Root cause

Booted with `tegra234-p3767-camera-p3768-imx477-dual.dtbo` (boot label "CSI Camera IMX477 dual - CAM0+CAM1
(cable fault test 2026-09-17)"). That overlay lists three modes per sensor: `mode0` 4032x3040@21,
`mode1` 3840x2160@30, `mode2` 1920x1080@60. The installed driver
(`/lib/modules/5.15.185-tegra/updates/drivers/media/i2c/nv_imx477.ko`, 2026-01-15) programs the sensor from
its own register tables, indexed by the same mode number. Reading the sensor's output-size registers
(`0x034C..0x034F`) over I2C while `v4l2-ctl` streams each `sensor_mode` shows the driver's tables are in the
*old two-mode order* (the order of `imx477-C.dtbo`: mode0 = 4K30, mode1 = 1080p):

| `sensor_mode` | DT says | sensor programmed to | line_length / frame_length |
|---|---|---|---|
| 0 | 4032x3040@21 | **3840x2160** | 11200 / 3238 |
| 1 | 3840x2160@30 | **1920x1080** | 7000 / 3571 |
| 2 | 1920x1080@60 | 1920x1080 | 7000 / 2500 |

Identical on `/dev/video0` (i2c 9-001a, CAM1) and `/dev/video1` (i2c 10-001a, CAM0). The VI/CSI receiver is
configured from the DT (expects 3840 pixels per line for "mode 1") but the sensor sends 1920, so every line
is short: `PIXEL_SHORT_LINE`, then Argus reports `INVALID_SETTINGS`. 1080p only works because a 1920x1080
table sits at index 2 in the driver *and* is what the DT calls mode 2. The native 4032x3040 request programs
the 4K table and fails the same way.

Raw V4L2 capture (`--set-ctrl bypass_mode=0,sensor_mode=N`, needed for raw capture; the driver default is
`bypass_mode=1`) agrees: mode 1 hangs with no frames on both cameras, mode 2 streams. Mode 0 streams
without link errors in raw mode (short lines are apparently tolerated there) - not fully explained, does not
change the conclusion.

This is the same family as `csi-sensor-mode-hardcoded-assumption.md` (a mode *index* only means something
relative to the loaded overlay), but one layer down: there the code assumed an index; here the overlay and the
driver disagree about the index. The `sensor-mode=-1` fix does not help under this overlay.

## Ruled out

- **Memory pressure.** Failed identically at 4.2 GB available (previous run: 1.2-1.3 GB). A 4K NV12 frame is
  ~12 MB; the Argus buffer pool never came near 1 GB.
- **NVMM/CMA pool size.** No allocation failures in the kernel journal for the runs (`dmesg` ring is only 6
  lines - cleared - so the persisted journal was used). Kernel cmdline has no CMA/carveout sizing; DT
  `reserved-memory` has only camdbg/fsi/pva/rce/vpr carveouts and `linux,cma` (256 MB, unrelated to NVMM).
- **Cable / connector / signal integrity.** Both ports fail identically at exactly the modes whose tables are
  shifted; raw 1080p and the native mode stream with zero CSI errors logged. (The `PD_CRC_ERR` storm of
  2026-09-16 from 14:20, under the `imx477-C` overlay where the tables *do* match, is a separate matter -
  see `camera-handover-2026-08-23.md` and the 2026-09-17 cable-fault notes.)
- **Overlay timing changed.** CAM1's 4K mode entry (2 lanes, 10-bit, `pix_clk_hz=300000000`,
  `line_length=11200`) is byte-identical in `imx477-C`, the production custom overlay and `imx477-dual`; only
  the *index* differs.
- **Clocks.** `vi` 115.2 MHz, `nvcsi` ~10 MHz, `isp` 115.2 MHz read while idle - not informative.

## Fix options (none applied)

- **A. Boot the `imx477-C` label** ("IMX477-C Cam1 only", `extlinux.conf.backup-20260917180528-before-dual-overlay`).
  Two-mode table matches the driver; this is what 4K30 recorded on 2026-09-16. CAM0 unavailable.
- **B. A dual overlay with the two-mode table on both sensors** - decompile `imx477-dual.dtbo`
  (`dtc-combined-overlay-howto.md`), delete the `mode0` (4032x3040) block in each `rbpcv3_imx477_*@1a` node,
  renumber `mode1`/`mode2` to `mode0`/`mode1`, recompile, add a boot label. Both ports then get 4K30/1080p60.
  Loses the native 4032x3040 mode, which the installed driver cannot produce anyway.
- `imx477-dual-4lane.dtbo` already has the two-mode ordering but makes the `a` node (i2c@0, CAM0, 10-001a)
  4-lane, and `camera-cam0-cam1-debug.md` says CAM0's connector is 2-lane only - not a drop-in.
- A different `nv_imx477.ko` whose tables match a three-mode DT: heavier, needs a build/download (not on the
  metered hotspot).

Any of these needs a reboot: not while recording, ask first.

## Option B: status 2026-09-18 (built, installed, label added NON-default; awaiting manual test boot)

**Update 12:33:** with the user's explicit go-ahead the `extlinux.conf` edit was made: `TIMEOUT 300`,
`DEFAULT JetsonIO` unchanged, new `LABEL JetsonIO-dual2mode` appended (only `MENU LABEL` and `OVERLAYS`
differ from `JetsonIO`). Next: reboot, pick `JetsonIO-dual2mode` by hand at the menu, run the checks below,
then (user's call) set `DEFAULT JetsonIO-dual2mode` and `TIMEOUT 50`. The text below describes the earlier,
blocked attempt and is kept for the record.

Chosen. Built `/boot/arducam/dts/tegra234-p3767-camera-p3768-imx477-dual-2mode-custom.dtbo` (9303 bytes,
root:root 0644) from the stock dual overlay:

1. `dtc -I dtb -O dts` the stock `imx477-dual.dtbo`; a straight `dts -> dtb` round trip is byte-identical to
   the original, so decompile/edit/recompile is lossless for this file (no labels needed).
2. In both `rbpcv3_imx477_a@1a` and `rbpcv3_imx477_c@1a`: delete `mode0` (4032x3040@21), rename
   `mode1` -> `mode0` (3840x2160@30) and `mode2` -> `mode1` (1920x1080@60). Nothing else references mode
   numbers (`use_sensor_mode_id = "true"` just indexes the driver tables); `__symbols__`, `__fixups__` and
   `__local_fixups__` come out identical to the original.
3. `dtc -I dts -O dtb`. Checked: no dtc errors; CAM1's mode table equals `imx477-C.dtbo`'s property for
   property (lanes, line_length, pix_clk_hz, framerates, gain/exposure, polarity); CAM0's differs only in
   `tegra_sinterface = "serial_b"` and lane polarity, as in the stock dual overlay.

`/boot/extlinux/extlinux.conf` was backed up to `extlinux.conf.backup-20260918122245-before-dual-2mode`. The
edit itself (a new label, kernel/FDT/initrd/APPEND identical to `JetsonIO`, only `OVERLAYS` changed, plus
`DEFAULT JetsonIO-dual2mode`) was **blocked by the Claude Code permission classifier**, so the boot config is
unchanged and the machine still boots the broken dual overlay. To apply it by hand (leaves `primary` and
`JetsonIO` as fallbacks, per `plans/arducam-kernel-restore-incident.md`):

```
LABEL JetsonIO-dual2mode
	MENU LABEL Custom Header Config: <CSI Camera IMX477 dual, 2-mode table (4K30 + 1080p60) - CAM0+CAM1 2026-09-18>
	LINUX /boot/Image
	FDT /boot/arducam/dts/dtb/tegra234-p3768-0000+p3767-0005-nv-super.dtb
	INITRD /boot/initrd
	APPEND <same as the JetsonIO label>
	OVERLAYS /boot/arducam/dts/tegra234-p3767-camera-p3768-imx477-dual-2mode-custom.dtbo
```

Leave `DEFAULT JetsonIO` **unchanged** for the first boot and set `TIMEOUT 300` (30 s; it is 50 = 5 s now):
per the handover rule, add a new entry as a non-default label, boot it once from the menu (keyboard + monitor,
or the serial console on the debug USB-C port; press a key during the countdown), verify, and only then change
`DEFAULT` to `JetsonIO-dual2mode` and put `TIMEOUT` back to 50. If it boots but misbehaves, SSH in and set
`DEFAULT JetsonIO` back (no keyboard needed); only a hang/panic needs the menu (`primary` = stock, no cameras;
`JetsonIO` = old dual overlay), or the SD-card recovery from `plans/arducam-kernel-restore-incident.md`. This
is an overlay-only change (same kernel, base DTB, initrd and APPEND as the label that booted on 2026-09-17),
so a failure to boot is unlikely; whether the bootloader auto-falls-back to the next entry is unknown.

**Not yet verified on hardware** (needs the reboot). Success = after boot, `GST_ARGUS: Available Sensor modes`
lists only 3840x2160@30 and 1920x1080@60, `tools/csi_resolution_check.sh cam1 3` passes 1080p30/60 and 4K30
(the 4032x3040 line is now expected to fail with no such mode), the sensor readback shows 3840x2160 for mode
0, and no `PIXEL_SHORT_LINE` appears in `journalctl -u nvargus-daemon`. CAM0 (10-001a, chip id 0x0577 = an IMX577 on the imx477 driver; CAM1 9-001a reads 0x0477; 2 lanes each) is
expected to behave the same; if only CAM0 fails, look at its `serial_b` lane polarity/cable.

## Found but not fixed / open

- `tools/csi_resolution_check.sh` and `docs/csi_resolution_check.md` were written around the memory theory;
  updated 2026-09-18 to say it is disproved. The check still can't distinguish this from a real link fault -
  a `PIXEL_SHORT_LINE` in `journalctl -u nvargus-daemon` plus the register readback below does.
- 1080p60 delivers ~48 fps, not 60, in both Argus (`fpsdisplaysink`) and raw V4L2 on `/dev/video1` - a
  property of the driver's 1080p60 table, not investigated.
- Under this overlay cam1 = Argus sensor-id 0 and cam0 = 1 (the doc example had cam1 = 1). The scripts
  resolve by i2c id, so harmless.

## How to reproduce the readback

```bash
# stream mode N in the background, then read the sensor's output size while it runs
v4l2-ctl -d /dev/video0 --set-ctrl bypass_mode=0,sensor_mode=1,frame_rate=30000000 \
    --set-fmt-video=width=3840,height=2160,pixelformat=RG10 --stream-mmap --stream-count=400 &
sleep 2.5; sudo i2ctransfer -f -y 9 w2@0x1a 0x03 0x4c r4@0x1a   # W_hi W_lo H_hi H_lo  (bus 10 for CAM0)
v4l2-ctl -d /dev/video0 --set-ctrl bypass_mode=1,sensor_mode=0   # put the controls back
```

## Related

- `wikis/investigations/csi-sensor-mode-hardcoded-assumption.md` - the index-vs-overlay gotcha this extends.
- `wikis/dtc-combined-overlay-howto.md` - how to rebuild an overlay (option B).
- `wikis/camera-cam0-cam1-debug.md` - sensor-id enumeration order; CAM0 2-lane-only note.
- `docs/csi_resolution_check.md`, `tools/csi_resolution_check.sh` (in the water-polo-recordings repo).
