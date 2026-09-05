# Handover — Dual-CSI Camera Stack on `ubuntu-neil`

**Prepared:** 2026-08-23 **For:** a local Claude Code session running on the board itself **Status:** both cameras verified working. Everything below is observed output, not assumption.

---

## 1\. What was achieved

The dual-CSI camera stack on `ubuntu-neil` is **working end-to-end for the first time since the August incident.**

| Milestone | Evidence |
| :---- | :---- |
| IMX296 streaming | 100 frames @ 1456×1088 RG10, locked **60.00 fps**, zero drops |
| IMX477 streaming | 60 frames @ 1920×1080 RG10 \= **248,832,000 bytes exactly** (4,147,200/frame) |
| IMX477 through Argus | 140,742-byte JPEG, `CONSUMER: Done Success`, no errors |
| **Both simultaneously** | **1200 frames, zero errors**, IMX296 at 60.00 fps and IMX477 at 30.00 fps held for the entire run |

The hand-authored combined overlay works. NVIDIA's stock overlays cannot do this — they claim the same NVCSI capture channel and can't be comma-listed.

Also completed:

- `nvidia-l4t-gstreamer` installed, **pinned to `36.5.0-20260115194252`**, and held. It was missing from the flash, which is why `nvarguscamerasrc` didn't exist.  
- `gstreamer1.0-plugins-bad` installed (stock Ubuntu jammy) for `h264parse`.  
- Claude Code 2.1.241 at `~/.local/bin/claude` (native installer, no Node, no system packages).  
- Firefox 154 aarch64 tarball at `~/firefox/` (no snap, no apt, no mesa).

### Two findings that overturned earlier conclusions

**The CAM0 failure was the wrong camera attached.** The signature — I2C ACK at `0x1a`, `invalid device model 0x0000`, constant `0x07` at every register address — looked exactly like dead core power or a missing input clock. It survived two cold boots and a careful ribbon reseat. It was simply not the IMX296 on that connector. With the correct sensor:

imx296 10-001a: Detected IMX296LQ (Color)

imx296 10-001a: found IMX296LQ (26.2C)

tegra-camrtc-capture-vi tegra-capture-vi: subdev imx296 10-001a bound

The connector, ribbon, clock path, carrier, overlay, and driver were all fine the whole time.

**The `PD_CRC_ERR` on the IMX477 was collateral.** CSI payload CRC errors were reported by the Argus daemon while the wrong (half-powered) sensor sat on CAM0 sharing the bus. Once CAM0 was correct: **zero CRC errors under twice the load.** No cable fault, no ribbon replacement needed.

---

## 2\. Verified camera configuration

### Board

|  |  |
| :---- | :---- |
| Hostname | `ubuntu-neil` (a.k.a. `nano-neil-2`), user `neil` |
| Hardware | Jetson Orin Nano Super, 931G NVMe |
| L4T | 36.5.0 / JetPack 6.2.2 |
| Kernel | `5.15.185-tegra` — **stock NVIDIA, not the Arducam build** |
| Root PARTUUID | `86dafc51-a53d-4466-afb3-6852dbe7c702` |
| apt holds | `nvidia-l4t-kernel`, `-bootloader`, `-initrd`, `-gstreamer` |

### Sensors

|  | CAM0 | CAM1 |
| :---- | :---- | :---- |
| Module | Inno-Maker CAM-IMX296Color-GS (global shutter, colour) | Inno-Maker IMX577, binds via `imx477` |
| I2C | bus **10**, addr `0x1a` | bus **9**, addr `0x1a` |
| Node (both connected) | `/dev/video0`, `tegra-capture-vi:1` | `/dev/video1`, `tegra-capture-vi:2` |
| Driver | `imx296_v2.0.6` | `imx477_v2.0.6` |

Files are labelled "Arducam" throughout due to shared driver lineage. The hardware is Inno-Maker.

**Node numbering is not stable.** It follows bind order. In earlier sessions the IMX477 was `/dev/video0`. Always confirm with `v4l2-ctl --list-devices` before using a node number from these notes.

### Mode tables

**IMX296** — one mode only:

- 1456×1088 @ 60 fps, RG10

**IMX477** — Argus reports three, V4L2 enumerates two sizes:

| Argus | V4L2 `sensor_mode` |
| :---- | :---- |
| 4032×3040 @ 21 fps | 0 → 3840×2160 |
| 3840×2160 @ 30 fps | 1 → 1920×1080 |
| 1920×1080 @ 60 fps | 2 |

The Argus and V4L2 mode tables **do not correspond one-to-one.** `sensor_modes` reports 3; `--list-formats-ext` shows only two sizes. Mode 1 was confirmed as 1080p by exact byte count; its framerate was not independently confirmed.

### Boot configuration

LABEL primary

      LINUX /boot/Image        \# stock, no FDT, no overlays — bootable fallback

LABEL JetsonIO

      MENU LABEL Custom Header Config: \<CSI Camera IMX296-C Cam0 and IMX477-C Cam1 (custom combined)\>

      LINUX /boot/Image

      FDT /boot/arducam/dts/dtb/tegra234-p3768-0000+p3767-0005-nv-super.dtb

      OVERLAYS /boot/arducam/dts/tegra234-p3767-camera-p3768-imx296-cam0-imx477-cam1-custom.dtbo

`primary` is a genuinely bootable fallback. Its absence is what caused the August unbootable state. **It must stay that way.**

---

## 3\. How to test and run

### Health check after any boot

sudo dmesg | grep \-iE 'imx296|imx477'

v4l2-ctl \--list-devices

Want to see `Detected IMX296LQ (Color)` plus `found IMX296LQ (NN.NC)` — that temperature is a real die reading and a good health signal — and `subdev ... bound` for both. `-121` (EREMOTEIO) means nothing is answering on that bus: check the ribbon is seated **and that the right camera is attached.**

### Raw V4L2 capture

Bypasses Argus and the ISP. Use it to prove the sensor → CSI → VI path.

\# IMX296 — 100 frames

sudo v4l2-ctl \-d /dev/video0 \\

  \--set-fmt-video=width=1456,height=1088,pixelformat=RG10 \\

  \--stream-mmap \--stream-count=100 \--stream-to=/dev/null

\# IMX477 — 60 frames to a file, byte count is the check

sudo v4l2-ctl \-d /dev/video1 \--set-ctrl sensor\_mode=1 \\

  \--set-fmt-video=width=1920,height=1080,pixelformat=RG10 \\

  \--stream-mmap \--stream-count=60 \--stream-to=/tmp/imx477.raw

ls \-l /tmp/imx477.raw     \# expect exactly 248832000

Frame size is width × height × 2 for RG10. An unexpected file size means the mode isn't what you asked for.

**Never Ctrl-C a running capture.** It trips a `videobuf2` teardown bug (`stop_streaming operation is leaving buf ... in active state`) that wedges the capture path until reboot. Always let `--stream-count` finish.

### Dual simultaneous capture

sudo dmesg \-C

sudo v4l2-ctl \-d /dev/video0 \--set-fmt-video=width=1456,height=1088,pixelformat=RG10 \\

  \--stream-mmap \--stream-count=600 \--stream-to=/dev/null &

sudo v4l2-ctl \-d /dev/video1 \--set-fmt-video=width=1920,height=1080,pixelformat=RG10 \\

  \--stream-mmap \--stream-count=600 \--stream-to=/dev/null &

wait

sudo dmesg | grep \-icE 'crc|error'

Pass condition: both hold nominal rate, error count is `0`. Output interleaves and looks messy — that's fine.

### Argus / GStreamer

\# single JPEG

gst-launch-1.0 nvarguscamerasrc sensor-id=0 num-buffers=1 \! \\

  'video/x-raw(memory:NVMM),width=1920,height=1080,framerate=60/1' \! \\

  nvjpegenc \! filesink location=/tmp/cam.jpg

**Confirm which camera you got by reading the mode table Argus prints.** A single 1456×1088 entry is the IMX296; the three-mode 4032×3040 table is the IMX477. Argus `sensor-id` ordering is independent of `/dev/videoN` and depends on which cameras are attached.

### Live preview (display attached)

DISPLAY=:0 gst-launch-1.0 nvarguscamerasrc sensor-id=0 \! \\

  'video/x-raw(memory:NVMM),width=1920,height=1080,framerate=60/1' \! \\

  nv3dsink sync=false

`nv3dsink` takes NVMM buffers directly — lowest latency, best for focusing. `nvvidconv ! xvimagesink` is the fallback. `nvoverlaysink` does not exist on 36.5.

For focus judgement, insert `videoconvert ! edgetv ! videoconvert` before the sink and adjust for maximum edge density.

---

## 4\. Debugging notes

### `INVALID_SETTINGS` is a misleading error

`nvarguscamerasrc` reports `INVALID_SETTINGS` / `CANCELLED` / "Argus Correctable Error Status" for almost everything. **The real error is in the daemon log:**

sudo journalctl \-u nvargus-daemon \-f

This is what revealed `PD_CRC_ERR: CRC error on packet payload` — a physical-layer fault completely invisible from the client side. Always check here before theorising.

Restart the daemon between failed Argus attempts; a failed pipeline leaves it holding sensor state:

sudo systemctl restart nvargus-daemon

Ignore `NvPclHwGetModuleList: No module data found` and `ImagerGUID 1` errors when only one camera is attached — that's Argus failing to open the absent sensor.

### Raw V4L2 does not validate CRC

V4L2 hands you whatever arrives. Argus checks CRC and aborts. **A clean 600-frame V4L2 capture does not prove the data is good** — it only proves buffers moved. Use Argus, or check the daemon log, to catch corruption.

### `sensor_mode` overrides `S_FMT`

The V4L2 `sensor_mode` control takes precedence over the requested format, and it is **sticky device state that persists across separate `v4l2-ctl` invocations and across reboots.** Symptom: you request 1920×1080 and get a 4K capture. Check it first whenever a resolution is unexpected:

sudo v4l2-ctl \-d /dev/videoN \--get-ctrl sensor\_mode

This also affects Argus, which does its own mode selection on top.

### Every `nvidia-l4t-*` install must be version-pinned

**This is the most important operational finding.** The L4T repo serves multiple point releases inside `r36.5`:

nvidia-l4t-gstreamer | 36.5.2-20260716114719 | ...r36.5/main   \<-- apt default

nvidia-l4t-gstreamer | 36.5.0-20260115194252 | ...r36.5/main   \<-- what you need

`apt install nvidia-l4t-anything` silently pulls **36.5.2**, which is the release train that caused the August failure. The camera `.deb` is version-pinned to 36.5.0 with no source available, so 36.5.0 is non-substitutable.

Always simulate first and read the plan:

sudo apt-get install \--no-install-recommends \-s \<pkg\>=36.5.0-20260115194252

Want `0 upgraded`. Anything appearing as an *upgrade* — especially `nvidia-l4t-*` — means stop. Note the holds only cover four packages; `nvidia-l4t-core`, `-3d-core`, and `-multimedia` are unprotected. `apt-mark hold` anything new you install.

Roughly 376–405 packages sit behind the holds. That number growing is the pinned-BSP state working as intended, not a backlog.

### Hardware limits

- **The Orin Nano has no NVENC.** `nvv4l2h264enc` / `nvv4l2h265enc` will never register — the elements probe for encode hardware and silently omit themselves. Software encode (`x264enc`) only, which will distort thermal and power figures in a soak test. Hardware encode requires the AGX.  
- `/dev/nvhost*` exposes GPU, ISP (`nvhost-ctrl-isp`), NVCSI (`nvhost-ctrl-nvcsi`), and both VI units (`vi0`, `vi1`). Full capture path, no media acceleration.

### Full power cycles

Warm reboots do not clear `nvmap`/`nvhost` buffer state or I2C mux driver registration. When camera behaviour is odd, power down fully and pull the barrel jack for \~10 seconds.

### Environment quirks

- `dmesg` requires `sudo` on this rootfs.  
- `curl` is **not** installed. Use `wget -qO- <url> | bash`.  
- `/tmp` has the sticky bit; files written by a root-run `v4l2-ctl` need `sudo rm`.  
- Password sudo times out at 15 min. `sudo -i` for a persistent root shell during long sessions.  
- `TIMEOUT` in `extlinux.conf` is tenths of a second. `TIMEOUT 300` \= 30 seconds.

---

## Hard rules

1. **Never overwrite `/boot/Image`.** Install any test kernel under a distinct filename, add it as a **non-default** extlinux label, test-boot from the menu, then change `DEFAULT`. At least one entry must remain genuinely bootable at all times.  
2. **No `apt upgrade` / `dist-upgrade`.** Install-only, version-pinned. The 2026-08-15 incident was `apt upgrade` pulling `nvidia-l4t-kernel 5.15.199-tegra-36.5.2` and orphaning the pinned camera kernel.  
3. **Full power cycles**, not warm reboots, when testing camera overlays.  
4. Use **Arducam's** jetson-io (`/opt/arducam/jetson-io/config-by-hardware.py`), never NVIDIA's — different overlay paths, base FDTs, and header-name stamps. Arducam's scans `/boot/arducam/dts/`, not `/boot/`.  
5. **36.5.0 is non-substitutable.** The IMX296 driver is a binary `.ko` for `5.15.185-tegra` with no source.  
6. **Check the camera is the one you think it is** before diagnosing a hardware fault. A whole evening went to this.

---

## Recovery artifacts

| Item | Location |
| :---- | :---- |
| Camera archive (verified superset of pre-flash NVMe) | AGX: `jetson-images/nano-neil-2-camera-config/nano-neil-2-camera-config-2026-08-17.tar.zst` |
| Archive SHA256 | `5a9bf53702f6ebeddced928f1dad3bf2f9a2cbebf5256b1afd3aca5da0a5582a` |
| Extracted on board | `/home/neil/restore/` |
| Prepared 36.5.0 flash tree | T7 SSD (`l4twork`), `/mnt/work/Linux_for_Tegra` — keep, \~1hr to rebuild |
| SD recovery card | JP6.2.1 / R36.4.4, proven bootable 2026-08-23 |
| Stock VI module backup | `/boot/tegra-camera.ko.stock-36.5.0` |
| imx296 installer rollback | `/opt/imx296-unified-backup-20260823075507` |

**AGX** (`jacob-ubuntu`, `192.168.1.78`) is the archive host — 4TB at `/mnt/a10c41a4-9100-48a3-8949-85f41b12b65f/`. It is aarch64 and **cannot flash Jetsons**; only the Windows PC (`DESKTOP-5FM61Q2`, x86) can.

---

## Open items

- **Multi-minute dual soak — never validated.** All testing to date is ≤1200 frames. Thermal throttling, NVMM buffer exhaustion, and sustained-load signal integrity are unexercised. Run in tmux with `jtop` watching; note there is no hardware encoder, so record raw or MJPEG rather than H.264.  
- Lens focus on both cameras. The IMX296 is a machine-vision module and may be fixed-focus — check before setting up a preview for it.  
- Argus verified on the IMX477 only. The IMX296 has not been through the ISP path.  
- Second copy of the camera archive (1.3 GB, irreplaceable) — currently only on the AGX and this board. The T7 is the obvious third home.  
- Node-order stability across reboots — observed to differ between sessions, never characterised.

