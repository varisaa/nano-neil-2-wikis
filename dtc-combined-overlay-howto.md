# How we built the combined IMX296+IMX477 device tree overlay (2026-07-18)

This is a from-scratch explanation of device tree overlays, the `dtc` compiler, and exactly what
we did to get CAM0 (IMX296) and CAM1 (IMX477) running from one boot configuration — written for
someone with no prior kernel/device-tree background. See `imx296-cam0-setup.md` for the
narrative debugging log; this doc is the "how it actually works and how to do it again" reference.

---

## 1. Background concepts

### What is a device tree?

On a PC, the OS discovers hardware at boot time (BIOS/UEFI, PCI enumeration, USB descriptors,
etc.) — nothing needs to be hard-coded about *which* graphics card or *which* USB controller is
plugged in. Embedded ARM boards like the Jetson mostly don't have that kind of self-describing
hardware bus for everything. Instead, the manufacturer ships a **device tree**: a data structure,
compiled into a binary blob, that tells the Linux kernel "here is every piece of hardware on this
board, here is how it's wired up (which I2C bus, which GPIO pins, which memory addresses, which
clocks), and here is which kernel driver should bind to each one."

The kernel reads this blob at boot instead of probing for hardware the way a PC does.

- **Base DTB** (`.dtb`, "device tree blob"): the *whole board's* hardware description, one big
  binary file. On this board, e.g. `tegra234-p3768-0000+p3767-0005-nv-super.dtb`.
- **Overlay** (`.dtbo`, "device tree blob overlay"): a small *patch* applied on top of the base
  DTB at boot, adding or modifying a few nodes without needing to recompile the entire base tree.
  Overlays are how pluggable/optional hardware (like a camera module that may or may not be
  attached) gets described — you keep one base DTB and swap which overlay(s) you apply depending
  on what's physically connected.

Both are compiled from human-readable **source** files (`.dts` / `.dtsi`, "device tree source")
using a compiler called `dtc`.

### What is `dtc`?

`dtc` (Device Tree Compiler) converts:
- `.dts` (text, human-readable) → `.dtb`/`.dtbo` (binary, what the bootloader/kernel actually reads)
- and — usefully for us — it can go the **other direction** too: `.dtb`/`.dtbo` → `.dts`. This
  "decompile" direction let us reverse-engineer the structure of overlays we didn't have the
  original source for.

### Nodes, fragments, and targets

A device tree is a tree of **nodes** (like `bus@0 { host1x@13e00000 { ... } }`), each with
**properties** (`key = value;` pairs — strings, numbers, or references to other nodes).

An overlay file is organized into one or more **fragments**, each saying "apply this chunk of
tree at this location in the base tree." Every overlay we looked at on this board uses a single
fragment with:
```
fragment@0 {
    target-path = "/";
    __overlay__ {
        ... nodes to add/merge in, starting from the tree root ...
    };
};
```
`target-path = "/"` means "merge everything below into the root of the base tree." Since the
overlay declares node names that already exist in the base tree at the same path (e.g. `bus@0`,
which is a real SoC-level bus node), the overlay's content gets **merged into** that existing
node — adding new children like `cam_i2cmux` — rather than replacing it.

### Labels, phandles, and why they matter

Some properties don't hold plain values — they need to **reference another node**. Example: "this
camera sensor's reset line is controlled by GPIO pin 62 on *this specific* GPIO controller." In
device tree source, you write that reference using a **label** with an `&` sigil:
```
reset-gpios = <&gpio 0x3e 0x00>;
```
`&gpio` means "the node elsewhere in the tree that has the label `gpio:` attached to it." When
`dtc` compiles this, it turns the label reference into a small integer called a **phandle**
(basically a pointer, but portable across the binary format). If you instead look at *already
compiled* output (like when we decompiled the existing overlays), you see the raw resolved
integers instead of names — e.g. `reset-gpios = <0xffffffff 0x3e 0x00>;` — which is much harder
to read or safely hand-edit, because the number `0xffffffff` by itself tells you nothing about
what it's pointing at.

This matters for overlays specifically because an overlay doesn't contain the *whole* tree — it
only contains its own small fragment, plus a promise to patch in references to nodes that live in
the base tree (which the overlay doesn't include). Three bookkeeping tables make that possible,
and `dtc` generates all three **automatically** — you never hand-write them:

- **`__symbols__`**: "the node I created at path X can be referred to by the label name Y" —
  for other overlays/parts of the *same* overlay to reference by name.
- **`__fixups__`**: "this property, in this exact node, needs to be patched at boot with the
  real address of a node named Z that lives in the **base tree**" (e.g. `gpio`, `gpio_aon`,
  `cam_i2c` — standard label names that already exist in this board's base DTB, for the main
  GPIO controller and the camera I2C bus).
- **`__local_fixups__`**: same idea, but the target is **inside this same overlay** (e.g. one
  sensor's output endpoint pointing at the CSI receiver's input endpoint, both defined in this
  same file).

You get these tables for free, correctly, **only if you compile from source with labels** (the
`-@` flag, below). If you hand-edit already-decompiled output (raw phandle numbers), you'd have to
manually keep all three tables consistent yourself — extremely error-prone. This is why we wrote
a fresh, labeled `.dts` source rather than patching the decompiled text directly.

### Camera-specific vocabulary used in these files

- **`nvcsi` / CSI channel**: the Tegra chip's MIPI CSI-2 receiver hardware. Each physical camera
  connector's data lanes land on a **channel** (`channel@0`, `channel@1`, ...). Two independent
  cameras need two different channels — reusing the same channel number for both is a real
  conflict (this is exactly the mistake we avoided — see §3).
- **`tegra-capture-vi`**: the "Video Input" block that reads frames off a CSI channel into memory.
  Also has a **port** per active camera, matched up to the corresponding CSI channel.
  matched up to the corresponding CSI channel.
- **`cam_i2cmux`**: a GPIO-controlled I2C multiplexer. Both CAM0 and CAM1's control (I2C)
  connections share one physical I2C bus at the SoC; a GPIO-driven mux switches which physical
  connector's I2C is "live" — modeled as two virtual I2C buses, `i2c@0` (CAM0) and `i2c@1` (CAM1).
- **`reset-gpios` / GPIO hogs**: the sensor's hard-reset / power-down pin. A "GPIO hog" is a node
  that just claims and drives a GPIO pin a fixed way at boot (no driver needed) — used here for
  the shared power-down lines (`cam0-pwdn`, `cam1-pwdn`) and, for IMX296 specifically, an
  additional dedicated reset line (`cam0-rst`) that IMX477 doesn't use.
- **`mode0`/`mode1`/... tables**: each sensor node lists one or more supported resolution/frame-
  rate "modes" (resolution, lane count, pixel clock, exposure/gain ranges, etc.) that the Argus
  camera stack picks from at runtime (this is exactly the list `gst-launch`'s
  `GST_ARGUS: Available Sensor modes` printout comes from).
- **`clocks` / fixed-clock**: IMX296 needs an explicit clock source description (`imx_296_fixed_cam_clk`,
  a simple fixed-frequency 54MHz clock node) that its driver references by name; IMX477's driver
  doesn't need this pattern.

---

## 2. The actual toolchain commands, explained

### Decompiling an existing overlay (reverse-engineering, read-only)

```bash
dtc -I dtb -O dts /boot/arducam/dts/tegra234-p3767-camera-p3768-imx477-dual.dtbo
```
- `-I dtb`: input format is a compiled binary blob.
- `-O dts`: output format is human-readable source text.
- No `-o` given → prints to stdout (we redirected to a file: `> dual.dts`).

We did this for three files to understand their structure before writing anything new:
1. `tegra234-p3767-camera-p3768-imx477-dual.dtbo` — the original, proven-working dual-camera
   overlay (CAM0=IMX477 + CAM1=IMX477).
2. `tegra234-p3767-camera-p3768-imx296-cam0.dtbo` — the already-confirmed-working single-camera
   IMX296 overlay.
3. `tegra234-p3767-camera-p3768-imx477-C.dtbo` — the standalone single-camera IMX477 (CAM1-only)
   overlay (read for comparison; ultimately **not used** in the final design — see §3).

Expect a wall of `Warning (unit_address_vs_reg)` / `Warning (i2c_bus_reg)` etc. — these are
harmless style complaints `dtc` makes about how the *original* NVIDIA/vendor tooling wrote these
files (e.g. disabled placeholder nodes with no `reg` property). They appeared identically when
decompiling the known-good, already-working files, so they're not a signal of a real problem.

### Compiling our new combined source

```bash
dtc -@ -I dts -O dtb -o combined.dtbo combined.dts
```
- `-@`: **generate the `__symbols__` table** for every labeled node in this source file. Without
  this flag, labels you define (`csi_in0_ep:`, `sensor_out0_ep:`, etc.) would compile fine for
  *internal* references, but the overlay-apply mechanism used at boot (and any future overlay
  wanting to reference nodes in *this* one) wouldn't be able to see them. This is the flag that
  makes an overlay's labels usable/cross-referenceable at all.
- `-I dts`: input is now text source (the file we hand-wrote).
- `-O dtb`: output is a compiled binary blob (despite the `.dtbo` name, it's the same binary
  format as `.dtb` — the "o" suffix is just a naming convention for "this one's an overlay").
- `-o combined.dtbo`: explicit output filename.

Exit code `0` and no non-cosmetic warnings means it compiled successfully — but that only proves
the **syntax and internal references** are valid, not that it will actually work on real hardware.
That's why we still decompiled the *result* and diffed its generated tables against the two
known-good originals (below) before ever touching the live board.

### Verifying the compiled result before installing it

```bash
dtc -I dtb -O dts combined.dtbo | sed -n '/__symbols__/,/^};/p'
dtc -I dtb -O dts combined.dtbo | sed -n '/__fixups__/,/^\t};/p'
```
Decompile our own freshly-built file right back, and read off the auto-generated `__symbols__`
and `__fixups__` tables. We confirmed:
- `__fixups__` referenced exactly the same three base-tree label names (`gpio_aon`, `cam_i2c`,
  `gpio`) that **both** of the original working overlays already depended on — strong evidence
  these fixups will resolve correctly at boot, since the base tree obviously already provides
  those labels (proven by the fact the originals already work).
- `__local_fixups__` showed every endpoint cross-reference (VI↔CSI↔sensor, for both cameras)
  resolved with no leftover unresolved/placeholder entries.

### Installing and wiring it into the boot config

```bash
sudo install -m 0644 combined.dtbo /boot/tegra234-p3767-camera-p3768-imx296-cam0-imx477-cam1-custom.dtbo
```
`install` here is just a slightly safer `cp` that also sets permissions explicitly (`0644` =
readable by everyone, writable only by root) — `/boot` is root-owned, hence `sudo`.

Then we edited `/boot/extlinux/extlinux.conf` (this board's boot menu config, read by the
bootloader before Linux even starts) — specifically the `OVERLAYS` and `FDT` lines of the active
boot label:
```
FDT /boot/arducam/dts/dtb/tegra234-p3768-0000+p3767-0005-nv-super.dtb
OVERLAYS /boot/tegra234-p3767-camera-p3768-imx296-cam0-imx477-cam1-custom.dtbo
```
- `FDT`: which **base** DTB to boot with.
- `OVERLAYS`: comma-separated list of overlay files to apply on top of that base DTB, in order,
  at boot.

**Important gotcha** (learned from the IMX296 driver package's own install notes, and worth
repeating): a boot label needs an explicit `FDT` line for `OVERLAYS` to actually take effect. A
label with `OVERLAYS` but no `FDT` line silently falls back to a default base tree and the
overlay is never applied — no error, no warning, the camera(s) just don't show up. Always check
both lines are present together.

---

## 3. Why we didn't just concatenate the two existing single-camera files

The obvious-looking shortcut — `OVERLAYS file1.dtbo,file2.dtbo` — would have been much less work,
so it's worth explaining precisely why it wouldn't have worked here.

Decompiling both standalone single-camera overlays showed:
- `imx477-C.dtbo` (CAM1-only): sensor sits on `nvcsi@15a00000/channel@0`
- `imx296-cam0.dtbo` (CAM0-only): sensor **also** sits on `nvcsi@15a00000/channel@0`

Both files were generated as standalone, "only one camera exists" configurations — and in that
context, always numbering the sole camera's CSI channel as `0` is a reasonable simplification.
But it means the two files are **not mutually compatible** as-is: loading both together would
have two sensors both trying to claim the same physical CSI receiver channel, and at best one
camera works while the other silently doesn't; at worst the overlay merge itself produces
inconsistent state.

The original **dual** overlay (`imx477-dual.dtbo`) doesn't have this problem, because it was
authored as a genuine two-camera file from the start: CAM0 on `channel@0`, CAM1 on `channel@1` —
a real hardware-accurate split. So instead of combining the two single-camera files, we used the
dual file as the structural template and only swapped the **sensor-specific** content on the
CAM0 side (compatible string, mode table, clock, extra reset GPIO) from IMX477 to IMX296, leaving
the CAM1 side and the channel-0/channel-1 split completely untouched.

---

## 4. Quick command reference (copy-paste log of this session)

```bash
# 1. Decompile existing overlays to understand their structure
dtc -I dtb -O dts /boot/arducam/dts/tegra234-p3767-camera-p3768-imx477-dual.dtbo > dual.dts
dtc -I dtb -O dts /boot/tegra234-p3767-camera-p3768-imx296-cam0.dtbo > imx296cam0.dts
dtc -I dtb -O dts /boot/arducam/dts/tegra234-p3767-camera-p3768-imx477-C.dtbo > imx477C.dts

# 2. (hand-author combined.dts, using dual.dts as the template + imx296cam0.dts's sensor block)

# 3. Compile the new source, generating the symbol table (-@)
dtc -@ -I dts -O dtb -o combined.dtbo combined.dts

# 4. Sanity-check the compiled result before installing
dtc -I dtb -O dts combined.dtbo | sed -n '/__symbols__/,/^};/p'
dtc -I dtb -O dts combined.dtbo | sed -n '/__fixups__/,/^\t};/p'

# 5. Install to /boot
sudo install -m 0644 combined.dtbo /boot/tegra234-p3767-camera-p3768-imx296-cam0-imx477-cam1-custom.dtbo

# 6. Point extlinux.conf's active label at it (FDT + OVERLAYS lines), then:
sudo reboot
```

## Related docs
- `imx296-cam0-setup.md` — narrative log of the IMX296/CAM0 bring-up and this combined-overlay attempt, including backups and revert commands.
- `camera-cam0-cam1-debug.md` — the original CAM0/CAM1 physical-port debugging history this whole effort builds on.
