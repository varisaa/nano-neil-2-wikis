# Incident: L4T upgrade removed Arducam CSI kernel, restore attempt broke boot

**Date:** 2026-08-15/16
**Machine:** ubuntu-neil (Jetson Orin Nano, SSD/NVMe root)

## Timeline

1. **2026-08-15 22:38** — An `apt` upgrade (initiated by the user for unrelated
   packages) pulled in `nvidia-l4t-kernel 5.15.199-tegra-36.5.2-20260716114719`,
   which **silently purged** `arducam-nvidia-l4t-kernel
   5.15.185-tegra-36.5.0-20260207170705` (a hard version-pinned dependency
   conflict) and overwrote `/boot/Image` with the stock kernel. This also
   appears to have updated the Jetson's UEFI/bootloader firmware to the
   36.5.2 BSP level, not just the kernel — see "What actually went wrong"
   below.

2. Investigating a pre-existing USB Global Shutter Camera YUYV capture
   failure (`uvcvideo ... UVC non compliance: max payload transmission size
   (49152) exceeds ep max packet (1024)`, camera stuck behind a USB hub),
   we initially suspected the kernel downgrade/upgrade as the cause and
   began restoring the Arducam kernel. **This premise was later found to be
   wrong**: the Arducam package only ships IMX477/IMX296 CSI camera support
   (device-tree overlays, `nv_imx477.ko`, `tegra-camera.ko`) — nothing
   touching `uvcvideo.ko` or the USB/xHCI stack. The USB YUYV issue was
   never actually diagnosed further after this was discovered, and remains
   **unresolved** — MJPG mode on the Global Shutter Camera continues to
   work as the fallback.

3. Decided to restore the Arducam kernel anyway, since the user separately
   wanted working CSI camera support back regardless (a dual-camera setup:
   Global Shutter/IMX296 on CSI Cam0, IMX477 on CSI Cam1, via a merged
   custom device-tree overlay that had been working as of 2026-08-01).

4. The original `.deb`
   (`arducam-nvidia-l4t-kernel-t234-nx-5.15.185-tegra-36.5.0-20260207170705_arm64_imx477.deb`)
   was found preserved at `/home/neil/work/arducam/`. Its `postinst` script
   was found to be stale/buggy for this system (hardcoded a nonexistent
   `/lib/modules/5.15.148-tegra/` path for one module, would have placed
   the other modules into whichever kernel happened to be running at
   install time rather than the target). Decision was made to unpack the
   package (`sudo dpkg -i`, which unpacked payload but correctly refused to
   run postinst due to the unmet dependency) and complete the restore by
   hand instead of forcing the buggy script to run.

5. Manual steps taken: placed `nv_imx477.ko`/`tegra-camera.ko` into
   `/lib/modules/5.15.185-tegra/...`, ran `depmod -a 5.15.185-tegra`,
   copied `/boot/arducam/Image` over `/boot/Image`, copied
   `/boot/initrd.img-5.15.185-tegra` over `/boot/initrd`, and — based on
   finding an old working config in
   `/boot/extlinux/extlinux.conf.backup-20260801145550-pre-console-res-fix`
   — restored a `LABEL JetsonIO` entry (dual-CSI overlay, `LINUX
   /boot/arducam/Image`, custom `FDT`, `OVERLAYS` for the merged IMX296+IMX477
   overlay) and set `DEFAULT JetsonIO`.

6. Backups taken before any of this (correctly, and this is what saved the
   system): `/boot/Image.stock-36.5.2-20260716114719.backup` and
   `/boot/initrd.stock-36.5.2-20260716114719.backup` (copies of the stock
   files *before* they were overwritten).

7. **Rebooted. The system failed to come up at all.** User had to recover
   using an external SD card (Jetson rescue/flash path), mount the SSD
   rootfs, and manually copy the two `.backup` files above back over
   `/boot/Image` and `/boot/initrd`. Even after that, on the next boot the
   user had to explicitly select **boot option 0 ("Default kernel")** at
   the Jetson's own boot menu to get a successful boot — the extlinux
   `primary`/`JetsonIO` menu alone wasn't sufficient/reachable in the state
   the system was in.

## What actually went wrong (root cause)

**The core mistake: Step 3 of the restore copied the untested Arducam
kernel over `/boot/Image` — the same file the `primary` fallback label
pointed at — while simultaneously setting the newly-added `JetsonIO` label
(pointing at the same kernel via a different path) as `DEFAULT`.** Both
boot menu entries ended up pointing at the same unverified 5.15.185 kernel
build. The `primary` label's entire purpose is to be a safe fallback if a
custom kernel doesn't boot; overwriting it with the same custom kernel
defeated that safety net entirely. There was no working entry left in the
extlinux menu itself.

The likely deeper technical cause of the boot failure: the 2026-08-15
22:38 upgrade almost certainly updated Jetson's UEFI/bootloader firmware
to the 36.5.2 BSP level as part of the same transaction (this is normal
L4T upgrade behavior — kernel and bootloader are versioned together). The
restored kernel/initrd/FDT combo was built for the older 36.5.0 BSP. Pairing
a newer bootloader with an older kernel+FDT is a known-risky combination on
Jetson and most likely caused the hang/failure, independent of anything
being wrong with the module-placement or extlinux syntax itself (both of
which were verified correct before reboot).

**Lesson for next time:** when restoring/testing an alternate kernel on
this class of device, never overwrite the fallback boot path (`/boot/Image`
under the `primary` label) with the same untested kernel as the new
default. Keep the fallback pointing at a file that is verified-bootable
under the *current* bootloader/firmware version, add the new kernel only
as a non-default menu entry with its own distinct filename, and test-boot
it manually (selecting it once from the menu) before ever changing
`DEFAULT`. A file-level backup is necessary but not sufficient — what
matters is that at least one boot menu entry remains genuinely bootable at
all times.

## Current state (as of this writing, 2026-08-16 ~04:22)

- System is up and stable, booted on stock `5.15.199-tegra`.
- `/boot/Image` and `/boot/initrd` are confirmed restored to stock (md5
  verified against the pre-incident backups).
- `/boot/extlinux/extlinux.conf`: `DEFAULT` reset back to `primary`. The
  `LABEL JetsonIO` block is still present pointing at `/boot/arducam/Image`
  (**still broken/untested against the current 36.5.2 bootloader** — left
  in place only as a reference, not to be selected).
- `nvidia-l4t-kernel` is on `apt-mark hold` to prevent a silent repeat.
- `arducam-nvidia-l4t-kernel` remains in dpkg state `iU`
  (unpacked/unconfigured) — package payload is on disk but not "installed".
- The user's own emergency backups from the recovery,
  `/boot/Image.broken-2026-08-16` and `/boot/initrd.broken-2026-08-16`,
  preserve the exact kernel/initrd pair that failed to boot, if this is
  revisited later.
- The original USB Global Shutter Camera YUYV/`uvcvideo` endpoint issue
  that started this whole investigation is **still unresolved**. MJPG mode
  works; YUYV does not, regardless of kernel.
- Dual-CSI camera support (Global Shutter/IMX296 + IMX477) is **not
  restored** — this incident is the reason that work is on hold.

## If this is revisited

Any future attempt to restore the Arducam CSI kernel should:
1. Confirm what bootloader/BSP version is actually active
   (`sudo dpkg -l | grep nvidia-l4t-bootloader` or equivalent) and whether
   it's compatible with a 36.5.0-era kernel/FDT before doing anything else.
2. Install the new kernel to a distinct filename (e.g. `/boot/Image-arducam`)
   never overwriting `/boot/Image` itself, and add it as a non-default
   extlinux label.
3. Manually select and test-boot that label once from the boot menu with
   `DEFAULT` still pointing at `primary`, and only flip `DEFAULT` after a
   confirmed successful boot.
4. Keep an SD card recovery image readily accessible before starting,
   given this device's recovery path already required one.
