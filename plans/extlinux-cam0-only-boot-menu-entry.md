# Plan: add a 3-entry extlinux boot menu (primary / dual-camera / cam0-only)

**Status: planned, not yet implemented.** No changes have been made to `/boot/extlinux/extlinux.conf` for this yet.

## Context

The Nano (`nano-neil-2`) currently has two extlinux boot entries: `primary` (stock kernel, no camera overlay) and `JetsonIO` (the combined dual-camera overlay for cam0/IMX296 + cam1/IMX477, at `/boot/arducam/dts/tegra234-p3767-camera-p3768-imx296-cam0-imx477-cam1-custom.dtbo`). A debugging session found that booting `JetsonIO` with only cam0 physically connected hangs hard (unreachable, even over SSH) — reproduced twice, on both a warm reboot and a genuine power cycle — while cam1-alone and both-together (power cycle) both boot clean. Decompiling the combined overlay showed why: cam0 (`sony,imx296`) and cam1 (`ridgerun,imx477`) sit behind one shared physical I2C bus via a GPIO mux (`cam_i2cmux`), and both sensor nodes are always declared in this overlay regardless of what's physically attached. Working theory: the `ridgerun,imx477` driver doesn't fail as gracefully as imx296's does when its sensor is absent, and that stall — on a bus shared with cam0 — is what cascades into the full hang.

There's already project history that independently confirms the fix. Per `imx296-cam0-setup.md` in this same directory, on 2026-07-18 a **separate, single-camera overlay** — `tegra234-p3767-camera-p3768-imx296-cam0.dtbo` (already present on disk at `/boot/arducam/dts/`) — was tested with cam0 alone via a full power cycle and bound perfectly cleanly (`Detected IMX296LQ (Color)`, bound, verified capture, zero `-121`/no-ACK errors). That overlay never declares an IMX477 node at all (confirmed again this session by decompiling it), so it structurally can't hit the same stall. So the fix below isn't a new guess — it's re-exposing an already-proven-working configuration as its own boot menu entry, for convenience, instead of needing to re-run `jetson-io.py` each time to switch between the dual and cam0-only profiles.

(Separately, `camera-cam0-cam1-debug.md` documents a long history of **CAM1's physical port** being the intermittently flaky one — marginal/loose FFC seating causing I2C timeouts, going back to when both ports ran IMX477. That's a different, older issue from the boot-hang investigated here, but worth keeping in mind if cam1 acts up again during normal use — try reseating that cable first.)

The goal is all three boot options exposed directly in the extlinux menu (rather than needing to re-run `jetson-io.py` each time to switch profiles), ordered as:
- 0 — `primary`
- 1 — dual camera (cam0 IMX296 + cam1 IMX477)
- 2 — cam0 only (IMX296)

The `DEFAULT` (auto-booted after the 30s `TIMEOUT` if nothing is pressed) stays as the dual-camera entry, unchanged from today's config — confirmed user preference.

## Current file

`/boot/extlinux/extlinux.conf` (already backed up during the debugging session to `/boot/extlinux/extlinux.conf.before-cam0-menu-20260825041551` — reuse that backup, no need to make a new one when this is implemented):

```
TIMEOUT 300
DEFAULT JetsonIO

MENU TITLE L4T boot options

LABEL primary
      MENU LABEL primary kernel
      LINUX /boot/Image
      INITRD /boot/initrd
      APPEND ${cbootargs} root=PARTUUID=86dafc51-a53d-4466-afb3-6852dbe7c702 rw rootwait rootfstype=ext4 mminit_loglevel=4 console=ttyTCU0,115200 firmware_class.path=/etc/firmware fbcon=map:0 video=efifb:off console=tty0 efi=runtime pci=pcie_bus_perf nvme.use_threaded_interrupts=1

... (commented-out backup-kernel example, unchanged) ...

LABEL JetsonIO
	MENU LABEL Custom Header Config: <CSI Camera IMX296-C Cam0 and IMX477-C Cam1 (custom combined)>
	LINUX /boot/Image
	FDT /boot/arducam/dts/dtb/tegra234-p3768-0000+p3767-0005-nv-super.dtb
	INITRD /boot/initrd
	APPEND ${cbootargs} root=PARTUUID=86dafc51-a53d-4466-afb3-6852dbe7c702 rw rootwait rootfstype=ext4 mminit_loglevel=4 console=ttyTCU0,115200 firmware_class.path=/etc/firmware fbcon=map:0 video=efifb:off console=tty0 efi=runtime pci=pcie_bus_perf nvme.use_threaded_interrupts=1
	OVERLAYS /boot/arducam/dts/tegra234-p3767-camera-p3768-imx296-cam0-imx477-cam1-custom.dtbo
```

## Changes to make (when this is implemented)

1. **Leave `DEFAULT JetsonIO` unchanged** at the top of the file — dual-camera stays the default boot.

2. **Leave the existing `LABEL primary` entry untouched** (still entry 0, first in the file — extlinux lists/boots in file order).

3. **Keep the existing `LABEL JetsonIO` entry as entry 1** (dual camera), untouched except optionally clarifying its `MENU LABEL` text (e.g. `Dual Camera: IMX296 Cam0 + IMX477 Cam1`) for consistency with the new entry below — cosmetic only, not required.

4. **Append a new third entry, entry 2**, modeled directly on the `JetsonIO` block but pointing at the cam0-only overlay:
   ```
   LABEL JetsonIO-cam0
   	MENU LABEL Custom Header Config: <CSI Camera IMX296-C Cam0 only>
   	LINUX /boot/Image
   	FDT /boot/arducam/dts/dtb/tegra234-p3768-0000+p3767-0005-nv-super.dtb
   	INITRD /boot/initrd
   	APPEND ${cbootargs} root=PARTUUID=86dafc51-a53d-4466-afb3-6852dbe7c702 rw rootwait rootfstype=ext4 mminit_loglevel=4 console=ttyTCU0,115200 firmware_class.path=/etc/firmware fbcon=map:0 video=efifb:off console=tty0 efi=runtime pci=pcie_bus_perf nvme.use_threaded_interrupts=1
   	OVERLAYS /boot/arducam/dts/tegra234-p3767-camera-p3768-imx296-cam0.dtbo
   ```
   Same `FDT` base (`nv-super.dtb`) and same `APPEND` cbootargs as the dual entry — only the `LABEL`, `MENU LABEL`, and `OVERLAYS` line differ (pointing at the cam0-only `.dtbo` confirmed present at `/boot/arducam/dts/tegra234-p3767-camera-p3768-imx296-cam0.dtbo`).

All edits go through `sudo` (passwordless sudo already confirmed working) via `Edit`/`tee`, since `/boot/extlinux/extlinux.conf` is root-owned.

## Verification (when this is implemented)

1. `sudo cat /boot/extlinux/extlinux.conf` — confirm `DEFAULT JetsonIO` (unchanged) and all three `LABEL` blocks are present, well-formed, and in the right order (primary, JetsonIO, JetsonIO-cam0).
2. No reboot as part of the edit itself — actually booting each new/changed entry to confirm behavior is a separate, explicit follow-up step, since each carries the usual reboot-risk caveats (e.g. the new cam0-only entry should be tested with only cam0 connected, to confirm it boots clean where the dual overlay hung).
