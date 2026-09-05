# CAM0/CAM1 debugging — findings (2026-07-18)

## Board
- NVIDIA Jetson Orin Nano Engineering Reference Developer Kit Super (`p3768-0000+p3767-0005-super`)
- L4T R36.5.0, kernel `5.15.185-tegra`
- Dual IMX477 (Arducam) via custom overlay, set in `/boot/extlinux/extlinux.conf`:
  - `FDT /boot/arducam/dts/dtb/tegra234-p3768-0000+p3767-0005-nv-super.dtb`
  - `OVERLAYS /boot/arducam/dts/tegra234-p3767-camera-p3768-imx477-dual.dtbo`
- Related repo: `/home/neil/work/arducam` (has the kernel module .deb, `install_full.sh`, `imx477_links.txt`)

## Symptom reported
- `cam0` (sensor-id=0) used to work, now fails.
- `cam1` (sensor-id=1) has never worked.
- Test command used:
  ```bash
  SENSOR_ID=0   # 0 for CAM0, 1 for CAM1
  FRAMERATE=30
  gst-launch-1.0 -e nvarguscamerasrc sensor-id=$SENSOR_ID ! \
    "video/x-raw(memory:NVMM),width=1920,height=1080,framerate=$FRAMERATE/1" ! \
    nvv4l2h264enc ! h264parse ! mp4mux ! filesink location=rpi_v3_imx477_cam$SENSOR_ID.mp4
  ```

## Device topology (confirmed via `v4l2-ctl --list-devices`, `media-ctl -p`)
- Both sensors sit on i2c address `0x1a`, muxed off ONE physical controller:
  - `tegra-i2c 3180000.i2c` (= `i2c-2`) → `i2c-mux-gpio bus@0:cam_i2cmux` → `i2c-9` (mux chan 1) / `i2c-10` (mux chan 0)
- At boot (when both probe successfully), both sensors bind cleanly (`imx477_v2.0.6` driver), no errors:
  ```
  imx477 9-001a: tegracam sensor driver:imx477_v2.0.6
  tegra-camrtc-capture-vi tegra-capture-vi: subdev imx477 9-001a bound
  imx477 10-001a: tegracam sensor driver:imx477_v2.0.6
  tegra-camrtc-capture-vi tegra-capture-vi: subdev imx477 10-001a bound
  ```

### Physical CAM0/CAM1 ↔ i2c bus mapping — CONFIRMED from the device tree overlay
Decompiled the overlay actually in use with `dtc`:
```bash
dtc -I dtb -O dts /boot/arducam/dts/tegra234-p3767-camera-p3768-imx477-dual.dtbo
```
The power-down GPIO hog labels each pin explicitly:
```
gpio@6000d000 {
    camera-control-output-low {
        gpios = <0xa0 0x00  0x3e 0x00>;
        label = "cam1-pwdn\0cam0-pwdn";   // GPIO 160="cam1-pwdn", GPIO 62="cam0-pwdn"
    };
};
```
Cross-referenced against each sensor node's own `reset-gpios`:
```
i2c@0 (mux channel 0, i2c bus 10) → rbpcv3_imx477_a@1a → reset-gpios = ...0x3e...  → GPIO 62  = "cam0-pwdn"
i2c@1 (mux channel 1, i2c bus  9) → rbpcv3_imx477_c@1a → reset-gpios = ...0xa0...  → GPIO 160 = "cam1-pwdn"
```
**Result — this is ground truth, not inferred:**
- **CAM0** (physical connector silkscreened "CAM0") = i2c mux channel 0 = i2c bus **10** = kernel device `imx477 10-001a`
- **CAM1** (physical connector silkscreened "CAM1") = i2c mux channel 1 = i2c bus **9** = kernel device `imx477 9-001a`

**Correction to earlier notes in this doc:** `imx477 9-001a` is **CAM1**, not CAM0 — an earlier note below said otherwise; that was wrong and is corrected in place.

**Important caveat — `sensor-id=N` is NOT a fixed physical mapping.** `nvarguscamerasrc sensor-id=0/1` is an Argus enumeration index assigned to whichever sensors currently bind successfully, in bind order — it is not hardwired to CAM0=0 / CAM1=1. When only one sensor is bound (e.g. one port has a probe failure), that lone sensor is always index 0, regardless of which physical port it is. **To know which physical port you're actually driving, check `v4l2-ctl --list-devices` / dmesg for the `9-001a` (CAM1) vs `10-001a` (CAM0) identity — never trust the `sensor-id` number alone.**

## Test results (isolated, dmesg cleared before each run)

**CAM0 alone (`sensor-id=0`)** — clean:
- Negotiates sensor modes (4032x3040@21, 3840x2160@30, 1920x1080@60), streams 60 buffers, clean EOS.
- Zero dmesg errors.

**CAM1 alone (`sensor-id=1`)** — fails every time:
```
nvbuf_utils: dmabuf_fd -1 mapped entry NOT found
Error generated. .../gstnvarguscamerasrc.cpp, threadExecute:760 NvBufSurfaceFromFd Failed.
```
dmesg floods with:
```
tegra-i2c 3180000.i2c: I2C transfer timed out
imx477 9-001a: imx477_write_reg: i2c write failed, 0x104 = 1
imx477 9-001a: imx477_set_group_hold: Group hold control error
regmap_util_write_table_8:regmap_util_write_table:-110
imx477 9-001a: Error writing mode
```
Note: errors are attributed to `imx477 9-001a`, which (per the DT-confirmed mapping above) is **CAM1's own i2c client** — so this is CAM1 throwing the errors on itself, not cross-attribution to the other sensor as originally guessed.

**CAM0 retried immediately after a failed CAM1 attempt** — now also fails:
```
ERROR: ... nvarguscamerasrc0: UNAVAILABLE
Additional debug info: Argus Uncorrectable Error Status
nvbuf_utils: dmabuf_fd -1 mapped entry NOT found
```
i.e. CAM1's failure corrupts shared state (mux and/or nvargus-daemon) and takes CAM0 down with it.

**Fix that restores CAM0:**
```bash
sudo systemctl restart nvargus-daemon
```
After this, CAM0 immediately goes back to capturing cleanly. This confirms CAM0's hardware/sensor is fine — it's collateral damage from CAM1 failing, not independently broken.

## Conclusion / working theory
- CAM0 (`10-001a`, i2c bus 10) hardware and driver path: **known good** — clean across every test in every session so far.
- CAM1 (`9-001a`, i2c bus 9): **the real, recurring fault**, reproduced across three separate sessions/boots with varying severity:
  - Session 1 (first boot tested): probed fine at boot, worked once in isolation, then failed with I2C timeouts under a second attempt, corrupting shared state and taking CAM0 down with it (fixed by `nvargus-daemon` restart).
  - Session 2 (after a reboot, before any cable changes): CAM1 failed to even ACK at boot probe (`error during i2c read probe (-121)`) — driver never bound, sensor didn't show up at all.
  - Session 3 (after reboot + reseating both ribbons): CAM1 probed fine at boot AND passed one isolated capture cleanly — but immediately failed again on the very next attempt with the same `NvBufSurfaceFromFd` / I2C-timeout signature, and kept failing on every subsequent attempt in a back-to-back stress test, while CAM0 continued working fine throughout.
- This pattern — sometimes probing, sometimes not; working once then failing; always the same I2C-timeout signature when it fails — is the classic signature of a **marginal/intermittent physical connection specific to the CAM1 port** (loose seating, slightly misaligned or partially-engaged locking tab, a marred contact, or a flexed/creased ribbon), not dead silicon and not a kernel/software bug. Reseating alone has not resolved it.
- Whenever CAM0 appears to stop working, it may just be inheriting CAM1's failure state (shared i2c mux/nvargus-daemon) — restart `nvargus-daemon` before concluding CAM0 itself is broken.

## Also noted (unrelated to the port fault)
- `nvv4l2h264enc` does not exist on this board. Jetson Orin Nano has **no hardware video encoder** (NVENC removed vs AGX Orin). `gst-inspect-1.0` on `libgstnvvideo4linux2.so` shows only `nvv4l2decoder` registered. Any recording pipeline needs a software encoder (e.g. `x264enc`) instead of `nvv4l2h264enc`.

## Session 3 (2026-07-18, after reboot + reseating both ribbons)
User rebooted and reseated both CAM0 and CAM1 ribbon cables, then asked to re-test.

- Boot log: both sensors probed and bound cleanly this time (`imx477 9-001a` and `imx477 10-001a` both bound, no errors) — better than session 2, where CAM1 didn't ACK at all.
- Isolated test, `sensor-id=0` (CAM0): clean, zero dmesg errors.
- Isolated test, `sensor-id=1` (CAM1): **also clean this one time** — zero dmesg errors, full clean capture.
- `i2cdetect` on both bus 9 and bus 10: both show `UU` at `0x1a` (driver bound on both).
- Rapid back-to-back stress test (`1, 0, 1, 0`): CAM1 (`sensor-id=1`) failed on **every** run after the first isolated one:
  ```
  Error generated. .../gstnvarguscamerasrc.cpp, threadExecute:760 NvBufSurfaceFromFd Failed.
  ```
  dmesg during/after:
  ```
  tegra-i2c 3180000.i2c: I2C transfer timed out
  imx477 9-001a: imx477_write_reg: i2c write failed, 0x104 = 1
  imx477 9-001a: imx477_set_group_hold: Group hold control error
  regmap_util_write_table_8:regmap_util_write_table:-110
  imx477 9-001a: Error writing mode
  ```
  CAM0 (`sensor-id=0`) kept working cleanly in every interleaved run in this test, unlike session 1 where it got knocked out by CAM1's failure.

**Takeaway: reseating did not fix CAM1.** It's now slightly more consistent (probes at boot, can even capture once) but still fails reliably under repeated/sustained use with the identical I2C-timeout signature seen in session 1. This reinforces the marginal-connection theory over a dead-sensor theory — a fully dead sensor wouldn't intermittently succeed.

## Session 4 (2026-07-18, after another reboot + reseat) — CAM1 held up this time
User rebooted and reseated both ribbons again, asked to re-test.

- Boot log: both sensors bound cleanly again, same as session 3.
- Isolated single tests: both `sensor-id=0` and `sensor-id=1` clean, zero dmesg errors.
- Alternating stress test (`1,0,1,0,1,0` — 6 runs): **all 6 clean**, zero dmesg errors. (Session 3's identical test had CAM1 fail on every run after the first.)
- Sustained CAM1 run, 300 buffers (~11s wall time): clean, zero dmesg errors.
- Bigger alternating stress test (10 rounds, `1,0` × 5): **all 10 clean**, zero dmesg errors.
- Total across this session: 16 pipeline invocations (8× CAM1, 8× CAM0), no I2C timeouts, no `NvBufSurfaceFromFd` errors, no `UNAVAILABLE`/`Invalid camera device` — only the benign `CANCELLED`/`Argus Correctable Error Status` cleanup message that appears on clean runs of both sensors.

**This is the best result across all sessions.** The CAM1 fault has previously been intermittent/connection-based (see session 3's contrast — same test, same cable, failed almost every time), so treat this as "meaningfully improved, not yet proven permanently fixed" rather than a hard guarantee: keep an eye out for recurrence with real workloads (longer recordings, dual-camera-simultaneous use, after any bump/vibration to the board). If it stays clean over normal day-to-day use, the reseat was the fix and the earlier failures were exactly the marginal-connection issue theorized in sessions 1–3.

## Session 5 (2026-07-18, another reboot + reseat) — clean again
User rebooted and reseated both ribbons again, asked to re-test.

- Boot log: both sensors bound cleanly, same as sessions 3–4.
- 10-round alternating stress test (`1,0` × 5, 20 buffers each): all runs completed, zero dmesg errors — no `NvBufSurfaceFromFd`, no I2C timeouts, no `UNAVAILABLE`/`Invalid camera device`.
- Sustained CAM1 run, 300 buffers (~11s): clean, zero dmesg errors.

**Two consecutive reseat sessions (4 and 5) are now fully clean** under the same stress test that reliably broke CAM1 in session 3. This is a meaningfully stronger signal that the physical reseat is the actual fix, not a fluke — but given the fault's history of intermittently working before failing again, still worth validating with a real multi-minute recording / dual-camera-simultaneous workload before fully closing this out.

## Session 6 (2026-07-18, another reboot + reseat) — clean, third in a row
User rebooted and reseated both ribbons again, asked to re-test.

- Boot log: both sensors bound cleanly (`imx477 10-001a` = CAM0 → `/dev/video1`, `imx477 9-001a` = CAM1 → `/dev/video0`), consistent with the confirmed mapping.
- 10-round alternating stress test (`1,0` × 5, 20 buffers each): all runs completed, only the benign `CANCELLED`/"Argus Correctable Error Status" shutdown artifact — zero dmesg errors, no I2C timeouts.
- Sustained CAM1 run, 300 buffers: completed normally via EOS in 11.1s (full 30fps rate), zero dmesg errors. (First invocation of this test raced against its own `timeout` wrapper and printed only "Terminated" with no captured stdout — a harness artifact, not a pipeline fault; re-run without the race showed a fully clean completion.)

**Three consecutive reseat sessions (4, 5, 6) are now fully clean.** The pattern is holding up well under the synthetic stress test. Still recommend validating with a real multi-minute recording and/or simultaneous dual-camera workload before calling this permanently resolved — the fault's history includes working cleanly for a while before regressing, so continued casual monitoring is worthwhile even if this is the fix.

## Session 7 (2026-07-18, another reboot + reseat) — clean, fourth in a row
User rebooted and reseated both ribbons again, asked to re-test.

- Boot log: both sensors bound cleanly (`imx477 10-001a` = CAM0 → `/dev/video1`, `imx477 9-001a` = CAM1 → `/dev/video0`). Boot log timestamps briefly jumped from "Jun 05" to "Dec 31 16:00:33" mid-sequence — RTC not yet synced this early in boot, a cosmetic artifact, not a device fault.
- 10-round alternating stress test (`1,0` × 5, 20 buffers each): all runs completed, only the benign `CANCELLED`/"Argus Correctable Error Status" shutdown artifact — zero dmesg errors.
- Sustained CAM1 run, 300 buffers: completed cleanly, zero dmesg errors.

**Four consecutive reseat sessions (4, 5, 6, 7) are now fully clean** under the same synthetic stress test that reliably broke CAM1 in session 3. At this point the synthetic test has stopped being a useful discriminator — it's not finding anything new. The recommended next validation step is a real-world workload (multi-minute recording, ideally both cameras simultaneously) rather than more repeats of this same stress test.

## Session 8 (2026-07-18) — hardware swap: Inno-Maker IMX577 module attached to CAM1, CAM0 empty
User physically swapped hardware: removed whatever was on CAM0 (left empty) and attached a **new Inno-Maker CAM-IMX577-12MP** module to CAM1, then rebooted.

**Initial confusion, corrected:** first pass at this looked like a driver/device-tree mismatch (CAM1 ACKed I2C fine but `nvarguscamerasrc sensor-id=1` failed with `NvBufSurfaceFromFd Failed`, no dmesg errors). That was chased down two dead ends before finding the real cause:
- **Not an Arducam module** — the earlier assumption that this needed Arducam's module-config tooling (`install_full.sh` / `modules.txt`) was wrong; this is an Inno-Maker part, unrelated packaging.
- **Not a missing driver/overlay** — researched Inno-Maker's CAM-IMX577-12MP product docs: it explicitly **reuses the existing IMX477 kernel driver under JetPack 6.2** (this board is L4T R36.5.0, i.e. within the JetPack 6.2 line) — no separate `imx577` driver or device-tree overlay exists or is needed. Confirmed no `imx577` references anywhere in `/boot`, `/lib/modules`, or the installed Arducam package; Jetson-IO itself (`/opt/nvidia/jetson-io/`) has no sensor-specific logic either — it only toggles which prebuilt `.dtbo` is loaded.
- **Real cause of the earlier confusing symptom**: with CAM0 completely empty, `imx477 10-001a` fails to probe at boot (`-121`, no ACK — expected, nothing attached), so **only one sensor binds at all**. Argus then enumerates just **one** camera, at index **0** — not index 1. The original test used `sensor-id=1`, which doesn't exist when only one device is bound (`Invalid camera device specified 1 specified, 0 max index`). This is the exact "sensor-id is enumeration order, not fixed hardware mapping" caveat already noted above — it bit us again here, in a new way (device count changed, not device order).

**Corrected test** — `sensor-id=0` (the only bound device, `imx477 9-001a` / CAM1 / the new IMX577 module):
```
GST_ARGUS: Running with following settings:
   Camera index = 0
   ...
CONSUMER: Producer has connected; continuing.
...
EOS received - stopping pipeline...
Execution ended after 0:00:01.727542967
```
Clean run, zero dmesg errors — only the benign `CANCELLED`/"Argus Correctable Error Status" shutdown artifact. **The Inno-Maker IMX577 module works on CAM1 using the existing IMX477 driver/overlay, exactly as Inno-Maker's docs describe — no device-tree changes needed.**

**Takeaway for future single-camera testing on this board:** whenever only one sensor is physically attached/bound, always probe with `sensor-id=0` first (or check `v4l2-ctl --list-devices` / dmesg to see what actually bound) rather than assuming the sensor keeps its usual `sensor-id=0`/`1` from dual-camera sessions.

## Next steps (need physical access to the board)
1. Re-check the CAM1 ribbon cable more carefully: locking tab fully and evenly engaged (not just closed on one side), cable not creased/twisted near the connector, contacts clean.
2. If a spare/known-good FFC cable is available, swap it onto CAM1 to rule out a damaged cable.
3. Swap the physical sensor module between the CAM0 and CAM1 ports:
   - If the failure follows the **port** (a known-good module still fails on CAM1) → suspect the board connector/mux itself.
   - If the failure follows the **module** → suspect that specific sensor unit.
3. Useful commands to re-run after reboot to pick this back up:
   ```bash
   # isolated test per port, watch dmesg live in another shell: sudo dmesg -w
   sudo dmesg -c > /dev/null
   timeout 8 gst-launch-1.0 -e nvarguscamerasrc sensor-id=0 num-buffers=60 ! \
     "video/x-raw(memory:NVMM),width=1920,height=1080,framerate=30/1" ! nvvidconv ! fakesink
   sudo dmesg

   sudo dmesg -c > /dev/null
   timeout 8 gst-launch-1.0 -e nvarguscamerasrc sensor-id=1 num-buffers=60 ! \
     "video/x-raw(memory:NVMM),width=1920,height=1080,framerate=30/1" ! nvvidconv ! fakesink
   sudo dmesg

   # if CAM0 misbehaves after a CAM1 attempt, try this before assuming CAM0 is broken:
   sudo systemctl restart nvargus-daemon
   ```

## CAM0/CAM1 connector + FFC cable "Type A / Type B" question
User has CAM0 on "Type A" cable and CAM1 on "Type B" cable — asked whether that's correct.

Could not confirm this pairing — no documentation found (NVIDIA or Arducam) that defines a
"Type A → CAM0, Type B → CAM1" assignment, and there's no way to verify it without physically
inspecting the board/cables.

What was established from public docs instead:
- **"Type A/Type B" most likely refers to FFC contact orientation** (contacts facing up vs. down),
  not a port assignment. It's about which way the ribbon needs to be flipped to seat correctly at
  each end (one end plugs into the Jetson connector, the other into the camera board — these often
  need opposite contact-facing), not which physical CSI port (CAM0 vs CAM1) it belongs to.
- **The authoritative source for port identity is the board silkscreen** — the Orin Nano devkit
  carrier board has "CAM0"/"CAM1" printed directly next to each connector. That's what determines
  `sensor-id=0` vs `sensor-id=1`, not the cable type.
- Per NVIDIA's hardware layout docs: CAM0 and CAM1 are identical 22-pin, 0.5mm-pitch, bottom-contact
  connectors. Contacts should face the heatsink side (silver pins down) on **both** connectors. The
  only functional difference: CAM0 is CSI 2-lane only, CAM1 supports 2-lane or 4-lane.
- So using two differently-labeled cables per port isn't inherently wrong, as long as each is seated
  silver-pins-down and fully locked at both ends. Given CAM1's failure is at the I2C level (not just
  CSI image data), cable seating/locking-tab engagement on CAM1 is a higher-priority check than the
  "type" label itself.

Open item: confirm what's actually printed on the two cables in hand, or check the board silkscreen
next to each connector, to settle the CAM0/CAM1 ↔ cable-type mapping. (The CAM0/CAM1 ↔ i2c-bus
mapping itself is now separately confirmed from the device tree — see the "Physical CAM0/CAM1 ↔ i2c
bus mapping" section above. The cable-type question is still unresolved and, given CAM1 keeps
failing intermittently even after a reseat, worth revisiting — e.g. try swapping in a different
physical cable on CAM1.)

Sources:
- [Hardware Layout — Jetson Orin Nano Developer Kit User Guide](https://docs.nvidia.com/jetson/orin-nano-devkit/user-guide/latest/hardware_layout.html)
- [IMX477 Arducam cable not fitting into Jetson Orin Nano CSI connector (CAM0) — NVIDIA Developer Forums](https://forums.developer.nvidia.com/t/imx477-arducam-cable-not-fitting-into-jetson-orin-nano-csi-connector-cam0/368971)
- [CSI Camera Connector Compatibility for Jetson Orin Nano](https://nvidia-jetson.piveral.com/jetson-orin-nano/csi-camera-connector-compatibility-for-jetson-orin-nano/)
