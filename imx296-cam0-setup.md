# Inno-Maker IMX296 on CAM0 — setup notes (2026-07-18)

## Goal
Attach an Inno-Maker **CAM-IMX296Color-GS** (Sony IMX296, global shutter, Color/IMX296LQR variant) to **CAM0**.
CAM1 currently has a working Inno-Maker **IMX577** module (confirmed clean via `sensor-id=0` — see
`camera-cam0-cam1-debug.md` Session 8; CAM0 was empty at the time, so the only bound sensor enumerated as
Argus index 0).

## Known-good baseline (backed up before touching anything)
- Active `/boot/extlinux/extlinux.conf` at time of writing: `OVERLAYS /boot/arducam/dts/tegra234-p3767-camera-p3768-imx477-dual.dtbo`, `FDT /boot/arducam/dts/dtb/tegra234-p3768-0000+p3767-0005-nv-super.dtb`.
- **Backed up to**: `/boot/extlinux/extlinux.conf.backup-20260718224738-imx477-dual-working` — restore this file (copy back over `/boot/extlinux/extlinux.conf`) to return to the known-working imx477-dual state if an IMX296 experiment goes wrong.
- Other pre-existing backups found on this system (not created by me, unclear provenance/history):
  - `/boot/extlinux/extlinux.conf.jetson-io-backup` (Jun 13)
  - `/boot/extlinux/extlinux.conf.manual-backup` (Jul 17 16:52)
  - `/boot/extlinux/extlinux.conf.manual-backup-imx296` (Jul 17 17:20) — this one has `OVERLAYS /boot/tegra234-p3767-camera-p3768-imx477-A.dtbo,/boot/tegra234-p3767-camera-p3768-imx296-cam1.dtbo` (CAM0=imx477-A, **CAM1=imx296 color**). User says this earlier attempt **did not work** — treat as an unverified/failed config, not a template to trust blindly.
  - `/boot/extlinux/extlinux.conf.nv-update-extlinux-backup` (Apr 5)

## Driver package already staged for this exact kernel
- `/home/neil/drivers/imx296_unified_binary_working_20260615_v1.1_k5.15.185-tegra/` — prebuilt for kernel `5.15.185-tegra` (matches `uname -r` on this board exactly).
- Package installs (per its `install_binary.sh`):
  - `imx296.ko` (auto-detects color vs mono chip) → `/lib/modules/5.15.185-tegra/kernel/drivers/media/i2c/imx296.ko`
  - `tegra-camera.ko` (Y10-patched, color-path unchanged) → `/lib/modules/5.15.185-tegra/updates/drivers/media/platform/tegra/camera/tegra-camera.ko`
  - 6 overlays → `/boot/tegra234-p3767-camera-p3768-imx296-{cam0,cam1,dual}.dtbo` (color/RG10) and `imx296mono-{cam0,cam1,dual}.dtbo` (mono/Y10)
  - Stamps each `.dtbo`'s `jetson-header-name` to match this machine's actual CSI header name (detected via `config-by-hardware.py -l`) — necessary because jetson-io silently hides overlays whose `jetson-header-name` doesn't match.
  - `/etc/modules-load.d/imx296.conf` (loads `imx296` at boot)
  - `imx296-reload.service` (reloads `imx296` + restarts `nvargus-daemon` every boot — installed to work around a known post-reboot black-screen issue)
- **Confirmed already installed on this system**: all 6 overlays exist directly under `/boot/`, `imx296.ko` and patched `tegra-camera.ko` are present under `/lib/modules/5.15.185-tegra/...`. Installation itself does not need to be redone — only overlay *selection* (via jetson-io) + reboot.

## Current live state (before any reboot into an IMX296 config)
- `imx296.ko` shows as **loaded** in `lsmod` right now, alongside `nv_imx477` — but there is **no bound imx296 device**: `v4l2-ctl --list-devices` only shows `imx477 9-001a` (CAM1/IMX577), and `/proc/device-tree` has no `imx296` node at all. The currently active DT is still plain `imx477-dual` (no IMX296 node exists in it), so this module is just sitting loaded/inert — harmless, but a reminder that "module loaded" ≠ "camera bound."

## Open risk flagged, not yet resolved
The old (failed) `imx296-cam1` backup config points `FDT` at **`/boot/dtb/kernel_tegra234-p3768-0000+p3767-0005-nv-super.dtb`** (sha256 `60d07abb...`), which is a **different file** from the currently-active Arducam-customized base DTB **`/boot/arducam/dts/dtb/tegra234-p3768-0000+p3767-0005-nv-super.dtb`** (sha256 `31516668...`, size differs too — 249353 vs 249899 bytes). Not yet diffed for what Arducam's base DTB customizes vs the stock one. This is one plausible reason the earlier CAM1 attempt "did not work" (wrong/mismatched base DTB), but not confirmed — could equally have been an overlay-not-listed-in-jetson-io issue, a lane/config mismatch, or something else entirely. **Do not reuse that old backup's FDT path without re-checking this.**

## Decisions made (user, 2026-07-18)
- Sensor variant: **Color** (IMX296LQR) — matches `tegra234-p3767-camera-p3768-imx296-cam0.dtbo` (not the `-mono` variant).
- Test scope: **IMX296 alone on CAM0 first** — do not combine with CAM1/IMX577 yet. Isolate one variable at a time, same methodology used throughout the CAM0/CAM1 debugging in `camera-cam0-cam1-debug.md`.

## Progress — overlay selected (2026-07-18, awaiting reboot)
1. **Done**: `sudo python3 /opt/nvidia/jetson-io/config-by-hardware.py -l` confirmed the stamping worked — Header 2 (Jetson 22pin CSI Connector) lists all 14 expected entries including `Camera IMX296-C Cam0` (#5). No re-stamping needed.
2. **Done**: Selected CAM0-only color overlay:
   ```
   sudo python3 /opt/nvidia/jetson-io/config-by-hardware.py -n '2=Camera IMX296-C Cam0'
   ```
   Result: `Modified /boot/extlinux/extlinux.conf to add following DTBO entries: /boot/tegra234-p3767-camera-p3768-imx296-cam0.dtbo` — `extlinux.conf` verified afterward, label is well-formed (`MENU LABEL`/`LINUX`/`FDT`/`INITRD`/`APPEND`/`OVERLAYS` all present, so `OVERLAYS` won't be silently ignored). `FDT` is `/boot/dtb/kernel_tegra234-p3768-0000+p3767-0005-nv-super.dtb` — jetson-io's standard baseline DTB (this is just how jetson-io always writes it, not specific to the earlier failed attempt).
   **Not yet rebooted.**
3. **Next**: reboot, then re-run the same verification battery used throughout this debugging effort: `v4l2-ctl --list-devices`, `journalctl -k -b | grep -iE "imx296|-121|nvcsi"`, then an isolated `gst-launch-1.0 nvarguscamerasrc sensor-id=0 ! 'video/x-raw(memory:NVMM),width=1456,height=1088,framerate=30/1' ! nvvidconv ! fakesink` (1456×1088 is IMX296's native resolution, not 1920×1080).
4. If it fails, compare actual failure signature (I2C-level vs buffer-level, same diagnostic approach used for the IMX577/CAM1 investigation) before assuming it's the same root cause as the old failed attempt.
5. Only after CAM0/IMX296 alone is confirmed working, consider a combined CAM0=imx296 + CAM1=imx477-C overlay for simultaneous dual-camera use.

## Result — CAM0/IMX296 confirmed working (2026-07-18)
User attached the IMX296 to CAM0 with the board fully powered off, then powered on.

- Boot log: `imx296 9-001a: probing v4l2 sensor at addr 0x1a` → driver read the chip ID registers and correctly identified the exact physical part: `Detected IMX296LQ (Color)`, `found IMX296LQ (26.8C)` — then `tegra-capture-vi: subdev imx296 9-001a bound`. Clean bind, no `-121`/no-ACK errors. (Bus enumerated as `9-001a` this time, not `10-001a` — expected: this is a single-channel CAM0-only overlay, so the i2c-mux channel numbering differs from the dual overlay's 2-channel case. The physical port itself is correct — confirmed by the fact that I2C reached the sensor at all, since the mux channel selects which physical connector is live.)
- `v4l2-ctl --list-devices`: single entry, `vi-output, imx296 9-001a` → `/dev/video0`. (Note: CAM1 is inactive in this boot — expected, this is the CAM0-only overlay, per the plan.)
- Isolated capture test, `sensor-id=0`, native resolution 1456×1088: `GST_ARGUS: Available Sensor modes: 1456 x 1088 FR = 60.375671 fps`, producer connected, ran cleanly to EOS. dmesg showed only normal sensor register traffic (gain/exposure/frame-rate/group-hold) — zero errors, only the usual benign `CANCELLED`/"Argus Correctable Error Status" shutdown artifact.

**IMX296 (Color) on CAM0 is confirmed working**, using the prebuilt driver package + jetson-io-selected `imx296-cam0.dtbo` overlay. This is a stronger result than the earlier failed CAM1 attempt — worth noting for future reference in case that combination is revisited.

**Next step (not yet done)**: build/select a combined overlay (`imx477-C.dtbo` for CAM1 + `imx296-cam0.dtbo` for CAM0) to run both cameras simultaneously, since CAM1/IMX577 is currently inactive under this CAM0-only overlay.

## Combined overlay — custom-authored (2026-07-18)

No prebuilt IMX296+IMX477 combo exists (jetson-io's "X and Y" entries are single pre-built dtbo files for specific pairs it ships — IMX219+IMX477, not IMX296+IMX477). Decompiling both single-camera files revealed they'd conflict if just concatenated: **both `imx477-C.dtbo` and `imx296-cam0.dtbo`, built as standalone single-camera overlays, independently default to `nvcsi channel@0`** — putting them both in one `OVERLAYS` line would have both cameras fighting over the same CSI channel.

**Fix**: instead of combining the two standalone single-camera files, I took the original **`imx477-dual.dtbo`** (the proven-working template that already correctly splits CAM0 on `channel@0`/`i2c@0` and CAM1 on `channel@1`/`i2c@1`) and swapped only the CAM0-side sensor block from `imx477_a` (IMX477) to `imx296_a` (IMX296) — CAM1's `imx477_c` block (channel@1, i2c@1) is untouched, byte-for-byte identical to the working dual overlay.

Key changes made to the CAM0 side, sourced from the proven-working `imx296-cam0.dtbo`:
- Added `imx_296_fixed_cam_clk` fixed-clock node (54MHz) — IMX296 needs an explicit clock reference the IMX477 driver doesn't use.
- Replaced `rbpcv3_imx477_a@1a` (compatible `ridgerun,imx477`) with `rbpcv3_imx296_a@1a` (compatible `sony,imx296`), single `mode0` (1456×1088, 1 lane) instead of imx477's 3 modes (2 lanes).
- `bus-width`/`num_lanes` on the CAM0-side VI/CSI endpoints changed from 2 (imx477) to 1 (imx296) — CAM1 side stays at 2, unchanged.
- Added the extra `gpio@2200000` "cam0-rst" GPIO hog that IMX296 needs (not present in the imx477-only dual overlay).
- `reset-gpios` for the new CAM0 sensor node kept at GPIO 0x3e ("cam0-pwdn") — same physical line the old imx477_a node used, confirming CAM0 identity didn't change.
- `tegra-camera-platform/modules/module0` sysfs path updated to point at the imx296 node instead of imx477_a.

Rewritten as a clean labeled DTS source (not hand-patched decompiled output) so `dtc -@` regenerates `__symbols__`/`__fixups__`/`__local_fixups__` correctly instead of manually maintaining raw phandle tables. Compiled cleanly (exit 0, only the same cosmetic warnings both original overlays already produce — no undefined-label or phandle errors). Decompiling the result back confirmed the fixup tables match the exact pattern of both original working overlays (`gpio_aon`/`cam_i2c`/`gpio` base-tree fixups, all internal endpoint cross-references resolved locally).

**Installed**:
- `/boot/tegra234-p3767-camera-p3768-imx296-cam0-imx477-cam1-custom.dtbo`
- `extlinux.conf`'s `JetsonIO` label updated: `FDT` set back to Arducam's base DTB (`/boot/arducam/dts/dtb/tegra234-p3768-0000+p3767-0005-nv-super.dtb` — the one CAM1/IMX477 has been proven against across all 8 prior debugging sessions, chosen over jetson-io's stock DTB since the CAM1 side has far more test history), `OVERLAYS` set to the new combined dtbo.
- Source DTS kept at `/tmp/.../scratchpad/dts/combined.dts` (session scratchpad — not a durable location, copy elsewhere if this needs to survive long-term).

**Result: confirmed working (2026-07-18).** Rebooted (full power cycle) and ran the full verification battery:

- Boot log: both sensors bound cleanly — `imx296 10-001a` (CAM0, `Detected IMX296LQ (Color)`) → `/dev/video0`, `imx477 9-001a` (CAM1) → `/dev/video1`. **I2C bus numbers are back to the original DT-confirmed mapping** (CAM0=bus 10, CAM1=bus 9 — matching the GPIO-hog-label evidence from `camera-cam0-cam1-debug.md`, unlike the single-camera-only overlay tests earlier which renumbered due to only one i2c-mux channel being registered). No `-121`/no-ACK errors, no probe failures.
- Isolated `sensor-id=0` (IMX296) and `sensor-id=1` (IMX477): both clean, both reported correct sensor mode lists (1456×1088@60fps for IMX296; 4032×3040/3840×2160/1920×1080 for IMX477), zero dmesg errors.
- **Simultaneous dual-camera single pipeline** (`nvarguscamerasrc sensor-id=0 ... ! ... nvarguscamerasrc sensor-id=1 ...` in one `gst-launch-1.0` invocation): both producers connected, both streamed for the full duration, zero real errors — only the usual benign `CANCELLED`/"Argus Correctable Error Status" shutdown artifact on one branch.

The custom combined overlay works as designed. Full command/decompile walkthrough for how it was built is in `dtc-combined-overlay-howto.md`.

**Not yet validated**: a real multi-minute simultaneous recording (this test was ~3-10s synthetic captures, consistent with the "synthetic test proves basic function, not long-run stability" lesson learned throughout the CAM0/CAM1 debugging history). Worth a longer joint run before treating this as fully production-ready.

## To revert if the combined overlay fails
```
# Back to CAM1/IMX577 only (imx477-dual, no IMX296):
sudo cp /boot/extlinux/extlinux.conf.backup-20260718224738-imx477-dual-working /boot/extlinux/extlinux.conf
sudo reboot

# Back to CAM0/IMX296 only (confirmed working single-camera state):
sudo cp /boot/extlinux/extlinux.conf.backup-20260718230857-imx296-cam0-working /boot/extlinux/extlinux.conf
sudo reboot
```

## Related doc
See `camera-cam0-cam1-debug.md` for the full CAM0/CAM1 physical-port debugging history (device-tree-confirmed CAM0=i2c bus 10/`imx477 10-001a`, CAM1=i2c bus 9/`imx477 9-001a`; the Argus `sensor-id` enumeration-order caveat; the Inno-Maker IMX577 CAM1 validation in Session 8).
